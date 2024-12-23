## 1. WAS(Web Application Server)란? 

WAS(Web Application Server)는 웹 애플리케이션을 실행하고 관리하는 미들웨어다.
- 클라이언트 요청 -> 애플리케이션 로직 실행 -> 결과 반환 의 과정을 담당
- 자바/스프링 기반의 경우 WAS를 서블릿 컨테이너라고 봐도 무방
- 주요 예시 : Tomcat, Jetty 등

### 1.1 WAS의 주요 기능
- 멀티 쓰레드 처리  
  많은 클라이언트 요청을 동시에 처리하기 위해 내부적으로 쓰레드를 관리
- 트랜잭션 관리  
  - 여러 작업(쿼리, 로직)이 하나의 논리적 단위로 묶였을 때, 데이터 무결성과 일관성을 보장
  - WAS에서 트랜잭션 환경을 제공(JNDI 데이터 소스 등)할 수 있지만, 실제 트랜잭션 처리는 보통 애플리케이션 레벨(Spring Framework 등)에서 관리하는 경우가 많다.
- 자원 관리  
  데이터베이스 연결 풀, 스레드 풀 등의 리소스를 효율적으로 관리
- 보안  
  인증, 권한 부여, HTTPS(SSL/TLS) 등의 기능을 제공해 안전한 통신을 지원
- 세션 관리  
  - 클라이언트와 서버 간의 상태를 관리하여, 쇼핑몰 장바구니나 사용자 로그인 상태 등을 유지
  - 톰캣은 내부적으로 `org.apache.catalina.session.StandardSession`이라는 클래스를 통해 세션을 관리 
  - 세션 데이터를 어디에 저장할지(메모리, DB, 분산 캐시 등)는 WAS 설정에 따라 결정

## 2. jakarta.servlet-api dependency의 역할

### 2.1 표준 인터페이스 제공
- `HttpServlet`, `Servlet`, `Filter`, `ServletRequest`, `ServletResponse` 등의 표준 인터페이스와 추상 클래스를 정의
- 이 표준이 있기에, **특정 WAS**(Tomcat, Jetty, etc.)에 **종속되지 않고** 동일한 코드를 여러 WAS에서 실행 가능

### 2.2 구현 분리
- `jakarta.servlet-api`는 인터페이스(또는 최소한의 추상 클래스 및 구현 클래스)만 제공
- 실제 구현체는 WAS가 담당

| 인터페이스               | Tomcat 구현체                                   | Jetty 구현체                          |
|---------------------|----------------------------------------------|------------------------------------|
| HttpServletRequest  | org.apache.catalina.connector.Request        | org.eclipse.jetty.server.Request   |
| HttpServletResponse | org.apache.catalina.connector.Response       | org.eclipse.jetty.server.Response  |
|HttpSession          | org.apache.catalina.session.StandardSession  |org.eclipse.jetty.server.session.Session |


## 3. 정리
- Tomcat 사용 시 org.apache.catalina.connector.Request / Jetty 사용 시 org.eclipse.jetty.server.Request 처럼 WAS별로 구현체 클래스가 다르다. 
- 어떤 하나의 WAS에 종속적이지 않고 표준화된 방식으로 서블릿 기반 애플리케이션을 개발할 수 있다.
- Tomcat, Jetty 등의 WAS가 필요한 이유는 실제 구현체가 WAS에 정의 되어 있고, 애플리케이션 단에서는 인터페이스로만 정의 되어 있기 떄문이다.

