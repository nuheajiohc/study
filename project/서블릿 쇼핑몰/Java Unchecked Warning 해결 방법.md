# Java Unchecked Warning 해결 방법

**ServletContext**의 **Attribute**에서 `List` 값을 가져올 때 **Unchecked Warning**이 발생했는데 이 경고를 해결하는 두 가지 방법을 소개해보겠다.


## Unchecked Warning 원인

### 제네릭과 타입 소거
**Unchecked Warning**은 주로 제네릭과 관련해서 발생한다. 제네릭은 **컴파일 타임**에만 타입을 유지하고 **런타임**에는 타입이 소거된다.  
이로 인해 컴파일 타임에는 타입 안전성을 보장하더라도 런타임에서는 이를 보장하지 못하게 된다.

### 주요 원인은 `Object`로 저장해서 제네릭 타입으로 다운 캐스팅을 해야할 때
   - 반환 타입이 `Object`이므로 특정 타입으로 캐스팅할 때 컴파일러는 타입 안전성을 검증할 수 없습니다.
   - 왜냐하면 런타임에는 제네릭이 소거되기 때문이다.

   ```java
   List<BaseController> controllers = (List<BaseController>) getServletContext().getAttribute("controllers");
   ```
   - 위 코드에서 컴파일러는 캐스팅의 안전성을 보장할 수 없으므로 **Unchecked Warning**을 발생시킨다.


## Unchecked Warning 해결 방법

Unchecked Warning을 해결하는 방법은 크게 두 가지로 나눌 수 있다.

### 1. `@SuppressWarnings("unchecked")` 사용
@SuppressWarnings("unchecked") 어노테이션을 사용하여 타입 안전성 경고를 제거할 수 있다.

#### 코드 예제
```java
@SuppressWarnings("unchecked")
List<BaseController> controllers = (List<BaseController>) getServletContext().getAttribute("controllers");
```

#### 장단점
- **장점**
   - 타입 안전성을 보장할 수 있다면 추가 코드 작성 없이 간단히 경고를 제거할 수 있다.
   - 테스트 코드를 작성하여 타입 안전성을 보장하면 더 좋다.

- **단점**
   - 런타임에 **`ClassCastException`**이 발생할 가능성을 배제할 수 없다.
   - 실수로 인해 타입 불일치가 발생할 위험이 존재한다.

### 2. 래퍼 클래스 활용

래퍼 클래스를 사용하여 타입 캐스팅 문제를 해결할 수 있다. 이 방식은 코드 레벨에서 타입 안전성을 보장한다.

#### 코드 예제
##### ControllerRegistry 클래스
```java
@Getter
@RequiredArgsConstructor
public class ControllerRegistry {

  public static final String CONTROLLER_REGISTRY_NAME = "controllers";
  private final List<BaseController> controllers;

}
```

##### DispatcherServlet 클래스 (주요코드)
```java
@WebServlet("/*")
public class DispatcherServlet extends HttpServlet {

  private HandlerMapping handlerMapping;

  @Override
  public void init() throws ServletException {
    initHandlerMapping();
  }

  private void initHandlerMapping() {
    ControllerRegistry registry = (ControllerRegistry) getServletContext().getAttribute(
        ControllerRegistry.CONTROLLER_REGISTRY_NAME);
    List<BaseController> controllers =
        (registry != null) ? registry.getControllers() : Collections.emptyList();
    this.handlerMapping = new RequestMappingHandlerMapping(controllers);
  }
}
```

#### 장단점
- **장점**
   - 타입 안전성을 코드 레벨에서 보장한다.
   - **Unchecked Warning** 없이 컴파일 타임에 문제를 예방할 수 있다.

- **단점**
   - 래퍼 클래스를 추가로 작성해야 하므로 코드의 복잡성이 증가할 수 있다.


## 정리

| 방법                     | 장점                        | 단점                      |
|--------------------------|---------------------------|-------------------------|
| `@SuppressWarnings` 사용 | 간단하고 추가 코드 작성이 필요 없음      | 런타임에 타입 불일치 발생 가능성 존재   |
| 래퍼 클래스 활용           | 타입 안전성을 코드 레벨에서 보장        | 추가적인 코드 작성 필요, 복잡성 증가   |

두 방식 중 정답은 없고, 상황에 따라 적합한 방법을 선택하는 것이 좋다.
- **단순한 경우**에는 `@SuppressWarnings("unchecked")`를 사용하고 테스트 코드로 타입 안전성을 강화할 수 있다.
- **복잡하거나 타입 안전성이 중요한 경우**에는 래퍼 클래스를 사용하여 타입 안전성을 코드레벨에서 해결한다.

> 나는 코드가 추가할 코드가 많다고 생각하지 않아서 래퍼 클래스로 문제를 해결하였다.
