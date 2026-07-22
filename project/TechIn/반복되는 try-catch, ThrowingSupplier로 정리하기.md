# 반복되는 try-catch, ThrowingSupplier로 정리하기

외부 API를 연동하다 보면 여러 메서드에서 비슷한 형태의 `try-catch`가 반복되는 경우가 있다.

이번에는 YouTube Data API를 호출하는 과정에서 `IOException`을 처리하는 코드가 여러 메서드에 반복됐다.

```java
try {
    return youtubeApiRequest.execute();
} catch (IOException e) {
    throw new YoutubeApiException("유튜브 API 요청에 실패했습니다.", e);
}
```

처음 한두 개의 메서드에서는 크게 문제라고 느끼지 않았다.

하지만 API 호출 메서드가 늘어나면서 각 메서드의 핵심 로직보다 반복되는 예외 처리 코드가 더 눈에 들어오기 시작했다.

이번 글에서는 반복되는 `try-catch`를 공통 메서드로 분리하고, checked exception을 처리할 수 있는 `ThrowingSupplier`를 활용해 코드를 정리한 과정을 쓰려고 한다.

---

## 문제 상황

YouTube 채널 정보를 조회하는 메서드는 다음과 같은 형태였다.

```java
public ChannelListResponse fetchYoutubeChannel(String forHandle) {
    try {
        return youtubeClient.channels()
            .list(List.of("snippet", "contentDetails"))
            .setForHandle(forHandle)
            .execute();
    } catch (IOException e) {
        throw new YoutubeApiException(
            "유튜브 채널 조회에 실패했습니다. handle=" + forHandle,
            e
        );
    }
}
```

채널의 업로드 재생목록에 포함된 영상을 조회하는 메서드도 비슷했다.

```java
public PlaylistItemListResponse fetchYoutubeChannelVideos(
    String playlistId,
    String pageToken,
    long maxResults
) {
    try {
        return youtubeClient.playlistItems()
            .list(List.of("snippet", "status"))
            .setPlaylistId(playlistId)
            .setMaxResults(maxResults)
            .setPageToken(pageToken)
            .execute();
    } catch (IOException e) {
        throw new YoutubeApiException(
            "유튜브 재생목록 영상 조회에 실패했습니다. playlistId=" + playlistId,
            e
        );
    }
}
```

영상의 상세 정보를 조회할 때도 같은 형태가 반복됐다.

```java
public VideoListResponse fetchVideo(String videoId) {
    try {
        return youtubeClient.videos()
            .list(List.of("contentDetails"))
            .setId(List.of(videoId))
            .execute();
    } catch (IOException e) {
        throw new YoutubeApiException(
            "유튜브 영상 조회에 실패했습니다. videoId=" + videoId,
            e
        );
    }
}
```

각 메서드에서 호출하는 YouTube API는 서로 다르다.

하지만 예외 처리 구조는 모두 동일하다.

```text
try {
    YouTube API 요청 실행
} catch (IOException e) {
    YoutubeApiException으로 변환
}
```

실제로 메서드마다 달라지는 것은 다음 두 가지뿐이었다.

```text
실행할 YouTube API 요청
실패했을 때 사용할 예외 메시지
```

---

## 반복되는 try-catch의 문제

`try-catch` 자체가 나쁜 것은 아니다.

예외가 발생했을 때 재시도하거나, 대체 값을 반환하거나, 상태를 복구해야 한다면 각 메서드에서 직접 예외를 처리하는 것이 더 적절할 수 있다.

문제는 catch 블록에서 수행하는 작업이 단순한 예외 변환뿐인데도 같은 코드가 여러 곳에 반복된다는 점이었다.

### 핵심 로직이 잘 보이지 않는다

다음 코드에서 실제 핵심은 YouTube API 요청을 구성하고 실행하는 부분이다.

```java
return youtubeClient.videos()
    .list(List.of("contentDetails"))
    .setId(List.of(videoId))
    .execute();
```

하지만 전체 메서드를 보면 `try-catch`와 예외 생성 코드가 함께 들어가 있어 핵심 흐름을 한눈에 파악하기 어렵다.

