---
title: kolla-ansible部署高可用openstack  
date: 2019-06-20
excerpt_separator: "<!--more-->"
tags:  
    - kolla-ansible  
    - openstack    
categories: 
    - Iaas  
author: linyuliang  
keywords: [kolla-ansible,openstack,部署,rocky]
description: kolla-ansible部署openstack入门教程，使用multinode  
toc: true
toc_sticky: true
---
从未使用过openstack，本文是对高可用openstack的第一次安装尝试，没时间整理，安装完再慢慢熟悉，后续整理。  
按照openstack官方当前最新release是stein版本的，但是pip install kolla-ansible安装的kolla-ansible，通过pip show kolla-ansible 查看是7.1.1版本，对应的是rocky版本的。所以本篇的kolla-ansible根据 [kolla-ansible用户指南](https://docs.openstack.org/project-deploy-guide/kolla-ansible/rocky/)，在Centos7.6服务器集群上尝试安装rocky版本的Openstack，后端采用ceph存储。  
kolla-ansible和openstack的版本需要对应起来，这很重要！  
参考资料：
- [Openstack高可用离线部署（使用Kolla部署，后端存储使用CEPH）](https://blog.csdn.net/liuyanwuyu/article/details/80821677)
- [kolla-ansible 部署openstack(vmware，多节点，ceph存储)](https://www.jianshu.com/p/f02f358a79f4)

<!-- more -->
## 本文主机配置
1. 主机必须满足以下最低要求： 
    - 2个网卡，都需要网线连接（内部管理网络建议万兆，外部网络千兆即可）
    - 8GB主内存
    - 40GB磁盘空间
2. 本文服务器配置
    - 至少2台controller，1台computer，本文采用1台monitor，3台controller，2台computer  

   | hostname | 网卡bondInner | 网卡bondOuter | 
   |  -------- | -------- | -------- |
   | monitor | 172.29.55.229 | 无ip   |
   | controller01 | 172.29.55.231 | 无ip   |
   | controller02 | 172.29.55.232 | 无ip   |
   | controller03 | 172.29.55.233 | 无ip  | 
   | compute01 | 172.29.55.234 | 无ip   |
   |  compute02 | 172.29.55.235 | 无ip   |
    
3.  系统其他要求
    - 安装系统的时候采用默认分区，并删除/home分区，剩余容量全部分给root分区
    - 系统其他硬盘，不要挂载，直接格式化
    - 服务器的网口至少两个，一个内网管理必须分配ip，连接网线；另一个不需要分配ip，连接网线。

## 所有主机初始化
1. yum加速源配置，并安装需要的依赖
    ```
    mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
    cd /etc/yum.repos.d/
    curl -o /etc/yum.repos.d/CentOS7-Base-163.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
    curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
    yum clean all
    yum makecache
    yum -y update
    
    yum install -y git net-tools ntp vim wget ansible gcc openssl-devel python-devel python-pip libffi-devel libselinux-python python-openstackclient python-neutronclient
    ```
2. ntp服务配置
    ```
    yum install ntp ntpdate -y
    cp /etc/ntp.conf /etc/ntp.conf.backup
    cat <<EOT > /etc/ntp.conf
    server ntp1.aliyun.com
    server ntp2.aliyun.com
    server ntp3.aliyun.com
    server ntp4.aliyun.com
    server ntp5.aliyun.com
    server ntp6.aliyun.com
    server ntp7.aliyun.com
    EOT
    systemctl enable ntpd.service && systemctl start ntpd.service && systemctl status ntpd.service
    ntpq -p
    ```
3. 修改服务器的hostname
    ```
    echo "monitor" > /etc/hostname
    hostname monitor
    hostnamectl set-hostname monitor
        
    echo "controller01" > /etc/hostname
    hostname controller01
    hostnamectl set-hostname controller01
    
    echo "controller02" > /etc/hostname
    hostname controller02
    hostnamectl set-hostname controller02
    
    echo "controller03" > /etc/hostname
    hostname controller03
    hostnamectl set-hostname controller03
    
    echo "computer01" > /etc/hostname
    hostname computer01
    hostnamectl set-hostname computer01

    echo "computer02" > /etc/hostname
    hostname computer02
    hostnamectl set-hostname computer02
    
    echo -e '172.29.55.229\tmonitor' >> /etc/hosts
    echo -e '172.29.55.231\tcontroller01' >> /etc/hosts
    echo -e '172.29.55.232\tcontroller02' >> /etc/hosts
    echo -e '172.29.55.233\tcontroller03' >> /etc/hosts
    echo -e '172.29.55.234\tcomputer01' >> /etc/hosts
    echo -e '172.29.55.235\tcomputer02' >> /etc/hosts
        
    scp /etc/hosts root@monitor:/etc/
    scp /etc/hosts root@controller01:/etc/
    scp /etc/hosts root@controller02:/etc/
    scp /etc/hosts root@controller03:/etc/
    scp /etc/hosts root@computer01:/etc/
    scp /etc/hosts root@computer02:/etc/
    ```
4. 配置monitor节点可以免密码访问其他节点
    ```
    ssh-keygen
    ssh-copy-id root@monitor
    ssh-copy-id root@controller01
    ssh-copy-id root@controller02
    ssh-copy-id root@controller03
    ssh-copy-id root@computer01
    ssh-copy-id root@computer02
    ```
    配置后，可以通过命令测试，是否可以免密访问
    ```
    ssh monitor
    ssh controller01
    ssh controller02
    ssh controller03
    ssh computer01
    ssh computer02
    ```
5. pip加速源配置
    ```
    mkdir ~/.pip
    cat > ~/.pip/pip.conf << EOF 
    [global]
    trusted-host=mirrors.aliyun.com
    index-url=https://mirrors.aliyun.com/pypi/simple/
    EOF
    ```
6. docker加速源配置以及安装
    ```
    sudo yum install -y yum-utils device-mapper-persistent-data lvm2
    sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    sudo yum makecache fast
    sudo yum -y install docker-ce
    # 如果不设置此项，kolla-ansible 部署neutron-dhcp-agent 容器的时候会失败，并抛出 APIError/HTTPError
    mkdir /etc/systemd/system/docker.service.d
    tee /etc/systemd/system/docker.service.d/kolla.conf << 'EOF'
    [Service]
    MountFlags=shared
    EOF
    # 配置阿里镜像加速器，xxxxxx不同阿里云用户不同
    sudo mkdir -p /etc/docker
    sudo tee /etc/docker/daemon.json <<-'EOF'
    {
      "registry-mirrors": ["https://xxxxxx.mirror.aliyuncs.com"],
    }
    EOF
    systemctl daemon-reload && systemctl enable docker && systemctl restart docker && systemctl status docker
    ```
6. 制作本地镜像源（部署节点不能为openstack的安装节点，否则预安装的时候，docker会重启失败，因为私有仓库包含在里面）  
    下面1-6步骤在monitor部署节点上执行，7在所有节点上执行
    1. 修改kolla镜像有关配置
        ```
        vim /etc/kolla/globals.yml
        openstack_release: "rocky"
        network_interface: "bondInner"
        ```
    2. 拉取镜像
        `kolla-ansible pull -vvv`
    3. 启动本地私有仓库的容器
        ```
        mkdir -p /opt/docker/registry
        docker run -d -p 4000:5000 -v /opt/docker/registry:/var/lib/registry --restart=always --name registry registry:latest
        ```
    5. 修改上面拉取到的kolla镜像tag,将kolla镜像上传到上面的私有仓库
        ```
        for i in `docker images|grep kolla|awk '{print $1}'`;do docker tag $i:rocky 172.29.55.229:4000/$i:rocky;done  
        # 对应移除tag命令：docker rmi -f $(docker images|grep 172.29.55.229|awk '{print $1":rocky"}')  
        for i in `docker images|grep 172.29.55.229|awk '{print $1}'`;do docker push $i:rocky;done  
        ```
    6. 查看镜像是否成功上传私有仓库
        ```
        curl -XGET http://172.29.55.229:4000/v2/_catalog  
        ```  
    7. 所有节点上，增加私有仓库加速配置，重启docker
        ```
        sudo tee /etc/docker/daemon.json <<-'EOF'
        {
          "registry-mirrors": ["https://fzcndk1t.mirror.aliyuncs.com"],
          "insecure-registries":["172.29.55.229:4000"]
        }
        EOF
        systemctl daemon-reload && systemctl restart docker 
        ```
7. 关闭防火墙
    ```
    systemctl stop firewalld && systemctl disable firewalld && systemctl status firewalld
    ```
8. 网卡配置
    create-bond-interfaces.sh:
    ```
    #!/usr/bin/env bash
    set -e
    
    EXITCODE=0
    
    function create_bond_on_centos () {
       # yum install -y lshw pciutils net-tools
       echo "Creating bond interfaces on centos"
       sudo lsmod | grep bonding >& /dev/null
       if lsmod | grep bond; then
          echo "Bonding module already loaded"
       else
          echo "Bonding module is not loaded"
          sudo modprobe --first-time bonding
       fi
       echo "bonding" > /etc/modules-load.d/bonding.conf
    
       if [[ "$1"x != "-"x ]]; then
       echo
       echo "Creating interface $1 file"
       echo
       cat << EOF > /etc/sysconfig/network-scripts/ifcfg-$1
    TYPE=Ethernet
    BOOTPROTO=none
    NAME=$1
    DEVICE=$1
    ONBOOT=yes
    MASTER=$3
    SLAVE=yes
    EOF
       fi
    
       if [[ "$2"x != "-"x ]]; then
       echo "Creating interface $2 file"
       echo
       cat << EOF > /etc/sysconfig/network-scripts/ifcfg-$2
    TYPE=Ethernet
    BOOTPROTO=none
    NAME=$2
    DEVICE=$2
    ONBOOT=yes
    MASTER=$3
    SLAVE=yes
    EOF
       fi
    
       echo "Creating bond interface $3 file"
      if [[ "$bondIp"x != "-"x ]]; then
       echo
       cat << EOF > /etc/sysconfig/network-scripts/ifcfg-$3
    DEVICE=$3
    NAME=$3
    TYPE=Bond
    ONBOOT=yes
    BOOTPROTO=static
    IPADDR=$bondIp
    PREFIX=$bondSubnet
    GATEWAY=$gateway
    DNS1=$dns1
    BONDING_MASTER=yes
    BONDING_OPTS="mode=4 miimon=100"
    EOF
        else
      echo
       cat << EOF > /etc/sysconfig/network-scripts/ifcfg-$3
    DEVICE=$3
    NAME=$3
    TYPE=Bond
    ONBOOT=yes
    BOOTPROTO=none
    BONDING_MASTER=yes
    BONDING_OPTS="mode=4 miimon=100"
    EOF
       fi
       echo "Creation of the interface configuration files is done!!"
       echo
       echo "Bring up interfaces now!"
       sudo ifdown $3
       sudo ifup $3
        if [[ "$1"x != "-"x ]]; then
            sudo ifdown $1
            sudo ifup $1
        fi
        if [[ "$2"x != "-"x ]]; then
            sudo ifdown $2
            sudo ifup $2
        fi
    
       echo -e "Output of ip a \n"
       ip a
       echo -e "Done !!!! \n\n"
    }
    
    echo "User input for the interfaces which are part of bond interface"
    echo "Bond interface name and IP Address"
    intf1=$1
    intf2=$2
    bondIntfName=$3
    bondIp=$4
    bondSubnet=$5
    gateway=$6
    dns1=$7
    
    echo create_bond_on_centos "$intf1" "$intf2" "$bondIntfName" "$bondIp" "$bondSubnet" "$gateway" "$dns1"
    create_bond_on_centos "$intf1" "$intf2" "$bondIntfName" "$bondIp" "$bondSubnet" "$gateway" "$dns1"
    
    exit $EXITCODE
    ```
    执行网卡绑定：
    ```
    #monitor节点
    ssh root@monitor "mkdir -p /opt/linyuliang"
    scp /opt/linyuliang/create-bond-interfaces.sh root@monitor:/opt/linyuliang/
    ssh root@monitor "chmod 777 /opt/linyuliang/create-bond-interfaces.sh"
    ssh root@monitor "yum install -y lshw pciutils net-tools;modprobe --first-time bonding"
    ssh root@monitor "sh /opt/linyuliang/create-bond-interfaces.sh enp5s0 enp6s0 bondInner 172.29.55.229 24 172.29.55.1 114.114.114.114"
    ssh root@monitor "sh /opt/linyuliang/create-bond-interfaces.sh enp7s0 enp8s0 bondOuter - 24 - -"
    ssh root@monitor "service network restart"

    ssh root@controller01 "mkdir -p /opt/linyuliang"
    scp /opt/linyuliang/create-bond-interfaces.sh root@controller01:/opt/linyuliang/
    ssh root@controller01 "chmod 777 /opt/linyuliang/create-bond-interfaces.sh"
    ssh root@controller01 "yum install -y lshw pciutils net-tools;modprobe --first-time bonding"
    ssh root@controller01 "sh /opt/linyuliang/create-bond-interfaces.sh - eno1 bondInner 172.29.55.231 24 172.29.55.1 114.114.114.114"
    ssh root@controller01 "sh /opt/linyuliang/create-bond-interfaces.sh - enp4s0f1 bondOuter - 24 - -"
    ssh root@controller01 "service network restart"

    ssh root@controller02 "mkdir -p /opt/linyuliang"
    scp /opt/linyuliang/create-bond-interfaces.sh root@controller02:/opt/linyuliang/
    ssh root@controller02 "chmod 777 /opt/linyuliang/create-bond-interfaces.sh"
    ssh root@controller02 "yum install -y lshw pciutils net-tools;modprobe --first-time bonding"
    ssh root@controller02 "sh /opt/linyuliang/create-bond-interfaces.sh - eno1 bondInner 172.29.55.232 24 172.29.55.1 114.114.114.114"
    ssh root@controller02 "sh /opt/linyuliang/create-bond-interfaces.sh - enp4s0f1 bondOuter - 24 - -"
    ssh root@controller02 "service network restart"
    
    ssh root@controller03 "mkdir -p /opt/linyuliang"
    scp /opt/linyuliang/create-bond-interfaces.sh root@controller03:/opt/linyuliang/
    ssh root@controller03 "chmod 777 /opt/linyuliang/create-bond-interfaces.sh"
    ssh root@controller03 "yum install -y lshw pciutils net-tools;modprobe --first-time bonding"
    ssh root@controller03 "sh /opt/linyuliang/create-bond-interfaces.sh - eno1 bondInner 172.29.55.233 24 172.29.55.1 114.114.114.114"
    ssh root@controller03 "sh /opt/linyuliang/create-bond-interfaces.sh - enp4s0f1 bondOuter - 24 - -"
    ssh root@controller03 "service network restart"
    
    ssh root@computer01 "mkdir -p /opt/linyuliang"
    scp /opt/linyuliang/create-bond-interfaces.sh root@computer01:/opt/linyuliang/
    ssh root@computer01 "chmod 777 /opt/linyuliang/create-bond-interfaces.sh"
    ssh root@computer01 "yum install -y lshw pciutils net-tools;modprobe --first-time bonding"
    ssh root@computer01 "sh /opt/linyuliang/create-bond-interfaces.sh ens1f0 - bondInner 172.29.55.234 24 172.29.55.1 114.114.114.114"
    ssh root@computer01 "sh /opt/linyuliang/create-bond-interfaces.sh - ens1f1 bondOuter - 24 - -"
    ssh root@computer01 "service network restart"

    ssh root@computer02 "mkdir -p /opt/linyuliang"
    scp /opt/linyuliang/create-bond-interfaces.sh root@computer02:/opt/linyuliang/
    ssh root@computer02 "chmod 777 /opt/linyuliang/create-bond-interfaces.sh"
    ssh root@computer02 "yum install -y lshw pciutils net-tools;modprobe --first-time bonding"
    ssh root@computer02 "sh /opt/linyuliang/create-bond-interfaces.sh ens1f0 - bondInner 172.29.55.235 24 172.29.55.1 114.114.114.114"
    ssh root@computer02 "sh /opt/linyuliang/create-bond-interfaces.sh - ens1f1 bondOuter - 24 - -"
    ssh root@computer02 "service network restart"
    ```
10. 所有节点都要安装
    ```
    sudo yum install -y wget
    sudo yum install -y epel-release
    sudo yum install -y python-devel libffi-devel gcc openssl-devel libselinux-python
    sudo yum install -y python-pip
    sudo pip install -U pip
    sudo pip install docker
    ```
11. 所有节点重启服务器
    ```
    reboot
    ```
12. 为存储节点的空白硬盘打ceph标签（本文存储节点，在controller和computer节点上）
    ```
    #controller01
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    #controller02
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    #controller03
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    
    #SSD硬盘标记为CACHE，会导致ceph创建失败，目前仅一个SSD，无法确认是否CACHE个数和副本数需要对应
    #computer01
    #parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_CACHE_BOOTSTRAP_BS 1 -1
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    parted /dev/sdc -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO2 1 -1
    parted /dev/sdd -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO3 1 -1
    parted /dev/sde -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO4 1 -1
    parted /dev/sdf -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO5 1 -1
    parted /dev/sdg -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO6 1 -1
    parted /dev/sdh -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO7 1 -1
    
    #computer02
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    parted /dev/sdc -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO2 1 -1
    parted /dev/sdd -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO3 1 -1
    parted /dev/sde -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO4 1 -1
    parted /dev/sdf -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO5 1 -1
    ```
13. 在monitor和compute节点上执行，为ceph_rgw创建池
    ```
    mkdir -pv /etc/kolla/config/ && touch /etc/kolla/config/ceph.conf
    cat > /etc/kolla/config/ceph.conf << EOF 
    [global]
    osd pool default size = 3
    osd pool default min size = 2
    EOF
    ```
### kolla-ansible安装主机monitor上执行
1. 安装依赖
    ```
    sudo yum install -y ansible
    ```
2. 安装Kolla-ansible
    ```
    sudo pip install kolla-ansible==7.1.1 --ignore-installed PyYAML
    sudo mkdir -p /etc/kolla
    sudo chown $USER:$USER /etc/kolla
    cp -r /usr/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
    cp /usr/share/kolla-ansible/ansible/inventory/* .
    ```
3. ansible配置
    ```
    cp /etc/ansible/ansible.cfg /etc/ansible/ansible.cfg.backup
    cat <<EOT > /etc/ansible/ansible.cfg
    [defaults]
    host_key_checking=False
    pipelining=True
    forks=100
    timeout=800
    deprecation_warnings=False
    EOT
    ```
4. 生成密码文件，此处修改成密码admin
    ```
    kolla-genpwd
    sed -i "s/^keystone_admin_password:.*/keystone_admin_password: admin/" /etc/kolla/passwords.yml
    ```
5. 修改全局配置,vi /etc/kolla/globals.yml,openstack_release 填stein，mariadb镜像启动报错
    ```
    kolla_base_distro: "centos"
    kolla_install_type: "binary"
    openstack_release: "rocky"
    
    kolla_internal_vip_address: "172.29.55.230"
    
    docker_registry: "172.29.55.229:4000"
    docker_namespace: "kolla"

    network_interface: "bondInner"
    neutron_external_interface: "bondOuter"

    enable_ceph: "yes"
    enable_ceph_rgw: "yes"
    enable_chrony: "yes"
    enable_cinder: "yes"

    enable_haproxy: "yes"
    enable_horizon: "yes"

    enable_neutron_dvr: "yes"
    enable_neutron_agent_ha: "yes"
    
    ceph_enable_cache: "yes"

    #ceph_target_max_bytes: "106181135928"  # 表示每个cache pool最大大小。注意，如果配置了cache盘，此项不配置会导致cache不会自动清空。cache_osd_size*cache_osd_num/replicated/cache_pool_num
    #ceph_target_max_objects: "" # 表示cache pool最大的object数量
    
    #PG和PGP数量一定要根据OSD的数量进行调整，计算公式如下，但是最后算出的结果一定要接近或者等于一个2的指数。
    #Total PGs = (Total_number_of_OSD * 100) / max_replication_count
    #例如15个OSD，副本数为3的情况下，根据公式计算的结果应该为500，最接近512，所以需要设定该pool(volumes)的pg_num和pgp_num都为512.
    ceph_pool_pg_num: 128
    ceph_pool_pgp_num: 128
    
    glance_backend_ceph: "yes"
    glance_backend_file: "no"
    cinder_backend_ceph: "{{ enable_ceph }}"
    cinder_backup_driver: "ceph"
    nova_backend_ceph: "{{ enable_ceph }}"
    ```
6. ansible主机清单配置，vi ./multinode
    ```
    [control]
    controller01
    controller02
    controller03
    
    [network]
    controller01
    controller02
    controller03
    
    [inner-compute]
    
    [external-compute]
    computer01
    computer02
    
    [compute:children]
    inner-compute
    external-compute
    
    [storage]
    controller01
    controller02
    controller03
    computer01
    computer02
    
    [monitoring]
    monitor
    
    [deployment]
    monitor
    ```
7. 在monitor节点，开始部署
    ```
    #预安装
    kolla-ansible -i ./multinode bootstrap-servers
    #安装之前还需要预检查一下主机环境以及配置文件是否正确
    ansible -i multinode all -m ping
    kolla-ansible -i ./multinode prechecks
    #拉取镜像
    kolla-ansible -i ./multinode pull
    #开始部署
    kolla-ansible -i ./multinode deploy
    
    #部署完成之后，还需要这步操作，生成环境变量和脚本：
    kolla-ansible -i ./multinode post-deploy
    #安装便捷客户端
    pip install -U python-openstackclient python-neutronclient
    #使admin环境生效
    source /etc/kolla/admin-openrc.sh
    #查看计算服务
    openstack compute service list
    
    #运行脚本创建示例网络，图像等
    . /usr/share/kolla-ansible/init-runonce
    #部署demo实例
    openstack server create \
        --image cirros \
        --flavor m1.tiny \
        --key-name mykey \
        --network demo-net \
        demo1
    ```
8. 在contoller节点，查看ceph状态 ，kolla-ansible默认权重都相等，并且为1，要自己调整，一个个调整，不能一下子全部调整完，等调整后看状态ok，再继续下一个调整  
    docker exec ceph_mon ceph osd crush reweight osd.4 4表示osd,4这个OSD的weight权重调整为4（设置1T为1，4T为4）
    ```
    docker exec ceph_mon ceph -s
    docker exec ceph_mon ceph osd tree
    docker exec ceph_mon ceph osd crush reweight osd.4 4
    docker exec ceph_mon ceph health detail
    docker exec ceph_mon ceph osd crush reweight osd.5 4
    docker exec ceph_mon ceph osd crush reweight osd.6 4
    docker exec ceph_mon ceph osd crush reweight osd.7 4
    docker exec ceph_mon ceph osd crush reweight osd.8 4
    docker exec ceph_mon ceph osd crush reweight osd.9 4
    docker exec ceph_mon ceph osd crush reweight osd.10 4
    docker exec ceph_mon ceph osd crush reweight osd.11 4
    docker exec ceph_mon ceph osd crush reweight osd.12 4
    docker exec ceph_mon ceph osd crush reweight osd.13 4
    docker exec ceph_mon ceph osd crush reweight osd.14 4
    ```
9. 通过vip，访问openstack  
    `http://172.29.55.230`
10. 销毁openstack  
    ``` 
    kolla-ansible destroy -i ./multinode  --yes-i-really-really-mean-it
    ```
### 其他
1. 建立flat网络注意事项：
    1. 在controller主机上查看物理网络：cat /etc/kolla/neutron-server/ml2_conf.ini  
    ```
    [ml2_type_flat]
    flat_networks = physnet1
    ```
    2. 物理网络 填写为：physnet1
    3. 创建网络 不勾选：外部网络
 2. 要使用vip，提供一个暴力的方法(更好的方案见：allowed-address-pair 特性)：  
    ```
    id=`openstack port list |grep "{要关掉安全组的IP}"|awk '{print $2}'`
    openstack port set --no-security-group $id
    openstack port set --disable-port-security $id
    ```
3. 不想使用ceph，则如下方式进行：
    修改全局配置,vi /etc/kolla/globals.yml,openstack_release 填stein，mariadb镜像启动报错
    ```
    kolla_base_distro: "centos"
    kolla_install_type: "binary"
    openstack_release: "rocky"
    
    kolla_internal_vip_address: "172.29.55.230"
    
    docker_registry: "172.29.55.229:4000"
    docker_namespace: "kolla"

    network_interface: "bondInner"
    neutron_external_interface: "bondOuter"

    enable_cinder: "yes"
    enable_cinder_backend_iscsi: "yes"
    enable_cinder_backend_lvm: "yes"

    enable_haproxy: "yes"
    enable_horizon: "yes"

    enable_neutron_dvr: "yes"
    enable_neutron_agent_ha: "yes"

    ```
    磁盘格式化，挂载卷，后使用vgs查看卷
    ```
    vgremove cinder-volumes && pvremove /dev/sdb
    parted -s /dev/sdb mklabel gpt &&dd if=/dev/urandom of=/dev/sdb bs=512 count=64 && pvcreate /dev/sdb && vgcreate cinder-volumes /dev/sdb
    vgs

    vgremove cinder-volumes && pvremove /dev/sdb
    parted -s /dev/sdb mklabel gpt &&dd if=/dev/urandom of=/dev/sdb bs=512 count=64 && pvcreate /dev/sdb && vgcreate cinder-volumes /dev/sdb
    vgs

    vgremove cinder-volumes && pvremove /dev/sdb
    parted -s /dev/sdb mklabel gpt &&dd if=/dev/urandom of=/dev/sdb bs=512 count=64 && pvcreate /dev/sdb && vgcreate cinder-volumes /dev/sdb
    vgs

    vgremove cinder-volumes && pvremove /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh
    parted -s /dev/sdb mklabel gpt &&dd if=/dev/urandom of=/dev/sdb bs=512 count=64
    parted -s /dev/sdc mklabel gpt &&dd if=/dev/urandom of=/dev/sdc bs=512 count=64
    parted -s /dev/sdd mklabel gpt &&dd if=/dev/urandom of=/dev/sdd bs=512 count=64
    parted -s /dev/sde mklabel gpt &&dd if=/dev/urandom of=/dev/sde bs=512 count=64
    parted -s /dev/sdf mklabel gpt &&dd if=/dev/urandom of=/dev/sdf bs=512 count=64
    parted -s /dev/sdg mklabel gpt &&dd if=/dev/urandom of=/dev/sdg bs=512 count=64
    parted -s /dev/sdh mklabel gpt &&dd if=/dev/urandom of=/dev/sdh bs=512 count=64
    pvcreate /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh && vgcreate cinder-volumes /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf /dev/sdg /dev/sdh
    vgs 

    vgremove cinder-volumes && pvremove /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf
    parted -s /dev/sdb mklabel gpt &&dd if=/dev/urandom of=/dev/sdb bs=512 count=64 
    parted -s /dev/sdc mklabel gpt &&dd if=/dev/urandom of=/dev/sdc bs=512 count=64 
    parted -s /dev/sdd mklabel gpt &&dd if=/dev/urandom of=/dev/sdd bs=512 count=64 
    parted -s /dev/sde mklabel gpt &&dd if=/dev/urandom of=/dev/sde bs=512 count=64 
    parted -s /dev/sdf mklabel gpt &&dd if=/dev/urandom of=/dev/sdf bs=512 count=64 
    pvcreate /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf && vgcreate cinder-volumes /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf 
    vgs
    ```
