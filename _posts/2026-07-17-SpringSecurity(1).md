---
layout: post
title:  "[Spring-Security] Spring Security 공부(1)"
author: kau-newbie
categories: [Spring-Security]
image: assets/images/forPost/Springicon.png
---

요약: Spring Security란 무엇일까

막간 상식
> jakarta가 뭘까?
> 
> 젬나이의 말에 따르면, Oracle에서 Eclipse로 서블릿(Servlet), JSP, JPA 등의 표준 자바 웹 기술들을 넘길 때 javax 말고 다른 이름을 쓰라함. (JAVA EE라고 한다.)
> 
> --> 투표로 바꾼 이름이 JAKARTA EE. 때문에 라이브러리 자체도 javax.xx --> jakarta.xx로 바꿨다고 한다.

# Spring Security란

- Spring에서 인증과 인가를 위한 기능들을 뜻한다. 물론 api로 제공하고 있다.

    - 여기서 인증(Authentication)이란, 사용자가 '누구인지' 확인하는 것을 말하고,
    - 인가(Authorization)란, 사용자가 '어떤 권한'을 갖는지 정하는 것을 말한다.
    > 예를 들면, 인증은 로그인이고, 인가란 해당 계정이 접근할 수 있는 리소스들(계정 정보조회 등등)을 관리(허용/거부 등)하는 것을 말한다.

- 기본적으로 Spring의 한 부분으로(한 부분이란 말은 Spring에서 공식적으로 제공하는 기능 중 하나란 뜻이다.), Spring Container에서 동작한다.

## Container란

여기서 말하는 container는 도커 컨테이너처럼 분리된 실행환경이 아닌,

각종 객체(Bean이라 부른다!)를 적절한 시기에 만들고, DI하거나, 생명주기를 관리하는 시스템(프로그램)이다.
- 이런 정의에서는 안드로이드 os도 컨테이너로 봐도 된다.

전통적으로 자바 웹 서버를 담당하던 '서블릿' 컨테이너가 있고, 이를 편리하게 사용하기 위해 나온 'Spring'만의 Spring 컨테이너가 있다.
- 대표적인 서블릿 컨테이너로는 '톰캣(Tomcat)'이 있다.

둘은 별개이므로, Spring Container에서 사용하는 개념인 '빈(Bean)'을 서블릿 컨테이너는 다루지 못한다. 
- 예를 들면, Spring에서는 @Controller 처럼 annotation을 써두면, Spring 컨테이너가 이를 bean으로 만들어(객체 인스턴스) 가지고 있다가, 적절하게 DI하는 등, 알아서 사용해준다. 이를 서블릿 컨테이너는 하지 못한다.

바로 아래에서 이 문제의 해결책을 제시하고 있다.

## Spring Security의 흐름

기본적으로 자바 웹 서버는 사용자 요청에 대해 서블릿(Servlet)을 실행하는 구조로 동작한다.

여기서 서블릿이란, 자바 프로그램(.java --> .class)으로 사용자 요청에 대응하는 프로그램이 서블릿 컨테이너에 의해 컴파일 된 후, 실행되는 구조이다.

### Filter, doFilter

서블릿 컨테이너는 기존에 사용자 요청 한 개 당 서블릿을 한 개씩 실행했다. 이때, 사용자 요청의 이상탐지 같은 공통적인 일종의 '필터링' 기능들을 따로 빼서, 앞단에서 먼저 실행하게 했다. 

이때, 필터들은 용도에 따라 여러개가 연달아 존재하고, 이 일련의 필터링 과정을 `FilterChain` 이라고 한다.

이때 서블릿 컨테이너에서 filtering 기능들은, 즉 FilterChain을 활용하는 예시코드를 다음과 같이 소개한다.

