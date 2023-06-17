---
title: Servlet Container, Servlet, Filter 와 Spring Boot에서 사용되는 원리
date: 2023-06-17 14:00:00 +0900
categories: [Spring, Web]
tags: [spring] # TAG names should always be lowercase
toc: true
---

## 서론
 이번 글은 내가 Spring Boot에서 Filter를 등록하기 위해 방법을 알아보던 중, 이제까지 제대로 이해 못하고 있던 관련된 내용들이 계속해서 이어져 나와서 공부하게 되었고 해당 내용을 정리하려 한다.

## Java Servlets API
 자바 서블릿 API는 서블릿을 작성하기 위한 클래스와 인터페이스를 제공하는데, 여기에는 `HttpServletRequest`, `HttpServletResponse`, `Servlet`, `Filter`, `Listener` 등이 포함되어 있고 이를 통해 request와 response를 다루고 task를 수행할 수 있다. Servlet API는 Java EE platform에 포함되어 있다. 여기에서 중요한 개념인 `Servlet Container`, `Servlet`, `Filter` 등에 대해 알아보자.

### Servlet[^footnote2]
> 서블릿은 클라이언트로부터 요청을 받고 응답을 하는 데 쓰이는 클래스
{: .prompt-info }

 서블릿에 대한 설명 중 이 한문장이 제일 간단하면서도 와닿았다. 서블릿은 그냥 클래스다. 처음 이 개념에 대해 배울 때 갑자기 많은 용어들이 나와서 엄청 복잡해 보였는데 저 설명이 그 복잡함을 일축시켜주었다. Servlet의 라이프 사이클은 `init()`, `service()`, `destroy()`[^footnote3] 가 있다고 하는데 이 부분에 대해 세세하게 파악해야 할 필요가 있는가 싶긴 하고(<s>이름만 봐도 뭐하는 건지는 알 것같으니,,</s>), 서블릿 컨테이너에 의해 초기에 생성되고 요청별로 재사용 된다고 한다. (물론 중간에 바꿀 수도 있다)
- 서블릿에 대한 추가적인 설명
  - `Servlet`은 generic interface이고 `HttpServlet`은 HTTP에 특정하게 적용되는 확장하는 sub interface임
  - Servlet 기술은 JSP와 Spring MVC와 같은 웹 기술에서 핵심적인 기술


### Servlet Container
> 서블릿 컨테이너는 서블릿을 제어하는 자바 어플리케이션.
{: .prompt-info }

 다른 어떤 설명보다도 위의 설명이 제일 간단했고 예시로는 Tomcat, Jetty 등이 있다.
 서블릿 컨테이너가 제공해주는 추가적인 기능은 아래와 같다.
 - Commnication Support <br>
 서블릿이 웹 서버와 통신하는 것을 쉽게 해줌. ServerSocket 생성, port listening, stream 생성 등의 작업을 맡아서 진행
 - Servlet Lifecycle Management <br>
 서블릿의 생명주기 관리(`init()`, `service()`, `destroy()`)를 해줌으로 자원 관리에 대한 걱정을 할 필요 없음
 - Multithreading Support <br>
 요청을 받을 때 마다 자바 스레드를 컨테이너가 만들고, 요청이 완료되면 스레드가 종료됨. Thread Safety에 대해서는 책임져 주진 않으니 주의해야함.
 - Declarative Security <br>
  보안 관련 설정을 하드 코딩 하지 않고 web.xml 파일로 할 수 있도록 도와줌
 - JSP Support <br>
  JSP Code를 Java로 바꿔주는 역할

### Filter
> Servlet과 함께 사용되며, 요청, 응답 데이터를 전처리하거나 후처리 할 수 있음
{: .prompt-info }
 - 필터는 주로 요청/응답의 헤더 변경, 인증, 로깅과 같은 비지니스 로직과는 다른 공통적인 관심사 처리를 위해 사용됨
 - Spring Boot Framework에서 Client로부터 오는 요청/응답에 대해서 최초/최종 단계의 위치에 존재함

