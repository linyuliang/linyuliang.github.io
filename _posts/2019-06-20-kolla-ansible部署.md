---
layout: post  
title: kolla-ansible部署  
tags:  
- kolla-ansible  
- openstack  
- rocky
- 部署  
categories: OpenStack  
author: linyuliang  
description: kolla-ansible部署入门教程  
---
# kolla-ansible部署 
从未使用过openstack，本文是对openstack的第一次安装尝试，没时间整理，安装完再慢慢熟悉，后续整理。  
按照openstack官方当前最新release是stein版本的，但是pip install kolla-ansible安装的kolla-ansible，通过pip show kolla-ansible 查看是7.1.1版本，对应的是rocky版本的。所以本篇的kolla-ansible根据 [kolla-ansible用户指南](https://docs.openstack.org/project-deploy-guide/kolla-ansible/rocky/)，在Centos7.6服务器集群上尝试安装rocky版本的Openstack，后端采用ceph存储。  
参考资料：
- [Openstack高可用离线部署（使用Kolla部署，后端存储使用CEPH）](https://blog.csdn.net/liuyanwuyu/article/details/80821677)
- [kolla-ansible 部署openstack(vmware，多节点，ceph存储)](https://www.jianshu.com/p/f02f358a79f4)

<!-- more -->
#kolla-ansible部署
##主机要求
1. 服务器配置
    - 至少2台controller，1台computer，本文采用3台controller，2台computer
2. 主机必须满足以下最低要求： 
    - 系统Centos7.6
    - 2个网卡
    - 8GB主内存
    - 40GB磁盘空间
3.  系统其他要求
    - 安装系统的时候采用默认分区，并删除/home分区，剩余容量全部分给root分区
    - 系统其他硬盘，不要挂载，直接格式化
    - 服务器的网口至少两个，一个内网管理必须分配ip，连接网线；另一个不需要分配ip，连接网线。

##所有主机初始化
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
4. 配置kolla安装免密码访问其他节点
    ```
    ssh-keygen
    ssh-copy-id root@monitor
    ssh-copy-id root@controller01
    ssh-copy-id root@controller02
    ssh-copy-id root@controller03
    ssh-copy-id root@computer01
    ssh-copy-id root@computer02
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
    # 配置阿里镜像加速器，这里为chenjinhua@ruijie.com.cn的镜像加速器
    sudo mkdir -p /etc/docker
    sudo tee /etc/docker/daemon.json <<-'EOF'
    {
      "registry-mirrors": ["https://fzcndk1t.mirror.aliyuncs.com"]
    }
    EOF
    systemctl daemon-reload && systemctl enable docker && systemctl restart docker && systemctl status docker
    ```

7. 关闭防火墙
    ```
    systemctl stop firewalld && systemctl disable firewalld && systemctl status firewalld
    ```
8. disable 掉selinux
    ```
    setenforce 0
    sed -i '/^SELINUX=.*/c SELINUX=disabled' /etc/selinux/config
    cat /etc/selinux/config
    ```
9. 网卡配置（TODO 未完成双网卡绑定，一个管理网络，一个外网，等后续双网卡再说）
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
    ssh root@computer01 "sh /opt/linyuliang/create-bond-interfaces.sh ens1f0 ens1f1 bondInner 172.29.55.234 24 172.29.55.1 114.114.114.114"
    ssh root@computer01 "sh /opt/linyuliang/create-bond-interfaces.sh enp5s0f0 enp5s0f1 bondOuter - 24 - -"
    ssh root@computer01 "service network restart"

    ssh root@computer02 "mkdir -p /opt/linyuliang"
    scp /opt/linyuliang/create-bond-interfaces.sh root@computer02:/opt/linyuliang/
    ssh root@computer02 "chmod 777 /opt/linyuliang/create-bond-interfaces.sh"
    ssh root@computer02 "yum install -y lshw pciutils net-tools;modprobe --first-time bonding"
    ssh root@computer02 "sh /opt/linyuliang/create-bond-interfaces.sh ens1f0 ens1f1 bondInner 172.29.55.105 24 172.29.55.1 114.114.114.114"
    ssh root@computer02 "sh /opt/linyuliang/create-bond-interfaces.sh enp5s0f0 enp5s0f1 bondOuter - 24 - -"
    ssh root@computer02 "service network restart"
    ```
10. 重启服务器
    ```
    reboot
    ```
11. 所有节点都要安装
    ```
    sudo yum install -y wget
    sudo yum install -y epel-release
    sudo yum install -y python-devel libffi-devel gcc openssl-devel libselinux-python
    sudo yum install -y python-pip
    sudo pip install -U pip
    sudo pip install docker
    ```
12. 在compute节点上执行,为计算主机的空白硬盘打ceph标签
    ```
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    
    #SSD硬盘标记为CACHE
    parted /dev/sdb -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO1 1 -1
    parted /dev/sdc -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO2 1 -1
    parted /dev/sdd -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO3 1 -1
    parted /dev/sde -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO4 1 -1
    parted /dev/sdf -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO5 1 -1
    parted /dev/sdg -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO6 1 -1
    parted /dev/sdh -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS_FOO7 1 -1
    
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
    osd pool default min size = 3
    EOF
    ```
###kolla-ansible安装主机上执行
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
    kolla_install_type: "source"
    openstack_release: "rocky"
    
    docker_registry: ""
    docker_namespace: "kolla"
    
    kolla_internal_vip_address: "172.29.55.230"
    
    network_interface: "bondInner"
    neutron_external_interface: "bondOuter"
    
    neutron_plugin_agent: "openvswitch"
    enable_haproxy: "yes"
    enable_cinder: "yes"
    enable_horizon: "yes"
    
    # 以下为ceph配置
    enable_ceph: "yes"
    enable_ceph_rgw: "yes"
    enable_ceph_rgw_keystone: "yes"
    #ceph_target_max_bytes: ""  # 表示每个cache pool最大大小。注意，如果配置了cache盘，此项不配置会导致cache不会自动清空。cache_osd_size*cache_osd_num/replicated/cache_pool_num
    #ceph_target_max_objects: "" # 表示cache pool最大的object数量
    ceph_pool_pg_num: 128 # Total PGs = ((Total_number_of_OSD * 100) / pool_count / replicated . 
    ceph_pool_pgp_num: 128
    glance_backend_ceph: "yes"
    glance_backend_file: "no"
    cinder_backend_ceph: "yes"
    cinder_backup_driver: "ceph"
    nova_backend_ceph: "yes"

    enable_neutron_dvr: "yes"
    enable_neutron_agent_ha: "yes"
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
    #查看环境变量
    cat /etc/kolla/admin-openrc.sh
    #安装便捷客户端
    pip install -U python-openstackclient python-neutronclient
    #使admin环境生效
    source /etc/kolla/admin-openrc.sh
    #查看计算服务
    openstack compute service list
    ```
8. 在contoller节点，查看ceph状态  
    `docker exec ceph_mon ceph -s`
9. 通过vip，访问openstack  
    `http://172.29.55.230`
10. 销毁openstack  
    ``` 
    kolla-ansible destroy -i ./multinode  --yes-i-really-really-mean-it
    ```
11. 创建实例时，便捷ssh登录脚本
    ```
    #!/bin/sh
    passwd root<<EOF
    123456
    123456
    EOF
    sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
    systemctl restart sshd
    ```