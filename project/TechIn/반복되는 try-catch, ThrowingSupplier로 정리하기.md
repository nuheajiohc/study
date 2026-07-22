# 반복되는 try-catch, ThrowingSupplier로 정리하기

개발을 하다 보면 같은 형태의 `try-catch`가 여러 메서드에 반복해서 등장하는 경우가 있다.

처음에는 큰 문제가 아닌 것처럼 보인다.

```java
try {
    return someOperation();
} catch (Exception e) {
    throw new RuntimeException(e);
}
```

하지만 이런 코드가 여러 곳에 반복되기 시작하면 코드의 핵심 흐름이 잘 보이지 않는다.

실제로 중요한 것은 `someOperation()`인데, 메서드마다 `try-catch`가 감싸고 있다 보니 읽는 사람은 매번 비슷한 예외 처리 코드를 지나쳐야 한다.

이번 글에서는 반복되는 `try-catch`를 `ThrowingSupplier`로 정리한 과정을 정리해보려고 한다.


## 문제 상황

예외가 발생할 수 있는 작업을 처리할 때 보통 다음과 같은 코드를 작성한다.

```java
public String methodA() {
    try {
        return externalOperationA();
    } catch (Exception e) {
        throw new RuntimeException("A 작업에 실패했습니다.", e);
    }
}
```

다른 메서드도 비슷하다.

```java
public String methodB() {
    try {
        return externalOperationB();
    } catch (Exception e) {
        throw new RuntimeException("B 작업에 실패했습니다.", e);
    }
}
```

또 다른 메서드에서도 같은 구조가 반복된다.

```java
public String methodC() {
    try {
        return externalOperationC();
    } catch (Exception e) {
        throw new RuntimeException("C 작업에 실패했습니다.", e);
    }
}
```

각 메서드가 수행하는 작업은 다르지만, 전체 구조는 거의 같다.

```text
try {
    실제 작업
} catch (Exception e) {
    예외 변환
}
```


## 반복되는 try-catch의 문제

`try-catch` 자체가 나쁜 것은 아니다.

예외가 발생할 수 있는 지점에서 명시적으로 처리하는 것은 당연히 필요하다.
문제는 **같은 예외 처리 패턴이 여러 곳에 반복될 때**다.

반복되는 `try-catch`는 몇 가지 문제를 만든다.

첫째, 핵심 로직보다 예외 처리 코드가 더 많이 보인다.

```java
public String methodA() {
    try {
        return externalOperationA();
    } catch (Exception e) {
        throw new RuntimeException("A 작업에 실패했습니다.", e);
    }
}
```

이 메서드에서 핵심은 `externalOperationA()`다.
하지만 코드의 절반은 예외 처리다.

둘째, 예외 처리 정책이 분산된다.

```java
throw new RuntimeException("A 작업에 실패했습니다.", e);
throw new IllegalStateException("B 작업에 실패했습니다.", e);
throw new IllegalArgumentException("C 작업에 실패했습니다.", e);
```

처음에는 각각 필요한 방식으로 작성했다고 생각할 수 있다.
하지만 시간이 지나면 어느 메서드는 `RuntimeException`, 어느 메서드는 `IllegalStateException`, 어느 메서드는 메시지 없이 예외를 던지는 식으로 일관성이 깨질 수 있다.

셋째, 수정 비용이 커진다.

나중에 예외 메시지 형식을 바꾸거나, 공통 로깅을 추가하거나, 특정 예외 타입으로 통일하고 싶다면 반복된 `catch` 블록을 모두 찾아서 수정해야 한다.


## Supplier를 사용하면 되지 않을까?

반복되는 실행 구조를 공통화하기 위해 처음에는 `Supplier<T>`를 떠올릴 수 있다.

`Supplier<T>`는 값을 반환하는 함수형 인터페이스다.

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

이를 사용하면 다음처럼 실행 코드를 전달할 수 있다.

