# Java List의 두가지 remove 메서드 : 값과 인덱스 어떤 게 호출될까?

## List 계층 구조

![List 계층 구조](https://github.com/user-attachments/assets/67def009-a474-44bd-8be8-53621ba643eb")

`Collection<E>` 인터페이스에는 `boolean remove(Object o)` 메서드가 정의되어 있다.
`List<E>` 인터페이스에는 `E remove(int index)` 메서드가 정의되어 있다.

여기서 의문이 들었던 것은 아래와 같은 상황에서 어떤 메서드가 호출되는가 하는 것이었다.

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
numbers.remove(1);  // 이건 인덱스 1을 삭제하는 걸까? 값 1을 삭제하는 걸까?
System.out.println(numbers); // 출력: [1, 3, 4, 5]
```

정답은 인덱스 1번 요소가 삭제된다. 즉, 2가 삭제된다.  
왜냐하면 **자바의 오버로딩 선택**은 **정확히 일치하는 타입의 메서드를 우선**으로 하기 때문이다.  
그래서 위 예시에서 1은 `int` 타입이기 때문에 `remove(int index)` 메서드가 호출된다.  

## 만약 값을 지우고 싶다면 어떻게 할까?

```java
numbers.remove(Integer.valueOf(1)); // 또는
numbers.remove((Integer) 1);
```
이렇게 타입을 명시해줘야 `remove(Object o)` 메서드가 호출된다.
