---
title: Shiro认证与权限管理
categories:
- shengjunjie
tags:
- java、springBoot 
---

考虑到项目需要对用户的权限需要管理，使各个公司之间的数据进行隔离，同时需要满足公司用户对其子用户其权限进行管理

<!--more-->

## 解决方案
### 前期准备
1. pom.xml文件的依赖（基于shiro文件依赖）
```
<!--shiro和spring整合-->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.3.2</version>
</dependency>
<!--shiro核心包-->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-core</artifactId>
    <version>1.3.2</version>
</dependency>
<!--shiro与redis整合-->
<dependency>
    <groupId>org.crazycake</groupId>
    <artifactId>shiro-redis</artifactId>
    <version>3.0.0</version>
</dependency>
```
2. 在.yml中进行redis的喷配置
```
spring:
  #数据库连接池
  datasource:
    url: jdbc:mysql://localhost:数据库端口/数据库名称?serverTimezone=GMT%2B8
    username: root（账号）
    password: root（密码）
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    database: Mysql
    show-sql: true
    open-in-view: true
  redis:
    host: 127.0.0.1
    port: 6381（redis的端口）
```
3. CustomSessionManager配置文件(用于认证，基于redis生成SessionId，通过redis可以解决session跨域请求的问题)

```
package com.dst_common.shiro.session;

import org.apache.shiro.web.servlet.ShiroHttpServletRequest;
import org.apache.shiro.web.session.mgt.DefaultWebSessionManager;
import org.apache.shiro.web.util.WebUtils;
import org.springframework.util.StringUtils;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import java.io.Serializable;

public class CustomSessionManager extends DefaultWebSessionManager {


    @Override
    protected Serializable getSessionId(ServletRequest request, ServletResponse response) {

        //1.获取请求头Authorization中的数据
        String id = WebUtils.toHttp(request).getHeader("Authorization");
        //如果没有生成新的sessionId
        if (StringUtils.isEmpty(id)) {
            return super.getSessionId(request, response);

        } else {
            //获取请求头中的信息：Bearer sessionId
            id = id.replaceAll("Bearer ", "");
            //返回sessionId
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_SOURCE, "header");
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID, id);
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_IS_VALID, Boolean.TRUE);

            return id;

        }


    }
}

```
4. 对Realm进行配置（用于控制登录认证与权限管理,在这里将权限管理与登录认证分离，通过继承的方法，用于满足微服务的认证模块与登录模块的需求）
* 登录认证：
```
package com.common.shiro.realm;

import com.domain.system.Permission;
import com.domain.system.User;
import com.domain.system.response.ProfileResult;
import com.dst_common.shiro.realm.CustomRealm;
import com.service.PermissionService;
import com.service.UserService;
import org.apache.shiro.authc.*;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.subject.PrincipalCollection;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class UserRealm extends CustomRealm {

    @Autowired
    private UserService userService;

    @Autowired
    private PermissionService permissionService;



    /**
     * 认证方法
     *
     * @param token
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

        //1.获取用户的工作工作号和密码
        UsernamePasswordToken upToken = (UsernamePasswordToken) token;
        String workNumber = upToken.getUsername();
        String password = String.valueOf(upToken.getPassword());
        //2.根据用户工作号查询用户
        User user = userService.findByWorkNumber(workNumber);
        //3.判断用户是否存在
        if (user != null && user.getPassword().equals(password)) {
            //4.构造安全数据并返回（安全数据：用户基本数据，权限信息 profileResult）
            ProfileResult result = null;

            /**
             * 这里通过 result = new ProfileResult(user);直接返回也可以
             */

            if ("user".equals(user.getLevel())) {
                result = new ProfileResult(user);
            } else {
                Map<String, Object> map = new HashMap<>();
                if ("coAdmin".equals(user.getLevel())) {
                    map.put("enVisible", "1");
                }
                List<Permission> list = permissionService.findAll(map);
                result = new ProfileResult(user, list);
            }
            //构造方法：安全数据，密码，realm域名
            SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(result, user.getPassword(), this.getName());
            return info;
        }

        //返回null,会抛出异常，标识用户名和密码不匹配，通过过滤器跳转"/autherror?code=1"
        return null;
    }
}
```
* 权限认证：
```
package com.dst_common.shiro.realm;


import com.domain.system.response.ProfileResult;
import org.apache.shiro.authc.*;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;
import java.util.Set;

public class CustomRealm extends AuthorizingRealm {


    public void setName(String name) {
        super.setName("CustomRealm");
    }

    /**
     * 授权方法
     *
     * @param principals
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {

        //1.获取安全数据
        ProfileResult result = (ProfileResult) principals.getPrimaryPrincipal();
        //2.获取权限信息
        Set<String> apisPerms = (Set<String>) result.getRoles().get("apis");
        //3.构造权限数据，返回值
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        info.setStringPermissions(apisPerms);

        return info;
    }

    /**
     * 认证方法
     *
     * @param token
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        return null;
    }
}

```