## Spring Boot에 적용
### Servlet을 Spring Boot에 추가하는 방법
 `@WebServlet`과 `@ServletComponentScan`[^footnote]을 이용해서 Servlet 등록이 가능하다. 그러나 우리가 굳이 Servlet을 등록할 일은 없다. 왜냐하면 Spring Boot는 `FrontControllerPattern`으로 구현되어 있어 자동으로 등록되는 Bean인 `DispatcherServlet`에서 어떤 `Controller`로 매핑되는지 확인후 요청을 넘겨주기 때문에 신경쓸 일이 많이 없다. 자세하게 어떻게 등록되어 있고, 어떤 순서로 호출되는지는 조금 뒤에 살펴보자

### Filter를 Spring Boot에 추가하는 방법[^footnote1]
1. `FilterRegistrationBean`을 통해 세밀한 설정으로 Filter 추가 (Bean 등록)
```java
@Configuration
public class MyFilterConfig {

    @Bean
    public FilterRegistrationBean<MyFilter> myFilterRegistrationBean() {
        FilterRegistrationBean<MyFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new MyFilter());
        registrationBean.addUrlPatterns("/*"); // Set the URL patterns for which the filter should be applied
        registrationBean.setOrder(1); // Set the order in which the filter should be executed (if multiple filters are registered)
        return registrationBean;
    }
}
```
2. `Filter` 인터페이스를 구현한 클래스에 @Component를 붙여서 Bean으로 등록하여 추가 (Bean 등록)
```java
@Component
@Order(1)
public class TransactionFilter implements Filter {

    @Override
    public void doFilter(
      ServletRequest request, 
      ServletResponse response, 
      FilterChain chain) throws IOException, ServletException {
 
        HttpServletRequest req = (HttpServletRequest) request;
        LOG.info(
          "Starting a transaction for req : {}", 
          req.getRequestURI());
 
        chain.doFilter(request, response);
        LOG.info(
          "Committing a transaction for req : {}", 
          req.getRequestURI());
    }

    // other methods 
}
```
3. `@ServletComponentScan`과 `@WebFilter` 어노테이션으로 Filter 추가 (claspath scanning)
- `@ServletComponentScan` 어노테이션을 `@Configuration`이 붙은 클래스나, `@SpringBootApplication` 이 붙은 클래스에 달아 주면 `@WebServlet`, `@WebFilter`, `@WebListener` 가 붙은 클래스를 자동으로 내장된 서블릿 컨테이너에 등록해준다.(빈으로 등록된다고 이해하면 됨)

```java
@SpringBootApplication
@EnableFeignClients
@EnableJpaAuditing
@ServletComponentScan
public class VirtualofficeApplication {

	public static void main(String[] args) {
		SpringApplication.run(VirtualofficeApplication.class, args);
	}

}

@WebFilter(urlPatterns = {"/*"})
@RequiredArgsConstructor
public class AddUnityPacketIdFilter implements Filter {

	private final ObjectMapper objectMapper = new ObjectMapper();
	private final String packetIdFieldName = "packetId";
  ...
}
```

### Spring Boot에서 request와 response 처리 흐름도
![spring-request-flow](/assets/images/spring-request-flow.png)
위의 이미지를 참고해서 실제적으로 어떤 클래스들이 호출되고 처리되는지 살펴볼 것이다.

draw.io 로 흐름도 그려서 넣고, 번호 넣어서
번호 별로 이미지 첨부해서 설명하는 식으로 글 작성하기

## 참고
[^footnote]: [Baeldung, The @ServletComponentScan Annotation in Spring Boot](https://www.baeldung.com/spring-servletcomponentscan)
[^footnote1]: [Baeldung, How to Define a Spring Boot Filter](https://www.baeldung.com/spring-boot-add-filter)
[^footnote2]: [블로그, 서블릿이란?](https://mangkyu.tistory.com/14)
[^footnote3]: [Baeldung, Introduction to Servlets](https://www.baeldung.com/intro-to-servlets)
<!-- [^footnote1]: [Baeldung, The @ServletComponentScan Annotations in Spring Boot](https://www.baeldung.com/spring-servletcomponentscan) -->
