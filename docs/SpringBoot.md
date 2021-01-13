# SpringBoot

### 1 更改默认密码

**springboot启动的默认用户名是 username=user,密码是一个UUID，每次启动都不一样，在控制台打印输出**

#### 1.1 更改默认密码

- 配置文件配置

```xml
spring.security.user.name=aqiang
spring.security.user.password=123
```

- 通过配置类注入

首先配置类继承WebSecrityConfigureAdaper,提供加密算法，重写congfig方法

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    PasswordEncoder passwordEncoder() {
        return BCryptPasswordEncoder();
    }
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("javaboy.org")
                .password("123").roles("admin");
    }
}
```

