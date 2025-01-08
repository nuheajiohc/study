# DispatcherServet 만들 때 매핑 주의할 점

스프링의 **DispatcherServlet**을 서블릿으로 구현할 때 과거에는 `*.do` 방식으로 요청을 가로챘기 때문에 상관없었지만, 루트 경로를 활용해서 모든 요청을 가로채는 방식에는 정적 리소스와 관련해서 주의해야 할 점이 있다. 
이 글에서는 주의할 점과 문제를 해결하는 방법, 그리고 실제 스프링에서 매핑을 처리하는 방식에 대해 알아보겠다. 


## 1. 과거의 '*.do' 방식
과거에는 `*.do`로 요청을 가로채는 방식으로 **DispatcherServlet**을 만들었기 때문에 JSP나 정적 리소스 요청과 서블릿 요청이 섞이지 않았다.  
정적리소스는 **DefaultServlet**이 처리하고, jsp는 **JspServlet**이 처리하고 동적 요청은 `*.do` 끝나기 때문에 **DispatherServlet**이 처리했다.

그런데 이 방식을 현대에 와서는 사용하지 않는다. 그 이유는 다음과 같다.  
- 현대에는 RESTful하게 URL을 설계하기 때문이다.
- 예: `/users.do` 대신 `/users`로 표현한다.
- RESTful한 방식이 더 직관적이다.

그래서 매핑 방식을 `/*` 과 `/`으로 차례대로 설정해보았다. 그랬더니 문제점이 생겼는데 바로 설명해 보겠다.


## 2. "/*"와 "/" 매핑 이슈

### 2.1 "/*"로 매핑한 경우
`/*`은 모든 요청을 가로채기 때문에 다음과 같은 문제가 발생한다.

1. JSP 파일이 로드되지 않음
   - JSP 요청은 서블릿 컨테이너의 **JspServlet**이 처리해야 하는데 모든 요청을 DispatcherSerlvet이 가로채서 Jsp 요청이 전달되지 않는다.
   - **JspServet**의 매핑 경로는 `*.jsp`로 설정되어 있는데 `/*`으로 모든 요청을 가로채버리기 때문이다.
2. 정적 리소스가 로드되지 않음
   - HTML, CSS, JS, 이미지 등 정적 리소스 요청은 **DefaultServlet**이 처리해야 하지만, 이 요청들도 **DispatcherServlet**이 가로채버린다.
   - **DefaultServlet**의 매핑 경로는 `/`로 설정되어 있는데 `/*`으로 모든 요청을 가로채버리기 때문이다.

### 2.2 "/"로 매핑한 경우
그래서 `/`로 수정해보았지만 여전히 정적 리소스는 로드되지 않는다.
1. JSP 요청 처리는 가능
   - JSP 요청(`*.jsp`)은 **JspServlet**이 매핑을 유지하므로 JSP 파일은 정상적으로 로드된다.
2. 정적 리소스 요청 문제
   - 여전히 정적 리소스 요청은 **DispatcherServlet**이 가로채므로 **DefaultServlet**이 이를 처리하지 못한다.

## 3. 해결 방법

### 3.1 DefaultServlet으로 요청 위임
**DispatcherServlet** 안에서 정적 리소스 요청을 확인하고 해당 요청을 **DefaultServlet**으로 전달하여 처리한다.

```java
@WebServlet("/")
public class DispatcherServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        // 요청 URI에서 Context Path를 제거한 경로
        String path = request.getRequestURI().substring(request.getContextPath().length());

        // 정적 리소스 경로 확인
        if (path.startsWith("/static/") ||
            path.startsWith("/css/") ||
            path.startsWith("/js/") ||
            path.startsWith("/images/")) {
            // DefaultServlet으로 요청 위임
            RequestDispatcher dispatcher = request.getServletContext().getNamedDispatcher("default");
            if (dispatcher != null) {
                dispatcher.forward(request, response);
                return;
            }
        }

        // DispatcherServlet의 나머지 로직 처리
        // ...
    }
}

```

### 3.2 DispatcherServlet에서 직접 정적 리소스 로직을 처리
**DispatcherServlet**에서 정적 리소스 처리 로직을 직접 만들 수도 있다.  
하지만 이미 **DefaultServlet**에 잘 구현되어 있기 때문에 위임을 해주는 것이 좋을 것 같다.


## 4. 실제 Spring DispatcherServlet에서 정적 리소스 처리 방식

### 4.1 DispatcherServlet이 정적 리소스를 직접 처리
Spring에서는 **addResourceHandlers()**를 사용하여 정적 리소스를 직접 처리한다.

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
  @Override
  public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/static/**")
      .addResourceLocations("classpath:/static/")
      .setCachePeriod(3600); // 클라이언트 캐싱 (3600초)
  }
}
```

### 4.2 DefaultServlet으로 위임
Spring에서는 **DispatcherServlet**에서 **DefaultServlet**으로 정적 리소스 요청을 위임할 수도 있다. 여기에는 두가지 방법이 있다.

#### 4.2.1 Java Config 설정
```java

@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
  @Override
  public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
    configurer.enable();
  }
}
```

#### 4.2.2 servlet-context.xml 설정
```xml
<mvc:default-servlet-handler />
```


> 나의 경우에는 DefaultServlet에 책임을 위임하는 방식을 구현했다.


## 같이 보면 좋은 자료
JspServlet과 매핑에 관한 내용은 아래 링크들에 자세하게 설명해두었다.

- [JSP 동작구조.md](../../Back-End/Servlet/JSP%20%EB%8F%99%EC%9E%91%EA%B5%AC%EC%A1%B0.md)
- [Jsp 사용할 때 루트 경로에서 index.jsp가 작동하는 이유.md](../../Back-End/Servlet/Jsp%20%EC%82%AC%EC%9A%A9%ED%95%A0%20%EB%95%8C%20%EB%A3%A8%ED%8A%B8%20%EA%B2%BD%EB%A1%9C%EC%97%90%EC%84%9C%20index.jsp%EA%B0%80%20%EC%9E%91%EB%8F%99%ED%95%98%EB%8A%94%20%EC%9D%B4%EC%9C%A0.md)
- [서블릿 매핑 우선순위.md](../../Back-End/Servlet/%EC%84%9C%EB%B8%94%EB%A6%BF%20%EB%A7%A4%ED%95%91%20%EC%9A%B0%EC%84%A0%EC%88%9C%EC%9C%84.md)