```java
private <T> T execute(Supplier<T> supplier) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

그리고 사용부는 이렇게 줄일 수 있을 것 같다.

```java
public String methodA() {
    return execute(() -> externalOperationA());
}
```

그런데 문제가 있다.

`Supplier<T>`의 `get()` 메서드는 checked exception을 던질 수 없다.

```java
T get();
```

즉, `externalOperationA()`가 checked exception을 던지는 메서드라면 다음 코드는 컴파일되지 않는다.

```java
return execute(() -> externalOperationA());
```

결국 람다 내부에서 다시 `try-catch`를 작성해야 한다.

```java
public String methodA() {
    return execute(() -> {
        try {
            return externalOperationA();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    });
}
```

이렇게 되면 반복을 제거하려던 목적이 사라진다.

---

## ThrowingSupplier 정의하기

그래서 checked exception을 던질 수 있는 Supplier가 필요했다.

```java
@FunctionalInterface
public interface ThrowingSupplier<T> {
    T get() throws Exception;
}
```

기존 `Supplier<T>`와 거의 같지만, 차이는 하나다.

```java
T get() throws Exception;
```

`throws Exception`이 있기 때문에 checked exception을 던지는 작업도 람다로 전달할 수 있다.

이제 공통 실행 메서드를 만들 수 있다.

```java
private <T> T runCatching(ThrowingSupplier<T> supplier) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

사용부는 이렇게 간단해진다.

```java
public String methodA() {
    return runCatching(() -> externalOperationA());
}
```

`externalOperationA()`가 checked exception을 던져도 `ThrowingSupplier`의 `get()`이 `throws Exception`을 선언하고 있기 때문에 람다로 전달할 수 있다.

---

## 예외 메시지를 다르게 주고 싶다면

단순히 모든 예외를 `RuntimeException`으로 감싸도 되지만, 실제 코드에서는 메서드마다 예외 메시지를 다르게 주고 싶을 수 있다.

예를 들어 다음처럼 작업마다 실패 메시지가 다를 수 있다.

```java
public String methodA() {
    try {
        return externalOperationA();
    } catch (Exception e) {
        throw new RuntimeException("A 작업에 실패했습니다.", e);
    }
}

public String methodB() {
    try {
        return externalOperationB();
    } catch (Exception e) {
        throw new RuntimeException("B 작업에 실패했습니다.", e);
    }
}
```

이 경우 `runCatching()`이 메시지를 함께 받도록 만들 수 있다.

```java
private <T> T runCatching(
    ThrowingSupplier<T> supplier,
    String errorMessage
) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw new RuntimeException(errorMessage, e);
    }
}
```

사용부는 다음처럼 바뀐다.

```java
public String methodA() {
    return runCatching(
        () -> externalOperationA(),
        "A 작업에 실패했습니다."
    );
}
```

```java
public String methodB() {
    return runCatching(
        () -> externalOperationB(),
        "B 작업에 실패했습니다."
    );
}
```

이제 반복되던 `try-catch`는 사라지고, 메서드는 “무슨 작업을 실행하는지”와 “실패했을 때 어떤 메시지를 줄지”만 표현한다.

---

## 예외 타입까지 다르게 주고 싶다면

실제로는 메시지만 다른 것이 아니라 예외 타입도 다를 수 있다.

어떤 경우는 내부 상태 오류로 보고 `IllegalStateException`을 던지고 싶을 수 있고, 어떤 경우는 잘못된 입력으로 보고 `IllegalArgumentException`을 던지고 싶을 수 있다.

그럴 때는 예외 변환 함수를 받도록 만들 수 있다.

```java
private <T> T runCatching(
    ThrowingSupplier<T> supplier,
    Function<Exception, RuntimeException> exceptionMapper
) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw exceptionMapper.apply(e);
    }
}
```

사용부는 이렇게 된다.

```java
public String methodA() {
    return runCatching(
        () -> externalOperationA(),
        e -> new IllegalStateException("A 작업에 실패했습니다.", e)
    );
}
```

```java
public String methodB() {
    return runCatching(
        () -> externalOperationB(),
        e -> new IllegalArgumentException("B 작업에 실패했습니다.", e)
    );
}
```


## 최종 코드

최종 형태는 다음과 같다.

```java
@FunctionalInterface
public interface ThrowingSupplier<T> {
    T get() throws Exception;
}
```

```java
private <T> T runCatching(
    ThrowingSupplier<T> supplier,
    Function<Exception, RuntimeException> exceptionMapper
) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw exceptionMapper.apply(e);
    }
}
```

사용 예시는 다음과 같다.

```java
public String methodA() {
    return runCatching(
        () -> externalOperationA(),
        e -> new IllegalStateException("A 작업에 실패했습니다.", e)
    );
}
```

```java
public String methodB() {
    return runCatching(
        () -> externalOperationB(),
        e -> new IllegalArgumentException("B 작업에 실패했습니다.", e)
    );
}
```

---

## 결론

이번 리팩토링의 목적은 단순히 `try-catch`를 없애는 것이 아니었다.

반복되는 예외 처리 패턴을 한 곳으로 모으고, 각 메서드가 자신의 핵심 작업에 더 집중하도록 만드는 것이 목적이었다.

`ThrowingSupplier`를 사용하면 checked exception을 던지는 작업도 람다로 전달할 수 있고, 공통 실행 메서드에서 예외 변환을 일관되게 처리할 수 있다.

물론 모든 `try-catch`를 이런 방식으로 바꿀 필요는 없다.
복구 로직이 있거나, 예외 처리 방식이 복잡하거나, 반복이 많지 않다면 명시적인 `try-catch`가 더 좋은 선택일 수 있다.

하지만 같은 패턴의 `try-catch`가 여러 곳에서 반복되고, catch 블록에서 하는 일이 단순히 예외를 변환하는 것이라면 `ThrowingSupplier`는 충분히 고려해볼 만한 작은 추상화다.

```text
반복되는 try-catch를 제거한 것이 아니라,
반복되는 예외 처리 패턴에 이름을 붙인 것이다.
```