[Spring공식문서(SpringSecurity)](https://docs.spring.io/spring-security/reference/servlet/architecture.html)

```java

@Override
public void doFilter(ServletRequest request, ServletResponse response,
		FilterChain chain) throws IOException, ServletException {
	// do something before the rest of the application
	chain.doFilter(request, response); // invoke the rest of the application
	// do something after the rest of the application
}

```

저 주석 `do something before the rest of the app`에서 원하는 작업을 하게 한 후, 계속해서 다음 필터(chainning에 속한 필터들)에게 FilterChain의 `doFilter`메서드를 통해 넘겨준다.
([자바웹표준기술문서에 따르면](https://jakarta.ee/specifications/servlet/4.0/apidocs/javax/servlet/filter) "this method allows the Filter to pass on the request and response to the next entity in the chain."이라고 한다.)
- [FilterChain](https://docs.oracle.com/javaee/7/api/javax/servlet/FilterChain.html)은 일련의(chain) filter들을 가리킨다고 보면 될 것 같다.

이때, Spring에서는 `DelegatingFilterProxy`란 특별한 필터를 제공한다.
- [공식문서설명](https://docs.spring.io/spring-security/reference/servlet/architecture.html)

```java

public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
	Filter delegate = getFilterBean(someBeanName);
	delegate.doFilter(request, response, chain);
}

// 1. Lazily get Filter that was registered as a Spring Bean. For the example in DelegatingFilterProxy is an instance of delegateBean Filter0.
// 2. Delegate work to the Spring Bean. (delegate = 대리하다)

```

공식 문서의 pseudo code를 보면, servlet container에서 bean을 실행시키고 있음을 알 수 있다.

정확히는 아래 그림과 같다.

### 서블릿 컨테이너와 스프링 컨테이너를 거치는 전반적인 과정

![spring컨테이너와servlet컨테이너전체흐름1](../assets/images/forPost/Spring/spring-security.png)

1. 사용자의 요청(http)이 tomcat으로 들어온다.
2. tomcat에서는 필터링 기능(chain of filters)을 수행한다.
3. 그 중, Spring에서 제공한 `DelegatingFilterProxy`가 `Spring Security`기능으로 이어준다.
    1. Spring Container 안에서는 Bean들이 동작하기 때문에, 이 Spring Security만의 Filter들은 Bean이다.
    2. 사용자 요청은 Bean들(각각의 security filter들)을 거친다.
    3. 모두 통과하면 다시 chain of filters의 나머지 필터로 돌아간다.(서블릿 컨테이너가 인식할 수 있는 다른 나머지 필터들)
4. 최종 필터를 통과한 이후엔 드디어 서블릿을 실행한다.
5. Spring에서는 `DispatcherServlet`이 요청에 맞는(e.g. "/home") controller를 실행함으로써 사용자 요청을 처리한다.
6. 사용자 요청 처리(요청에 맞는 자바 코드 실행) 후, 응답은 이 역순으로 진행된다.

### 코드로 살펴보자.

코드와 함께 이해해보자. 우선, 복잡한 것들은 다 무시하고, SecurityConfig 클래스 아래 @Bean으로 선언된(Spring Container에게 생명주기를 관리받는 객체라는 뜻이다.) `filterChain` 메서드를 주목하자.

해당 메서드는 `SecurityFilterChain` 타입의 객체를 반환한다. 아래 코드를 살펴보자.

```java

package com.example.demo.config; 

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
	// 1. 비밀번호를 안전하게 암호화해주는 도구(BCrypt)
    @Bean
    public PasswordEncoder passwordEncoder() {
        // 비밀번호를 데이터베이스에 안전하게 암호화하여 저장하기 위한 인코더
        return new BCryptPasswordEncoder();
    }
    // 2. 어떤 주소는 그냥 통과시키고, 어떤 주소는 로그인을 막을지 결정하는 필터 설정
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. 페이지 접근 권한 설정
            .authorizeHttpRequests(auth -> auth
                // 대시보드(/home), 정적 리소스(CSS, JS)는 로그인 없이도 누구나 접근 가능하도록 허용
                .requestMatchers("/home", "/css/**", "/js/**", "/images/**","/bug-report/**").permitAll()
                // 그 외 모든 요청은 로그인이 필요하도록 설정
                .anyRequest().authenticated()
            )
            // 2. 로그인 설정
            .formLogin(form -> form
                .loginPage("/home")             // 커스텀 로그인 페이지를 따로 두지 않고 /home(대시보드)을 로그인 페이지로 활용
                .loginProcessingUrl("/login")   // HTML의 <form action="/login" method="POST">가 던지는 로그인 요청을 가로채서 처리함
                .usernameParameter("username")
                .passwordParameter("password")
                .defaultSuccessUrl("/home", true) // 로그인 성공 시 대시보드로 이동
                .permitAll()
            )
            // 3. 로그아웃 설정
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/home") // 로그아웃 성공 시 대시보드로 이동
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
                .permitAll()
            )
            // 4. 개발 편의를 위한 CSRF 보안 비활성화 (실무에선 활성화하지만 학습용으로는 꺼두는 게 편리합니다)
            .csrf(csrf -> csrf.disable());

        return http.build();
    }
}

```

SecurityFilterChain는 [공식문서](https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/web/SecurityFilterChain.html)에 따르면, "Defines a filter chain which is capable of being matched against an HttpServletRequest."라고 한다.

즉, 사용자 http request에 대해 매칭된(사전에 request의 url과 매칭되는 필터묶음을 정의해놓는다.) 필터 chain(일련의 묶음)을 뜻한다.

http는 빌더 패턴으로, 최종적으로 .build() 메서드를 통해 SecurityFilterChain을 반환하게 되는데, 이때 `FilterChainProxy`가 이 SecurityFilterChain의 
- `matches(jakarta.servlet.http.HttpServletRequest request)`
- `getFilters()`

메서드들을 이용해서 각각 사용자의 http request의 url이 '이 SecurityFilterChain과 매칭되는지' 확인하고, true일 때 getFilters로 Filter들을 불러와서 필터링을 하게 되는 것이다. (마찬가지로 doFilter()로 계속 chainning할 것이다.)

`FilterChainProxy`는 또 뭔가, 할 수 있는데,

### FilterChainProxy와 DelegatingFilterProxy






이때, 의문이 들 수 있는 점이, SecurityFilterChain을 만들기도 전에 http의 메서드를 보면

```java

 http

            // 1. 페이지 접근 권한 설정

            .authorizeHttpRequests(auth -> auth

                // 대시보드(/home), 정적 리소스(CSS, JS)는 로그인 없이도 누구나 접근 가능하도록 허용

                .requestMatchers("/home", "/css/**", "/js/**", "/images/**","/bug-report/**").permitAll()

                // 그 외 모든 요청은 로그인이 필요하도록 설정

                .anyRequest().authenticated()

            )

```

와 같이 url들("/home", "/css/**", "/js/**", "/images/**","/bug-report/**")에 대해 매칭을 수행하는 것처럼 보인다.

하지만 이는 '매칭을 설정'해 놓는 것이다. 나중에 request가 들어왔을 때 이렇게 매칭하라는(.requestMathcers()) 메서드인 것이다.

마친 `authorizeHttpRequests()` 메서드의 공식 문서 설명도 다음과 같다.
> "Allows restricting access based upon the HttpServletRequest using RequestMatcher implementations (i.e. via URL patterns)."

공식 문서에 SecurityFilterChain은 interface로 정의됐고, 이 interface의 구현체(implements)는 DefaultSecurityFilterChain라고 써있다.

제미나이가 쉬운 설명을 위해 이 DefaultSecurityFilterChain을 간단히 코드로 적어보면 아래와 같다고 한다.

```java

// 빌더가 최종적으로 찍어내는 실제 구현체 모양 (DefaultSecurityFilterChain)
public class DefaultSecurityFilterChain implements SecurityFilterChain {

private final RequestMatcher requestMatcher; // 👈 1. 빌더가 넣어준 URI 기준 책갈피
private final List<Filter> filters; // 👈 2. 빌더가 넣어준 보안 필터 묶음

// 생성자
public DefaultSecurityFilterChain(RequestMatcher requestMatcher, List<Filter> filters) {
this.requestMatcher = requestMatcher;
this.filters = filters;
}

@Override
public boolean matches(HttpServletRequest request) {
// 실제 유저 요청이 들어왔을 때, 빌더가 넣어줬던 그 기준과 일치하는지 판별!
return this.requestMatcher.matches(request);
}

@Override
public List<Filter> getFilters() {
return this.filters;
}

```

저렇게 final 멤버변수(필드)로 requestMatcher와 filters라는 Filter의 List가 들어있음을 확인할 수 있다.

아무튼, 다시 돌아와서,



(writing...)

[httpsecurity](https://docs.spring.io/spring-security/reference/api/java/org/springframework/security/config/annotation/web/builders/HttpSecurity.html)