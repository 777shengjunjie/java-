---
title: 在springBoot使用swagger
categories:
- shengjunjie
tags:
-  java、springBoot 、swagger
---
为方便前后端分离，实现前后端对于api的需求，因此在本文提供了配置swagger的方法

<!--more-->

## 解决方案
### 前期准备
在pom.xml文件中依赖相关的配置、
```
 <!--Swagger-->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.7.0</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```
### 核心思路
在springmvc总配置相关swagger。
### 详细过程
创建自定义配置类，实现对WebMvcConfigurer的继承（在这里如果在控制层不需要考虑返回逻辑视图，可以选择继承WebMvcConfigurationSupport）
```
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    /**
     * 添加swagger
     * @param registry
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");

    }
}
```
或者
```
@Configuration
public class WebMvcAutoConfiguration extends WebMvcConfigurationSupport {

    /**
     * 添加swagger
     * @param registry
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");

    }


}
```
完成自定义的mvc的配置后需要对swagger进行配置设置。
```
@Configuration
@EnableSwagger2
@ComponentScan(basePackages = {"com.controller"})
public class SwaggerConfig {


    @Bean
    public Docket customDocket() {
        return new
                Docket(DocumentationType.SWAGGER_2).apiInfo(apiInfo()
        );
    }

    private ApiInfo apiInfo() {

        Contact contact = new Contact("SHENGJJ",
                "", "");
        return new ApiInfoBuilder()
                .title("DST项目API")
                .description("DST登录接口")
                .contact(contact)
                .version("1.1.0")
                .build();

    }

}
```

## 总结
完成上述配置后，需要在控制层，添加相关注解例如@Api,@ApiOperation(value = "用户登录")，即可实现对swagger的使用（访问http://localhost:8080/swagger-ui.html即可查看相关已经配置的接口）
