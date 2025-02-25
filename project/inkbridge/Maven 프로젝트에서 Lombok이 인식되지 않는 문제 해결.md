# Maven 프로젝트에서 Lombok이 인식되지 않는 문제 해결

## 문제 상황

과거에 진행했던 **Maven 기반의 팀 프로젝트**를 다시 리팩토링하기 위해 클론을 받아왔다.  
하지만 프로젝트를 컴파일하는 과정에서 **Lombok을 인식하지 못하는 문제**가 발생했다.

### 문제 분석
이 프로젝트는 **Maven 기반**으로 작성되었으며, 당시 정상적으로 작동하던 프로젝트였다.
하지만 현재 환경에서 빌드하자 **Lombok이 정상적으로 적용되지 않는 문제**가 발생했다.
이 프로젝트 이후에 나는 **Gradle**만 사용했는데 **Gradle**에서는 `annotationProcessor` 설정을 추가하여 Lombok을 사용한다.  
그래서 Maven에서도 유사한 설정이 필요할 것으로 생각했다.

## 해결 방법

### `pom.xml`에 `maven-compiler-plugin`에 대한 Lombok 설정 추가

Maven의 `maven-compiler-plugin`을 사용하여 Lombok을 올바르게 인식하도록 설정해야 한다.

```xml
<build>
    <plugins>
        ...
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                   <path>
                    <groupId>org.projectlombok</groupId>
                    <artifactId>lombok</artifactId>
                    <version>1.18.30</version>
                   </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
        ...
    </plugins>
</build>
```
## 원인 분석

### JDK 버전에 따른 차이
- 이전에는 JDK 11 환경에서 실행했지만, 현재 클론 후 IntelliJ에서 실행되면서 자동으로 JDK 23이 적용되었다.
- JDK 23부터는 애너테이션 프로세싱 방식이 변경되어, 명시적으로 활성화하지 않으면 Lombok이 정상적으로 동작하지 않는다.

> 이 프로젝트는 11버전이라 버전만 바꿔주면 해결됐지만, 원인이 궁금하여 알아보았다.

### Gradle과 Maven 차이
- Gradle 5 이하에서는 annotationProcessor 없이도 Lombok이 자동으로 활성화되었다.
- 하지만 Gradle 6 이상부터는 명시적으로 annotationProcessor를 추가해야 한다.
- 요즘에는 대부분 Gradle 8을 사용하므로, Lombok을 적용하려면 annotationProcessor 설정이 필수적이다.
- Maven도 최신 JDK 환경에서는 애너테이션 프로세싱을 수동으로 설정해야 한다.

## 결론
Lombok과 같은 애너테이션 기반 코드 생성 라이브러리는 JDK와 Gradle/Maven 버전에 따라 annotationProcessor 활성화 방식이 다르므로 환경에 상관없이 명시적으로 설정하는 것이 가장 안전하다고 생각한다.

## 참고
[Using Lombok Library With JDK 23](https://dzone.com/articles/using-lombok-library-witk-jdk-23)