---
layout: post
title:  "[Spring-Security] Spring Security 공부(1)"
author: kau-newbie
categories: [Spring-Security]
image: ./assets/images/Springicon.png
---

요약: Spring Security란 무엇일까

막간 상식
> jakarta가 뭘까?
> 
> 젬나이의 말에 따르면, Oracle에서 Eclipse로 서블릿(Servlet), JSP, JPA 등의 표준 자바 웹 기술들을 넘길 때 javax 말고 다른 이름을 쓰라함. (JAVA EE라고 한다.)
> 
> --> 투표로 바꾼 이름이 JAKARTA EE. 때문에 라이브러리 자체도 javax.xx --> jakarta.xx로 바꿨다고 한다.

# Spring Security란

Spring에서 인증과 인가를 위한 기능들을 뜻한다. 물론 api로 제공하고 있다.

## Container란

여기서 말하는 container는 도커 컨테이너처럼 분리된 실행환경이 아닌,

각종 객체를 적절한 시기에 만들고, DI하거나, 생명주기를 관리하는 시스템(프로그램)이다.
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
([자바웹표준기술문서에 따르면](https://jakarta.ee/specifications/servlet/4.0/apidocs/javax/servlet/filter) this method allows the Filter to pass on the request and response to the next entity in the chain.라고 한다.)
- [FilterChain](https://docs.oracle.com/javaee/7/api/javax/servlet/FilterChain.html)은 일련의(chain) filter들을 가리킨다고 보면 될 것 같다.

이때, Spring에서는 `DelegatingFilterProxy`란 특별한 필터를 제공한다.
[공식문서설명](https://docs.spring.io/spring-security/reference/servlet/architecture.html)

```java

public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
	Filter delegate = getFilterBean(someBeanName);
	delegate.doFilter(request, response, chain);
}

// 1. Lazily get Filter that was registered as a Spring Bean. For the example in DelegatingFilterProxy is an instance of delegateBean Filter0.
// 2. Delegate work to the Spring Bean. (delegate = 대리하다)

```

공식 문서의 pseudo code를 보면, servlet container에서 bean을 실행시키고 있음을 알 수 있다.

정확히는

### 서블릿 컨테이너와 스프링 컨테이너를 거치는 전반적인 과정

아래 그림과 같다.

![spring컨테이너와servlet컨테이너전체흐름1](./assets/images/forPost/Spring/spring-security.png)

1. 사용자의 요청(http)이 tomcat으로 들어온다.
2. tomcat에서는 필터링 기능(chain of filters)을 수행한다.
3. 그 중, Spring에서 제공한 `DelegatingFilterProxy`가 `Spring Security`기능으로 이어준다.
    1. Spring Container 안에서는 Bean들이 동작하기 때문에, 이 Spring Security만의 Filter들은 Bean이다.
    2. 사용자 요청은 Bean들(각각의 security filter들)을 거친다.
    3. 모두 통과하면 다시 chain of filters의 나머지 필터로 돌아간다.(서블릿 컨테이너가 인식할 수 있는 다른 나머지 필터들)
4. 최종 필터를 통과한 이후엔 드디어 서블릿을 실행한다.
5. Spring에서는 `DispatcherServlet`이 요청에 맞는(e.g. "/home") controller를 실행함으로써 사용자 요청을 처리한다.
6. 사용자 요청 처리(요청에 맞는 자바 코드 실행) 후, 응답은 이 역순으로 진행된다.


(writing...)