# Bean 등록할 때 @Configuration을 사용해야 하는 이유

Spring 애플리케이션에서 객체를 관리하는 핵심은 **Spring 컨테이너**이다.  
**Spring 컨테이너**는 Bean을 생성하고 관리할 때 `@Configuration` 어노테이션을 함께 사용한다.  
`@Configuration` 어노테이션은 싱글톤을 보장하면서 Bean들 사이의 의존성 주입을 해준다.  
이 글에서는 `@Configuration`이 필요한 이유에 대해서 설명해보겠다.

## @Configuration이란?

`@Configuration`은 클래스가 **Spring 컨테이너에 Bean을 등록하기 위한 클래스**임을 나타낸다.

주요 기능
1. **Bean 등록 및 관리**: `@Bean` 메서드를 통해 Bean을 정의한다.
2. **Singleton 보장**: 컨테이너는 @Bean 메서드를 호출할 때 스코프가 싱글톤(Singleton)일 경우 항상 동일한 인스턴스를 반환한다.
3. **의존성 관리**: Bean 간의 의존성을 명확하게 정의하고 연결할 수 있다.

> @Configuration은 어떤 원리로 Bean들에 싱글톤 객체를 주입할 있는 걸까?
> 싱글톤을 보장할 있는 이유는 스프링 컨테이너가 @Configuration이 붙은 클래스를 CGLIB라는 프록시로 관리하기 때문이다.
> 예시와 함께 아래서 알아보자.

## `proxyBeanMethods` 속성: true vs false
`@Configuration`은 `proxyBeanMethods`라는 속성을 제공한다.  
이 속성은 `@Configuration` 클래스가 **프록시를 사용하여 Bean 메서드를 호출할지 여부**를 결정한다. 기본값은 `true`이다.

### `proxyBeanMethods = true`
- Spring 컨테이너는 `@Configuration` 클래스를 **프록시 객체**로 생성한다.
- Bean 메서드가 호출되면 컨테이너가 해당 메서드의 반환 값을 관리하고, **동일한 인스턴스(Singleton)**를 반환한다.
- 별다른 처리를 안해도 기본값으로 true가 설정되어 있다.

#### 실험

```java
@Configuration
public class BabyConfig {

  @Bean
  public BabyBean babyBean() {
    return new BabyBean();
  }
}

@Configuration
@Import(BabyConfig.class)
public class ParentConfig {

  private final BabyConfig babyConfig;

  public ParentConfig(BabyConfig babyConfig) {
    this.babyConfig = babyConfig;
  }

  @Bean
  public MotherBean motherBean() {
    return new MotherBean(babyConfig.babyBean());
  }

  @Bean
  public FatherBean fatherBean() {
    return new FatherBean(babyConfig.babyBean());
  }
}

public class BabyBean {

  public BabyBean() {
    System.out.println("애기 빈 : " + this);
  }
}

public class FatherBean {

  public FatherBean(BabyBean babyBean) {
    System.out.println("아빠 빈 : " + babyBean);
  }
}

public class MotherBean {

  public MotherBean(BabyBean babyBean) {
    System.out.println("엄마 빈 : " + babyBean);
  }
}

public class Main {
  public static void main(String[] args) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ParentConfig.class);
    System.out.println(context.getBean(BabyConfig.class));
  }
}
```

### 결과:
```
애기 빈 : hello.configurationEx.service.BabyBean@55fe41ea
엄마 빈 : hello.configurationEx.service.BabyBean@55fe41ea
아빠 빈 : hello.configurationEx.service.BabyBean@55fe41ea
hello.configurationEx.config.BabyConfig$$SpringCGLIB$$0@313b2ea6
```

우리가 기대하던 결과이다.  
그런데 잘 생각해보면 `babyBean()`이 매개변수에서 실행될 때 새로운 객체가 만들어져야 할 것 같은데 기존의 객체가 주입이 된다.  
위 결과를 보면 새롭게 생성이 안된 것을 알 수 있다.  
`BabyConfig.class` bean의 출력문을 보면 `$$SpringCGLIB`이 붙어 있는 것을 알 수 있다.  
이것이 싱글톤을 보장해주는 핵심이다.  
`proxyBeanMethods = false` 이렇게 실행을 하면 스프링은 설정파일로 인식하지못하고 일반 객체로 빈을 등록하게 된다.  
먼저 이 예시를 살펴보자.

