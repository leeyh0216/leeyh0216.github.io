---
layout: post
title:  "Spring Security - Introduction, Architecture"
date:   2019-05-30 10:00:00 +0900
author: leeyh0216
categories: spring-security
---

# Introduction

## Spring Security란?

Spring Security는 Java EE 기반의 엔터프라이즈 어플리케이션에 보안 서비스를 제공한다.

보안 영역에는 `Authentication(인증)`과 `Authorization(인가)`가 존재한다.

`Authentication`의 경우 요청자(사용자, 기기, 시스템 등)가 실제 사용자(**Principal**)가 맞는지를 검증하는 과정이고, `Authorization`의 경우 사용자가 특정 행위를 할 수 있는지를 결정하는 과정이다.

Spring Security에서는 다양한 방법으로 인증과 인가 기능을 제공한다.

* Http basic authentication header
* LDAP
* OpenID
* Kerberos

# Technical Overview

## Core Components

### UserDetails와 UserDetailsService

`UserDetails`는 사용자 정보를 저장하는 객체가 구현해야하는 인터페이스이며, Application에 맞게 확장하여 사용할 수 있다. `UserDetails`는 우리가 데이터베이스에 저장할 사용자 객체인 동시에 `SecurityContextHolder`에서도 내부적으로 사용된다.

> [UserDetails](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/core/userdetails/UserDetails.html) 페이지를 확인해보면 기본적으로 username, password, autoorities 등의 정보를 필드로 가지고 있는 것을 확인할 수 있다.

`UserDetails`를 관리하는 서비스는 `UserDetailsService` 인터페이스를 구현해야 하며, 반드시 `loadUserByUsername` 메소드를 구현해야 한다.

아래와 같이 `UserDetails`와 `UserDetailsService`를 구현하여 테스트를 진행하였다.

`Customer`
```
package com.leeyh0216.spring.security.examples.userdetailsservice;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

public class Customer implements UserDetails {

    public String username;

    public String password;

    public List<GrantedAuthority> authorities;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return authorities;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return false;
    }

    @Override
    public boolean isAccountNonLocked() {
        return false;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return false;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

`CustomerService`
```
package com.leeyh0216.spring.security.examples.userdetailsservice;

import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service
public class CustomerService implements UserDetailsService {

    List<Customer> customers = new ArrayList<>();

    public Customer createCustomer(Customer customer) throws Exception{
        for(Customer c: customers){
            if(c.username.equals(customer.username))
                throw new Exception("Username already exists.");
        }
        customers.add(customer);
        return customer;
    }

    @Override
    public Customer loadUserByUsername(String username) throws UsernameNotFoundException {
        Customer customer = null;
        for(Customer c: customers){
            if(c.username.equals(username)){
                customer = c;
                break;
            }
        }
        if(customer == null)
            throw new UsernameNotFoundException("Cannot find user who's name is "+username);
        else
            return customer;
    }
}
```

위의 경우 직접 `UserDetailsService`를 상속하여 구현하였지만 JDBC, LDAP 등을 쉽게 연동할 수 있는 방법도 제공하고 있다.

### SecurityContextHolder, SecurityContext and Authentication Objects

`SecurityContextHolder`는 현재 접속 중인 사용자의 Principal 정보를 가진 `SecurityContext`를 저장하는 객체이다.

기본적으로 `SecurityContextHolder`는 ThreadLocal에 정보를 저장하기 때문에, 같은 Thread에서 동작하는 코드들은 동일한 `SecurityContext` 객체에 접근할 수 있다. 또한 (ThreadLocal 기반으로 동작하기 때문에) 각 코드가 명시적으로 `SecurityContextHolder` 객체를 전달받지 않아도 접근이 가능하다.

만일 ThreadLocal을 사용할 수 없는 상황(기능 실행 중 새로운 Thread 생성 하는 등)의 경우 `SecurityContextHolder`의 저장 방식을 `SecurityContextHolder.MODE_THREADLOCAL`(기본 설정)에서 `SecurityContextHolder.MODE_INTERITABLETHREADLOCAL`로 변경하면 된다. 저장방식 변경은 SystemProperty로 설정하거나 static method인 `SecurityContextHolder.setStrategyName(String strategyName)`을 호출하면 된다.

#### Obtaining information about the current user

`SpringContextHolder`에는 현재 사용자의 정보를 `Authentication` 객체에 저장하고 있다. 아래와 같은 방법으로 현재 사용자의 `Authentication` 객체의 Principal을 얻을 수 있다.

```
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
if (principal instanceof UserDetails) { 
    String username = ((UserDetails)principal).getUsername(); 
} 
else { 
    String username = principal.toString();
}
```

다만, 위 예제를 `SpringApplication.run` 메소드 아래에 넣는 경우 현재 사용자가 없어 Principal을 얻어오는 과정에서 `Null Pointer Exception`이 발생할 수 있으므로, 아래와 같이 `WebSecurityConfigurerAdapter`와 `Controller`를 구성하여 테스트해보았다.

`Application`
```
package com.leeyh0216.spring.security.examples.securitycontextholder;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;

@SpringBootApplication
public class Application {
    public static void main(String[] args) throws Exception{
        SpringApplication.run(Application.class, args);
    }
}

```

`SampleController`
```
package com.leeyh0216.spring.security.examples.securitycontextholder;

import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SampleController {

    @GetMapping("/")
    public String helloWorld(){
        Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if(principal instanceof UserDetails){
            UserDetails userDetails = (UserDetails)principal;
            return String.format("Principal: %s, %s, %s", userDetails.getUsername(), userDetails.getPassword(), userDetails.getAuthorities());
        }
        else
            return principal.toString();
    }
}
```

`WebSecurityConfig`
```
package com.leeyh0216.spring.security.examples.securitycontextholder;

import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .httpBasic();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        manager.createUser(User.withUsername("user").password("{noop}password").roles("USER").build());
        return manager;
    }
}
```

> 비밀번호 앞에 {noop}를 붙이지 않으면 암호화된 형태로 전송해야 하므로, 평문으로 전송 및 인증을 수행하는 {noop} 키워드를 앞에 붙인다.

위 예제를 실행 후 `http://localhost:8080`을 호출 후 ID, PASSWORD를 넣고 로그인하는 경우 아래와 같이 페이지가 출력되는 것을 확인할 수 있다.

```
Principal: user, null, [ROLE_USER]
```