### 예외 처리 정책이 분산된다

현재는 모든 `IOException`을 `YoutubeApiException`으로 변환하고 있다.

```java
throw new YoutubeApiException(message, e);
```

하지만 이 코드가 여러 메서드에 흩어져 있으면 이후 일부 메서드에서 원인 예외를 누락하거나, 메시지 형식이 달라지는 문제가 발생할 수 있다.

```java
throw new YoutubeApiException("유튜브 조회 실패");
```

```java
throw new RuntimeException(e);
```

```java
throw new YoutubeApiException("API 오류", e);
```

같은 종류의 API 요청인데도 예외 처리 방식이 달라질 수 있는 것이다.

### 수정 비용이 증가한다

나중에 공통 로깅이나 모니터링 코드를 추가하려면 반복된 모든 catch 블록을 찾아 수정해야 한다.

예외 처리 정책을 한곳에 모으면 변경이 필요할 때 공통 메서드만 수정할 수 있다.

---

## 실행할 작업을 메서드로 전달하기

반복되는 부분을 공통 메서드로 분리하려면 실행할 API 요청 자체를 인자로 전달해야 한다.

원하는 형태는 다음과 같다.

```java
private <T> T execute(
    실행할 작업,
    String errorMessage
) {
    // 작업 실행 및 예외 처리
}
```

YouTube API마다 반환 타입은 서로 다르다.

```java
ChannelListResponse
PlaylistItemListResponse
VideoListResponse
```

따라서 공통 메서드는 제네릭 타입 `T`를 반환하도록 정의할 수 있다.

```java
private <T> T execute(
    ...
) {
    ...
}
```

이제 첫 번째 인자로 실행할 작업을 람다 형태로 전달해야 한다.

---

## Supplier를 사용하면 되지 않을까?

인자를 받지 않고 값을 반환하는 대표적인 함수형 인터페이스로 `Supplier<T>`가 있다.

```java
@FunctionalInterface
public interface Supplier<T> {

    T get();
}
```

이를 사용하면 다음과 같은 공통 메서드를 만들 수 있을 것처럼 보인다.

```java
private <T> T execute(
    Supplier<T> supplier,
    String errorMessage
) {
    return supplier.get();
}
```

호출부에서는 YouTube API 요청을 람다로 전달할 수 있다.

```java
return execute(
    () -> youtubeClient.videos()
        .list(List.of("contentDetails"))
        .setId(List.of(videoId))
        .execute(),
    "유튜브 영상 조회에 실패했습니다. videoId=" + videoId
);
```

하지만 이 코드는 컴파일되지 않는다.

YouTube API의 `execute()` 메서드는 checked exception인 `IOException`을 던질 수 있기 때문이다.

반면 `Supplier<T>`의 `get()`에는 `throws` 선언이 없다.

```java
T get();
```

람다 표현식은 자신이 구현하는 함수형 인터페이스의 메서드가 허용하지 않는 checked exception을 던질 수 없다.

결국 일반적인 `Supplier<T>`에는 `IOException`을 던지는 YouTube API 요청을 그대로 전달할 수 없다.

---

## IOException을 던질 수 있는 IoSupplier 만들기

처음에는 `IOException`을 던질 수 있는 함수형 인터페이스를 직접 만들었다.

```java
@FunctionalInterface
public interface IoSupplier<T> {

    T get() throws IOException;
}
```

`Supplier<T>`와 구조는 같지만, `get()`에 `throws IOException`을 선언했다.

```java
T get() throws IOException;
```

이제 공통 메서드에서 작업을 실행하고 `IOException`을 처리할 수 있다.

```java
private <T> T execute(
    IoSupplier<T> supplier,
    String errorMessage
) {
    try {
        return supplier.get();
    } catch (IOException e) {
        throw new YoutubeApiException(errorMessage, e);
    }
}
```

사용부는 다음과 같이 정리됐다.