### `proxyBeanMethods = false`
- Spring 컨테이너는 `@Configuration` 클래스를 프록시로 생성하지 않는다.
- Bean 메서드가 호출될 때마다 **새로운 인스턴스**를 생성한다.

#### 실험

```java
@Configuration(proxyBeanMethods = false)
public class BabyConfig {

  @Bean
  public BabyBean babyBean() {
    return new BabyBean();
  }
}

@Configuration
@Import(BabyConfig.class)
public class ParentConfig {

  private final BabyConfig babyConfig;

  public ParentConfig(BabyConfig babyConfig) {
    this.babyConfig = babyConfig;
  }

  @Bean
  public MotherBean motherBean() {
    return new MotherBean(babyConfig.babyBean());
  }

  @Bean
  public FatherBean fatherBean() {
    return new FatherBean(babyConfig.babyBean());
  }
}

public class BabyBean {

  public BabyBean() {
    System.out.println("애기 빈 : " + this);
  }
}

public class FatherBean {

  public FatherBean(BabyBean babyBean) {
    System.out.println("아빠 빈 : " + babyBean);
  }
}

public class MotherBean {

  public MotherBean(BabyBean babyBean) {
    System.out.println("엄마 빈 : " + babyBean);
  }
}

public class Main {
  public static void main(String[] args) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ParentConfig.class);
    System.out.println(context.getBean(BabyConfig.class));
  }
}
```

### 결과:
```
애기 빈 : hello.configurationEx.service.BabyBean@367ffa75
애기 빈 : hello.configurationEx.service.BabyBean@ba2f4ec
엄마 빈 : hello.configurationEx.service.BabyBean@ba2f4ec
애기 빈 : hello.configurationEx.service.BabyBean@1c1bbc4e
아빠 빈 : hello.configurationEx.service.BabyBean@1c1bbc4e
hello.configurationEx.config.BabyConfig@6a2b953e
```
이번에는 엄마 빈, 아빠 빈이 만들어질 때 새로운 애기 빈이 만들어진 것을 알 수 있다.  
또한 BabyConfig도 일반 클래스로 등록된 것을 알 수 있다.  
표로 비교해보자. 

## 비교: `proxyBeanMethods = true` vs `proxyBeanMethods = false`

| 특성                         | `proxyBeanMethods = true`                  | `proxyBeanMethods = false` |
|-----------------------------|-------------------------------------------|----------------------------|
| **Singleton 보장**            | O                                         | X                          |
| **프록시 사용**               | O                                         | X                          |
| **Bean 간 의존성 관리**         | 자동으로 관리됨                             | 자동으로 관리 안됨                 |
| **새로운 인스턴스 생성 여부**    | 동일한 Bean 반환                          | 매 호출 시 새 인스턴스 생성           |

## CGLIB 프록시로 싱글톤을 보장하는 원리
Spring 컨테이너는 `@Configuration`이 붙은 클래스를 CGLIB 프록시로 감싼다.
CGLIB 프록시는 기존 클래스를 상속받아 동적으로 프록시 클래스를 생성하는 기술이다.

### 싱글톤 보장의 동작 원리
- `@Configuration`클래스는 프록시 객체(예:`MyConfig$$SpringCGLIB$$0`)로 생성된다. 
- 이 프록시 객체는 이미 컨테이너에 등록된 Bean이 있다면 해당 Bean을 반환하고, 등록되지 않은 경우에만 새로운 Bean을 생성한다.
- 따라서 동일한 @Bean 메서드를 호출하더라도 항상 컨테이너에 저장된 **같은 인스턴스(Singleton)**를 반환한다.

## 결론
`@Configuration`은 Bean을 관리하며 의존성 주입시에 싱글톤을 유지하게 도와준다.
그리고 이 특성은 `proxyBeanMethods`를 기본값(true)으로 유지함으로써 보장된다.

## 참고
[@Configuration은 어떻게 빈을 등록하고, 싱글톤으로 관리할까?](https://tecoble.techcourse.co.kr/post/2023-05-22-configuration/)