5. 配置shiroConfig配置文件
```
package com.config;


import com.common.shiro.realm.UserRealm;
import com.dst_common.shiro.realm.CustomRealm;
import com.dst_common.shiro.session.CustomSessionManager;
import org.apache.shiro.mgt.SecurityManager;
import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.apache.shiro.web.session.mgt.DefaultWebSessionManager;
import org.crazycake.shiro.RedisCacheManager;
import org.crazycake.shiro.RedisManager;
import org.crazycake.shiro.RedisSessionDAO;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.LinkedHashMap;
import java.util.Map;

@Configuration
public class ShiroConfig {

    //1.创建realm
    @Bean
    public CustomRealm getRealm() {
        return new UserRealm();
    }

    //2.创建安全管理器
    @Bean
    public SecurityManager getSecurityManager(CustomRealm customRealm) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(customRealm);

        //将自定义的会话管理器注册到安全管理器
        securityManager.setSessionManager(sessionManager());

        //将自定义redis缓存管理器注册到安全管理器中
        securityManager.setCacheManager(cacheManager());

        return securityManager;
    }

    //3.配置shiro的过滤器工厂

    /**
     * 在web程序中，shiro进行权限控制全部是通过一组过滤器集合进行控制
     */
    @Bean
    public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager) {
        //1.创建过滤器工厂
        ShiroFilterFactoryBean factoryBean = new ShiroFilterFactoryBean();
        //2.设置安全管理器
        factoryBean.setSecurityManager(securityManager);
        //3.通用配置(跳转登录页面，未授权跳转的页面)
        factoryBean.setLoginUrl("/autherror?code=1");//如果未登录，跳转未登录的页面
        factoryBean.setUnauthorizedUrl("/autherror?code=2");//如果未授权，跳转到未授权的页面
        //4.设置过滤器集合（ 过滤链定义，从上向下顺序执行，一般将/**放在最为下边）使用LinkedHashMap
        Map<String, String> filterMap = new LinkedHashMap<>();
        /**
         * "anon":无参，开放权限，可以理解为匿名用户或游客
         * "authc":无参，需要认证
         * "perms[User]":参数可写多个，表示需要某个或某些权限才能通过，多个参数时写 perms[“user,admin”]，
         * 当有多个参数时必须每个参数都通过才算通过
         */
        //设置登录时过滤器为anon，游客访问
        //测试swagger使用
        filterMap.put("/swagger-ui.html","anon");
        filterMap.put("/swagger-resources/**","anon");
        filterMap.put("/v2/api-docs/**","anon");
        filterMap.put("/webjars/springfox-swagger-ui/**","anon");
        //登录页面开放
        filterMap.put("/sys/login", "anon");
        //公共错误页面也需要开放
        filterMap.put("/autherror", "anon");
        //其他页面需要登录后才能访问，设置为authc
        filterMap.put("/**", "authc");
        //权限通过在方法上注解，进行授权
        //注入过滤器集合
        factoryBean.setFilterChainDefinitionMap(filterMap);

        //返回数据结果
        return factoryBean;

    }

    /**
     * @return
     */
    public DefaultWebSessionManager sessionManager() {
        CustomSessionManager sessionManager = new CustomSessionManager();
        sessionManager.setSessionDAO(redisSessionDAO());
        //禁用cookie与url重写（可以不禁用）
        sessionManager.setSessionIdCookieEnabled(false);
    /*
        //警用url重写 url;jsessionid=id(不美观)
        sessionManager.setSessionIdUrlRewritingEnabled(false);
    */
        return sessionManager;

    }

    /**
     * sessionDao
     * @return
     */
    public RedisSessionDAO redisSessionDAO(){
        RedisSessionDAO sessionDAO=new RedisSessionDAO();
        sessionDAO.setRedisManager(redisManager());
        return sessionDAO;
    }

    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private int port;



    /**
     * 设置redis的控制器，操作redis
     * @return
     */
    public RedisManager redisManager(){
        RedisManager redisManager=new RedisManager();
        redisManager.setHost(host);
        redisManager.setPort(port);
        return redisManager;
    }

    public RedisCacheManager cacheManager(){
        RedisCacheManager redisCacheManager=new RedisCacheManager();
        redisCacheManager.setRedisManager(redisManager());
        return redisCacheManager;
    }


    //开启对shiro注解的支持
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }

}

```