```java
public ChannelListResponse fetchYoutubeChannel(String forHandle) {
    return execute(
        () -> youtubeClient.channels()
            .list(List.of("snippet", "contentDetails"))
            .setForHandle(forHandle)
            .execute(),
        "유튜브 채널 조회에 실패했습니다. handle=" + forHandle
    );
}
```

```java
public PlaylistItemListResponse fetchYoutubeChannelVideos(
    String playlistId,
    String pageToken,
    long maxResults
) {
    return execute(
        () -> youtubeClient.playlistItems()
            .list(List.of("snippet", "status"))
            .setPlaylistId(playlistId)
            .setMaxResults(maxResults)
            .setPageToken(pageToken)
            .execute(),
        "유튜브 재생목록 영상 조회에 실패했습니다. playlistId=" + playlistId
    );
}
```

```java
public VideoListResponse fetchVideo(String videoId) {
    return execute(
        () -> youtubeClient.videos()
            .list(List.of("contentDetails"))
            .setId(List.of(videoId))
            .execute(),
        "유튜브 영상 조회에 실패했습니다. videoId=" + videoId
    );
}
```

각 메서드에서 반복되던 `try-catch`가 사라지고, 실제로 실행할 YouTube API 요청과 실패 메시지만 남았다.

---

## 직접 만든 IoSupplier가 꼭 필요할까?

`IoSupplier<T>`를 사용해 반복되는 예외 처리 코드는 제거할 수 있었다.

하지만 다시 살펴보면 `IoSupplier`의 역할은 매우 일반적이다.

```java
@FunctionalInterface
public interface IoSupplier<T> {

    T get() throws IOException;
}
```

결국 다음 역할을 수행하는 함수형 인터페이스다.

> 값을 반환하면서 checked exception을 던질 수 있는 Supplier

이러한 목적의 인터페이스가 이미 라이브러리에 존재한다면 직접 정의할 필요가 없다.

프로젝트에서 사용할 수 있는 함수형 인터페이스를 확인한 결과, checked exception을 던질 수 있는 `ThrowingSupplier`를 사용할 수 있었다.

따라서 직접 만든 `IoSupplier`를 제거하고 `ThrowingSupplier`로 변경했다.

---

## ThrowingSupplier 적용하기

`ThrowingSupplier`는 일반적인 `Supplier`와 비슷하지만 checked exception이 발생할 수 있는 작업을 표현할 수 있다.

이를 활용하면 공통 메서드는 다음과 같은 형태가 된다.

```java
private <T> T execute(
    ThrowingSupplier<T> supplier,
    String errorMessage
) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw new YoutubeApiException(errorMessage, e);
    }
}
```

사용부는 기존 `IoSupplier`를 사용했을 때와 거의 동일하다.

```java
public ChannelListResponse fetchYoutubeChannel(String forHandle) {
    return execute(
        () -> youtubeClient.channels()
            .list(List.of("snippet", "contentDetails"))
            .setForHandle(forHandle)
            .execute(),
        "유튜브 채널 조회에 실패했습니다. handle=" + forHandle
    );
}
```

```java
public PlaylistItemListResponse fetchYoutubeChannelVideos(
    String playlistId,
    String pageToken,
    long maxResults
) {
    return execute(
        () -> youtubeClient.playlistItems()
            .list(List.of("snippet", "status"))
            .setPlaylistId(playlistId)
            .setMaxResults(maxResults)
            .setPageToken(pageToken)
            .execute(),
        "유튜브 재생목록 영상 조회에 실패했습니다. playlistId=" + playlistId
    );
}
```

```java
public VideoListResponse fetchVideo(String videoId) {
    return execute(
        () -> youtubeClient.videos()
            .list(List.of("contentDetails"))
            .setId(List.of(videoId))
            .execute(),
        "유튜브 영상 조회에 실패했습니다. videoId=" + videoId
    );
}
```

직접 정의했던 `IoSupplier`는 제거됐고, 각 조회 메서드에는 실제 API 요청과 실패 메시지만 남았다.

---

## 변경 전후 비교

### 변경 전

