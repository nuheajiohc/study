# Tomcat 버전별 Jakarta 설정 및 서블릿 디펜던시 설정

**Apache Tomcat**은 Java 웹 애플리케이션을 실행하는 데 가장 보편적으로 사용되는 서블릿 컨테이너 중 하나이다.  
Jakrta EE 버전에 따라서 지원하는 서블릿 디펜던시도 다르다.  
이 글에서는 톰캣 버전별 Jakarta dependency 설정에 대해서 정리해보겠다.

## Tomcat 버전별 Jakrta EE 스펙 사양
| Servlet Spec | JSP Spec | EL Spec | WebSocket Spec | Authentication Spec (JASPIC) | Apache Tomcat Version | Latest Released Version | Supported Java Versions |
|--------------|----------|---------|----------------|------------------------------|----------------------|-------------------------|-------------------------|
| 6.1          | 4.0      | 6.0     | 2.2            | 3.1                          | 11.0.x               | 11.0.1                  | 17 and later            |
| 6.0          | 3.1      | 5.0     | 2.1            | 3.0                          | 10.1.x               | 10.1.33                 | 11 and later            |
| 5.0          | 3.0      |4.0	    | 2.0            |	2.0	                        |10.0.x (superseded)	| 10.0.27 (superseded)	 | 8 and later              |
| 4.0          | 2.3      | 3.0     | 1.1            | 1.1                          | 9.0.x                | 9.0.97                  | 8 and later             |

- Tomcat 9.0.x: Java EE 8 사양을 지원하며, `javax.*` 네임스페이스를 그대로 사용한다. 기존 Java EE 기반의 서블릿, JSP, JSTL 등도 모두 지원한다.
- Tomcat 10.0.x: Jakarta EE 9를 지원하며, 네임스페이스가 `javax.*`에서 `jakarta.*`로 변경되었다. 이 전환은 Java EE에서 Jakarta EE로 넘어가는 과정의 시작을 의미한다.
- Tomcat 10.1.x: Jakarta EE 10을 완벽히 지원하며, 최신 Servlet 6.0 사양을 따릅니다. 네임스페이스는 `jakarta.*`로 완전히 변경되었으며, Jakarta 표준 사양과 호환된다.

## 톰캣 10.1.x 를 선택했을 때 의존성
```gradle
    compileOnly('jakarta.servlet:jakarta.servlet-api:6.0.0') //서블릿 API 정의
    implementation 'org.glassfish.web:jakarta.servlet.jsp.jstl:3.0.1' // JSTL 구현체
```
`implementation 'jakarta.servlet.jsp.jstl:jakarta.servlet.jsp.jstl-api:3.0.1'`은 `org.glassfish.web:jakarta.servlet.jsp.jstl`에 포함되어 있고 후자가 전자의 구현체이다.  
그래서 따로 추가 시킬 필요 없다.




## 추가 자료 링크
[Java EE 8 자세한 스펙 사양](https://www.oracle.com/java/technologies/java-ee-8.html)  
[Jakarta EE 9 자세한 스펙 사양](https://jakarta.ee/release/9/)  
[Jakarta EE 10 자세한 스펙 사양](https://tomcat.apache.org/tomcat-10.1-doc/index.html)


> 스프링 부트 3.4.0에서 지원하는 Tomcat 버전은 10.1.x 이다.  
[스프링부트 톰캣 사양](https://docs.spring.io/spring-boot/system-requirements.html)