### 核心思路
1. 为满足跨域请求，需要将shiroConfig（配置）中的session替换为redis替代，（此时需要在.yml配置相关端口），shiro进行权限认证主要是基于realm，所以需要在realm对doGetAuthorizationInfo（权限认证），doGetAuthenticationInfo（登录认证）。
其中shiroConfig配置包括：
* 1.创建realm
* 2.创建安全管理器
* 3.配置shiro的过滤器工厂
   


### 详细过程

1. 配置pom与yml
2. 设置shiroConfig,CustomSessionManager,realm
3. 在这里对权限的管理不在会话管理器中设置，采用注解的方式进行权限管理。
```
 @RequiresPermissions("api-user-delete")
    @RequestMapping(value = "/user/{id}", method = RequestMethod.DELETE, name = "api-user-delete")
    public Result delete(@PathVariable(value = "id") String id) {
        userService.deleteById(id);
        return new Result(ResultCode.SUCCESS);
    }
```
4. 登录分为两个步骤：
* 对登录用户和密码进行验证
```
@ApiOperation(value = "用户登录")
    @RequestMapping(value = "/login",method = RequestMethod.POST)
    public Result login(@RequestBody Map<String,String> loginMap) {

        //登录账号
        String workNumber=loginMap.get("workNumber");
        //登录密码
        String password=loginMap.get("password");

        try {
            //1.构造登录令牌 UsernamePasswordToken
            //加密密码(密码，盐，加密次数)
            password=new Md5Hash(password,workNumber,3).toString();//
            UsernamePasswordToken upToken=new UsernamePasswordToken(workNumber,password);
            //2.获取subject
            Subject subject = SecurityUtils.getSubject();
            //3.调用login方法，进入realm完成认证
            subject.login(upToken);
            //4.获取sessionID
            String sessionId = (String)subject.getSession().getId();
            //构造返回结果，这里需要返回sessionID到前端
            return new Result(ResultCode.SUCCESS,sessionId);
        } catch (Exception e) {
            return new Result(ResultCode.MOBILEORPASSWORDERROR);
        }
    }
```
* 根据sessionId返回用户数据
```
 /**
     *  1.获取用户id
     *  2.根据用户id查询用户
     *  3.构建返回对象
     *  4.响应
     * @param request
     * @return
     */
    @ApiOperation(value = "用户登录后获取信息")
    @RequestMapping(value = "/profile",method = RequestMethod.POST)
    public Result profile(HttpServletRequest request){
//        /**
//         * 从请求头信息中获取token数据 jwt
//         * 1.获取请求头信息：名称=Authorization
//         * 2.替换Bearer+空格
//         * 3.解析token
//         * 4.获取clamis
//         */


        //1.获取session中的安全数据
        Subject subject = SecurityUtils.getSubject();
        //2.subject获取所有的安全数据
        PrincipalCollection principals = subject.getPrincipals();
        //3.获取安全数据
        ProfileResult result = (ProfileResult)principals.getPrimaryPrincipal();
        //返回结果
        return new Result(ResultCode.SUCCESS,result);
    }
```
## 总结
注意对shiroConfig以及数据交互层的撰写。别忘要开启对shiro的注解支持。
```
//开启对shiro注解的支持
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }
```