```java
public VideoListResponse fetchVideo(String videoId) {
    try {
        return youtubeClient.videos()
            .list(List.of("contentDetails"))
            .setId(List.of(videoId))
            .execute();
    } catch (IOException e) {
        throw new YoutubeApiException(
            "유튜브 영상 조회에 실패했습니다. videoId=" + videoId,
            e
        );
    }
}
```

### 변경 후

```java
public VideoListResponse fetchVideo(String videoId) {
    return execute(
        () -> youtubeClient.videos()
            .list(List.of("contentDetails"))
            .setId(List.of(videoId))
            .execute(),
        "유튜브 영상 조회에 실패했습니다. videoId=" + videoId
    );
}
```

변경 전에는 하나의 메서드가 다음 책임을 모두 가지고 있었다.

```text
YouTube API 요청 구성
YouTube API 요청 실행
IOException 처리
YoutubeApiException 생성
```

리팩토링 후에는 책임이 분리됐다.

```text
각 조회 메서드
→ 어떤 YouTube API 요청을 실행할지 정의한다.

execute()
→ 요청을 실행하고 예외를 YoutubeApiException으로 변환한다.
```

단순히 코드의 줄 수만 줄어든 것이 아니라 각 메서드의 역할이 더 분명해졌다.

---

## 예외 변환 함수를 받지 않은 이유

공통 메서드를 더 범용적으로 만들려면 예외 변환 함수까지 인자로 받을 수 있다.

