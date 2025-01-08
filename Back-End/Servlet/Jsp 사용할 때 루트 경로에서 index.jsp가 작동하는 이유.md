# Jsp 사용할 때 루트 경로에서 index.jsp가 작동하는 이유

Jsp 기반 웹 애플리케이션을 개발할 때, **webapp** 디렉토리 아래에 **index.jsp** 파일만 있어도 루트 경로(`/`)로 접근 시 자동으로 **index.jsp**가 실행된다.  

먼저 간단하게 설명하자면 index 파일 처리 과정은 아래와 같다.  
1. 루트 경로("/")로 요청이 들어오면, 톰캣은 **웰컴 파일 목록(welcome-file-list)**을 먼저 확인한다.
2. **웰컴파일**이 발견되면, 해당 파일로 **내부 포워드**하여 처리한다.  
3. 파일이 **jsp**확장자라면 **JspServlet**이 처리하고, **html**이라면 **DefaultServlet**이 처리한다.


## 1. 톰캣의 웰컴 파일 목록
톰캣은 기본적으로 `conf/web.xml` 파일에 아래와 같이 **웰컴 파일 목록**을 정의해둔다.
```xml
    <welcome-file-list>
        <welcome-file>index.html</welcome-file>
        <welcome-file>index.htm</welcome-file>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
```
- 루트 경로로 요청이 들어올 때, 컨테이너(톰캣)는 위 순서대로 파일을 찾는다.


## 2. 루트 요청의 처리 흐름
1. `/` 요청
   - 톰캣은 해당 경로의 요청이 오면 `/webapp` 디렉토리 내에서 웰컴 파일 리스트 순서대로 찾는다.
2. 내부 포워드
   - `/`에서 `/index.jsp`로 서버 내부에서 요청 경로를 변경한다.
   - 클라이언트 입장에서는 `/`로 요청했지만 실제 처리는 `/index.jsp`가 담당한다.
   - 이것은 리다이렉트가 아니라 포워드이기 때문에 브라우저 URL은 바뀌지 않는다.
3. JspServlet에서 처리
   - 이제 요청 경로가 `/index.jsp`이기 떄문에 ***.jsp** 매핑을 담당하는  **JspServlet**이 이 파일을 컴파일하고 실행한다.
   - `/index.html` 일 경우에는 DefaultSerlvet이 이 파일을 처리한다.
   - [JSP 동작구조.md](JSP%20%EB%8F%99%EC%9E%91%EA%B5%AC%EC%A1%B0.md)에서 JspServlet에 대한 내용을 확인할 수 있다.

## 3. DefaultServlet

톰캣은 기본적으로 `conf/web.xml` 파일에 아래와 같이 **DefaultServlet** 매핑 정보를 작성해두었다.
```xml
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    ...
</servlet>

<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

- **DefaultServlet**은 주로 정적 리소스(HTML, CSS, JS, 이미지)를 처리하기 위한 기본 서블릿이다.
- `/` 경로로 매핑되어 있어서 다른 서블릿이나 매핑이 처리하지 않는 요청을 최종적으로 받아낸다.

즉,
- 웰컴 파일이 **index.jsp**인 경우
  → 톰캣이 `/index.jsp`로 포워드하고 **JspServlet**이 처리
- 웰컴 파일이 **index.html**인 경우
  → 톰캣이 `/index.html`로 포워드하고 **DefaultServlet**이 처리

> 루트 요청이 있더라도 웰컴 파일의 유형에 따라 처리하는 서블릿이 다르다.

## 4. "/"요청을 다른 파일로 사용하려면?
루트 경로에 있는 `index.jsp`가 아닌, 다른 jsp나 서블릿으로 루트 경로를 처리하고 싶다면 아래 두 가지 방법 중 하나를 사용한다.

- 기존의 `index.jsp`를 삭제한 후 원하는 위치에 jsp파일을 생성하고 서블릿으로 `/`경로를 매핑하여 포워딩한다.
- 웹 애플리케이션의 `web.xml`에서 웰컴파일리스트를 작성한다.

