---
title: spring boot入门(三)-Validator参数校验
date: 2021-01-04
excerpt_separator: "<!--more-->"
tags:  
  - spring boot
  - Validator
  - Valid
  - Validated
categories: 
  - Web开发  
author: linyuliang  
description: spring boot入门(三)-Validator参数校验 统一参数校验
toc: true
toc_sticky: true
---
本文是spring boot的新手入门教程(三)，目标是学会使用统一参数校验注解@Valid和@Validated，完成接口和方法参数的校验。  

参考资料：
- [valid 和 validated的使用小结](https://blog.csdn.net/liyanqiang19/article/details/107318650/)
- [Spring Validation最佳实践及其实现原理，参数校验没那么简单！](https://blog.csdn.net/xiaoxiaole0313/article/details/107903144)

<!-- more -->
## 概述  
　　基础概念和校验注解，网上有很多的资料,例如上面的参考资料，本文章不再复制描述，只关注一些重点。  
- @Validated和@Valid两种数据校验注解，Spring都支持。  
- @Valid（标准JSR-303规范）是javax（即Oracle）提供的校验注解,所属包为：`javax.validation.Valid`，`Hibernate-validator`对其进行了实现，并增加了一些自定义的校验注解。同时，它并非一定要和Spring结合在一起使用，可以手动导入`Hibernate-validator`依赖，如果是Spring boot用，可以导入`spring-boot-starter-validation`，这个包会依赖引入`Hibernate-validator`。
- @Validated（Spring's JSR-303规范，是标准JSR-303的一个变种）是Spring Validation验证框架对参数的验证机制提供的注解。  
- 配合BindingResult都可以直接提供参数验证结果。  
- 可以认为@Validated是@Valid的二次封装，二者在基本验证功能上没有太多区别。但是在分组、注解地方、嵌套验证等功能上两个有所不同。  
  ![valid&validated区别](/images/20210104/valid&Validated.png)  
1. 分组  
   @Validated：提供了一个分组功能，可以在入参验证时，根据不同的分组采用不同的验证机制，这个网上也有资料，不详述。  
   @Valid：作为标准JSR-303规范，还没有吸收分组的功能。  
   所以如果需要进行分组校验，使用@Validated注解
2. 注解的地方  
@Validated：可以用在类型、方法和方法参数上。但是不能用在成员属性（字段）上  
@Valid：可以用在方法、构造函数、方法参数和成员属性（字段）上  
其中两者是否能用于成员属性（字段）上直接影响能否提供嵌套验证的功能。    
3. 嵌套验证  
   以下面这个实体类为例，它的`attributes`属性，即为嵌套属性，指向另一个包含校验注解的实体类，只能使用`@Valid`注解来触发嵌套属性的参数校验，非集合，如果是个Bean属性也是一样的算嵌套属性。  

    ```java
    import io.swagger.annotations.ApiModel;
    import io.swagger.annotations.ApiModelProperty;
    import lombok.Data;
    import lombok.EqualsAndHashCode;
    import org.hibernate.validator.constraints.Length;

    import javax.validation.Valid;
    import javax.validation.constraints.NotBlank;
    import java.util.List;

    /**
    * <p>
    * 皮肤模板
    * </p>
    *
    * @author linyuliang
    * @since 2020-12-25
    */
    @Data
    @EqualsAndHashCode(callSuper = true)
    @ApiModel(value = "SkinTemplate创建请求对象", description = "皮肤模板")
    public class SkinTemplateCreateReq {
        @ApiModelProperty(value = "模板名称")
        @NotBlank(message = "模板名称不能为空")
        @Length(max = 64, message = "模板名称最多64个字")
        private String name;

        @Valid // 嵌套验证必须用@Valid
        @ApiModelProperty(value = "模板属性定义")
        private List<TemplateCreateAttribute> attributes;

        @Data
        public static class TemplateCreateAttribute {
            @ApiModelProperty(value = "属性名称")
            @NotBlank(message = "属性名称不能为空")
            @Length(max = 64, message = "属性名称最多64个字")
            private String name;

            @ApiModelProperty(value = "属性编码")
            @NotBlank(message = "属性编码不能为空")
            @Length(max = 64, message = "属性编码最多64个字")
            private String code;

            @ApiModelProperty(value = "属性类型")
            @NotBlank(message = "属性类型不能为空")
            @Length(max = 64, message = "属性类型最多64个字")
            private String type;

            @ApiModelProperty(value = "属性子类型")
            private String subType;

            @ApiModelProperty(value = "属性值，对于模板即默认值")
            @Length(max = 255, message = "属性值最多255个字")
            private String value;
        }

    }
    ```  
## 最简单的参数校验（建议方案）  
1. 对象参数校验使用@Valid注解
2. 属性参数校验使用在类上加@Validated注解，然后直接在属性参数上加校验注解，如下面的例子
3. 嵌套属性注解使用@Valid注解
4. 如果是分组校验，使用@Validated注解  

    ```java
    @Validated
    @RestController
    @RequestMapping("/v1/skin-theme-attr")
    public class SkinThemeAttrController extends AbstractController {
        @Autowired
        private ISkinThemeAttrService iSkinThemeAttrService;

        @Autowired
        ISkinThemeService skinThemeService;

        @ApiOperation(value = "创建皮肤主题属性")
        @RequestMapping(method = RequestMethod.POST)
        public SkinThemeAttrDTO create(@Valid @RequestBody SkinThemeAttrCreateReq skinThemeAttrCreateReq) {
            return iSkinThemeAttrService.create(skinThemeAttrCreateReq);
        }


        @ApiOperation(value = "分页获取皮肤主题属性列表")
        @RequestMapping(value = "", method = RequestMethod.GET)
        public PageImpl<SkinThemeAttrDTO> getPageByKeyword(@RequestParam(name = "offset", required = true) int offset,
                                                        @RequestParam(name = "limit", required = true) int limit,
                                                        @RequestParam(name = "sort", required = false) String sort,
                                                        @RequestParam(name = "searchKey", required = false) String searchKey,
                                                        @RequestParam(value = "themeId", required = false) Long themeId,
                                                        @RequestParam(value = "type", required = false) @Length(max = 64) String type
        ) {
            Pageable pageable = PageUtils.getPageable(offset, limit, sort, Arrays.asList(""), false);
            return iSkinThemeAttrService.listByPage(pageable, themeId, type, searchKey);
        }
    }
    ```  
## 借用总结图
看见这篇文章[《valid 和 validated的使用小结》](https://blog.csdn.net/liyanqiang19/article/details/107318650/)  里面的总结图：  
  ![参数校验总结图](/images/20210104/validator.png)  

## 高级特性
同样直接参考[《valid 和 validated的使用小结》](https://blog.csdn.net/liyanqiang19/article/details/107318650/) 
- 嵌套校验 `@Valid`
- 自定义校验 自定义注解，并实现`ConstraintValidator`接口
- 分组校验 
  - `@Validated({IGroupA.class,IGroupB.class})` `@Null(groups=IGroupA.class)`
  - 分组校验顺序注解`@GroupSequence({IGroupA.class,IGroupB.class})`
- 手动校验 
  - 原生  
    ```java
    ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
    Validator validator = factory.getValidator();
    Set<ContraintViolation<Input>> violations = validator.validate(user);
    if(!violations.isEmpty()){}
    ```
  - Spring  
    ```java
    @Autowired
    javax.validation.Validator globalValidator;

    Set<ContraintViolation<User>> set = globalValidator.validate(user);
    ```
  - Fail Fast Bean Validation默认会校验完所有字段，然后才抛出异常。可通过一些简单的配置，开启Fali Fast模式，一旦校验失败就立即返回。  