```java
private <T> T execute(
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

호출부에서는 작업별로 다른 예외를 생성할 수 있다.

```java
return execute(
    () -> someOperation(),
    e -> new SomeException("작업에 실패했습니다.", e)
);
```

하지만 현재 클래스에서는 모든 YouTube API 호출 실패를 `YoutubeApiException`으로 변환한다.

```java
throw new YoutubeApiException(errorMessage, e);
```

호출부마다 달라지는 것은 예외 타입이 아니라 실패 메시지뿐이다.

따라서 현재 요구사항에서는 예외 변환 함수까지 전달받는 것보다 문자열 메시지만 전달받는 방식이 더 단순하다.

```java
private <T> T execute(
    ThrowingSupplier<T> supplier,
    String errorMessage
)
```

추상화는 범용적일수록 항상 좋은 것이 아니다.

현재 필요한 범위까지만 공통화하고, 실제로 서로 다른 예외 타입이 필요해질 때 확장하는 편이 코드의 의도를 더 명확하게 유지할 수 있다.

---

## 공통 메서드 이름 정하기

공통 메서드의 이름을 단순히 `execute()`로 작성할 수도 있다.

```java
private <T> T execute(
    ThrowingSupplier<T> supplier,
    String errorMessage
)
```

다만 YouTube API 요청 객체 자체에서도 마지막에 `execute()`를 호출하고 있기 때문에 코드에서 두 `execute()`가 함께 보일 수 있다.

```java
return execute(
    () -> youtubeClient.videos()
        .list(List.of("contentDetails"))
        .setId(List.of(videoId))
        .execute(),
    errorMessage
);
```

이름이 겹치는 것이 혼란스럽다면 공통 메서드의 역할을 더 구체적으로 표현할 수 있다.

```java
private <T> T executeYoutubeRequest(
    ThrowingSupplier<T> supplier,
    String errorMessage
) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw new YoutubeApiException(errorMessage, e);
    }
}
```

사용부도 의미가 더 분명해진다.

```java
public VideoListResponse fetchVideo(String videoId) {
    return executeYoutubeRequest(
        () -> youtubeClient.videos()
            .list(List.of("contentDetails"))
            .setId(List.of(videoId))
            .execute(),
        "유튜브 영상 조회에 실패했습니다. videoId=" + videoId
    );
}
```

공통 메서드가 현재 클래스에서 어떤 역할을 하는지 고려해 적절한 이름을 선택하면 된다.

---

## 모든 try-catch를 공통화할 필요는 없다

이번 리팩토링에서 catch 블록은 다음 작업만 수행하고 있었다.

```text
IOException을 잡는다.
YoutubeApiException으로 변환한다.
작업별 오류 메시지를 추가한다.
```

별도의 복구 로직이나 예외별 분기가 없었기 때문에 공통화하기 적합했다.

반면 다음과 같은 경우에는 명시적인 `try-catch`가 더 적절할 수 있다.

* 예외 종류에 따라 서로 다른 처리가 필요한 경우
* 실패한 작업을 재시도해야 하는 경우
* 실패 시 대체 값을 반환해야 하는 경우
* 일부 상태나 자원을 복구해야 하는 경우
* 특정 예외는 그대로 전달해야 하는 경우
* 예외 처리 흐름을 코드에 명시적으로 보여주는 것이 더 중요한 경우

추상화의 목적은 모든 `try-catch`를 숨기는 것이 아니다.

동일한 예외 처리 정책이 반복될 때 이를 한곳에서 관리하도록 만드는 것이 목적이다.

---

## 주의할 점

### 원인 예외를 보존해야 한다

예외를 새로운 예외로 변환할 때는 기존 예외를 원인으로 전달하는 것이 좋다.

```java
throw new YoutubeApiException(errorMessage, e);
```

다음처럼 원인 예외를 제거하면 실제 장애 원인을 추적하기 어려워질 수 있다.

```java
throw new YoutubeApiException(errorMessage);
```

외부 API 응답 오류나 네트워크 오류의 스택 트레이스를 확인하려면 기존 예외를 반드시 보존해야 한다.

### RuntimeException까지 다시 감쌀지 결정해야 한다

`ThrowingSupplier`의 구현에 따라 공통 메서드에서 `Exception`을 잡으면 checked exception뿐 아니라 `RuntimeException`까지 함께 잡을 수 있다.

```java
catch (Exception e) {
    throw new YoutubeApiException(errorMessage, e);
}
```

작업 내부에서 이미 의미 있는 런타임 예외가 발생했는데 이를 다시 `YoutubeApiException`으로 감싸면 원래 예외의 의미가 흐려질 수 있다.

런타임 예외를 그대로 전달하고 checked exception만 변환하려면 다음과 같이 구분할 수 있다.

```java
private <T> T executeYoutubeRequest(
    ThrowingSupplier<T> supplier,
    String errorMessage
) {
    try {
        return supplier.get();
    } catch (RuntimeException e) {
        throw e;
    } catch (Exception e) {
        throw new YoutubeApiException(errorMessage, e);
    }
}
```

반대로 YouTube API 연동 계층에서 발생한 모든 예외를 `YoutubeApiException`으로 통일하려는 정책이라면 런타임 예외까지 변환할 수도 있다.

어떤 방식이 맞는지는 애플리케이션의 예외 처리 정책에 따라 결정해야 한다.

---

## 결론

이번 리팩토링에서는 YouTube API 호출 메서드마다 반복되던 `try-catch`를 공통 실행 메서드로 분리했다.

처음에는 `IOException`을 던질 수 있는 `IoSupplier<T>`를 직접 정의했다.

```java
@FunctionalInterface
public interface IoSupplier<T> {

    T get() throws IOException;
}
```

이후 같은 목적의 `ThrowingSupplier`를 사용할 수 있다는 것을 확인하고 직접 만든 함수형 인터페이스를 제거했다.

그 결과 각 메서드에는 실제로 실행할 API 요청과 실패 메시지만 남게 됐다.

```java
return executeYoutubeRequest(
    () -> youtubeClient.videos()
        .list(List.of("contentDetails"))
        .setId(List.of(videoId))
        .execute(),
    "유튜브 영상 조회에 실패했습니다. videoId=" + videoId
);
```

이번 변경에서 중요한 것은 단순히 코드의 줄 수를 줄였다는 점이 아니다.

각 조회 메서드는 YouTube API 요청을 구성하는 일에 집중하고, 공통 메서드는 요청 실행과 예외 변환을 담당하도록 책임을 나눴다.

```text
try-catch를 없앤 것이 아니라,
반복되던 YouTube API 예외 처리 정책을 하나의 메서드로 모은 것이다.
```
