# Java for-each 문에서 ConcurrentModificationException이 발생하는 이유

**모던 자바 인 액션 8장: 8.2.1 removeIf 메서드** 파트를 읽다가
for-each 문을 사용한 예시에서 list의 요소를 제거할 때 `ConcurrentModificationException`이 발생할 수 있다는 것을 알게 되었다.  

책에서 에러가 발생하는 이유는 반복자의 상태와 컬렉션의 상태가 동기화되지 않아서라고 설명하며 해결책에 대해서 설명해준다.  

나는 동기화되지 않는 이유가 궁금하여 이 부분에 대해서 좀 더 찾아보았다.

## ConcurrentModificationException이 발생하는 이유

**에러가 발생하는 코드**  
```java
for (Transaction transaction : transactions) { // 예외가 발생하는 지점
    if(Character.isDigit(transaction.getReferenceCode().charAt(0))) {
        transactions.remove(transaction);
    }
}
```
이 코드는 for(...)라인에서 `ConcurrentModificationException`이 발생한다.  

### 언제 `ConcurrentModificationException`이 발생할까?

ConcurrentModificationException 클래스 주석을 보면 다음과 같은 상황에서 예외를 발생시킨다고 나와있다.

1. 컬렉션이 순회 중에 변경되었을 때
   - ex. 한 스레드가 컬렉션을 **순회(iterate)**하고 있을 때, 다른 스레드가 요소를 추가/삭제하면 예외 발생 가능
2. 같은 스레드라도 규칙을 어기면 예외 발생 
   - ex. 한 스레드가 순회 도중 `Iterator.remove()`가 아닌 `list.remove()`를 사용하는 경우
3. 동기화 되지 않은 수정이 발생했을 때 예외가 나지 않을 수 있더라도 안전한 상황은 아니기 때문에 **fail-fast**로 예외 발생

> 이건 내가 임의로 요약한 내용이고, 자세한 내용은 해당 클래스 주석을 보면 확인할 수 있다.

for-each 문에서 예외가 발생하는 이유는 두 번째, 세 번째 경우에 해당된다.  
for-each 문은 내부적으로 `Iterator`를 사용하여 컬렉션을 순회한다.  
즉, 위 에러가 발생하는 코드가 실제로 해석된다면 다음과 같은 코드로 변경된다.

**변환된 에러가 발생하는 코드**
```java
Iterator<Transaction> iterator = transactions.iterator();
while (iterator.hasNext()) { 
    Transaction transaction = iterator.next(); // 이 부분에서 예외 발생
    if (Character.isDigit(transaction.getReferenceCode().charAt(0))) {
        transactions.remove(transaction); 
    }
}
```
iterator는 원본 컬렉션을 참조해서 순회를 하는데, 중간에 원본 컬렉션의 수정이 발생해서(여기서는 remove 메서드) 안전한 상황으로 인식되지 않아서 예외가 발생하는 것이다.  

실제로 어떻게 구현되어 있는지 내부코드를 확인해보자.

**ArrayList - remove 코드**

<p align="center">
<img width="648" alt="arraylist-remove" src="https://github.com/user-attachments/assets/658c8b37-3529-4e0f-a7d1-1420dc28cfdd" />
</p>

ArrayList는 내부적으로 배열로 구현되어 있어서, 배열과 크기를 복사한다.  
remove의 대상이 null이면 for문을 돌면서 null인 요소를 찾았거나 null이 아니라면 객체의 동등성을 확인하면 fastRemove를 호출한다. 그 외에는 false를 반환한다.  

remove에서 중요한 메서드는 fastRemove 메서드이다.  
먼저 간단하게 설명하면 System.arraycopy를 사용하여 요소들을 한칸씩 앞당기고, 마지막 요소는 null로 지운다.  

하지만 여기서 중요한 점은 modCount를 증가시키는 부분이다.  
**modCount의 원문 주석을 해석해보면 구조적인 변경(요소 추가/삭제 등)이 발생했을 때 증가돼서 변경의 횟수를 나타낸다.**  


그러면 이번에는 ArrayList에 정의되어있는 iterator를 살펴보자.  
**ArrayList - iterator 코드**  

<p align="center">
<img width="609" alt="arraylist-iter-next" src="https://github.com/user-attachments/assets/921454a0-272a-4fe8-8edc-998c00327db2" />
</p>

iterator는 내부적으로 modCount를 복사해서 expectedModCount에 담아 가지고 있다.  
next() 메서드를 호출할 때 내부적으로 `checkForComodification` 메서드를 호출한다.
여기서 modCount와 expectedModCount를 비교하는데 두 값이 다르면 `ConcurrentModificationException`을 발생시킨다.  
값이 달라지는 경우는 다른 스레드가 컬렉션을 수정했거나, for-each 구문에서 `Iterator.remove()`가 아닌 `list.remove()`를 사용했을 때 발생한다.  
ArrayList의 remve 메서드에서 modCount를 증가시키기 때문에, for-each 구문에서 `list.remove()`를 사용하면 expectedModCount와 modCount가 다르게 되어 예외가 발생하는 것이다.


**그렇다면 어떻게 사용해야 하는 걸까?**

```java
Iterator<Transaction> iterator = transactions.iterator()
while (iterator.hasNext()) {
    Transaction transaction = iterator.next();
    if (Character.isDigit(transaction.getReferenceCode().charAt(0))) {
        iterator.remove(); // Iterator의 remove 메서드를 사용
    }
}
```

이 코드가 잘 동작하는 이유는 아래의 코드를 보면 알 수 있다.

**ArrayList - iterator remove 코드**  

<p align="center">
<img width="494" alt="arraylist-iter-remove" src="https://github.com/user-attachments/assets/eb5086cd-1c5d-442c-a3ae-76b81dae9120" />
</p>

다른 메서드처럼 `checkForComodification` 메서드를 호출한 뒤 try블록에서 ArrayList의 remove를 호출하는 것을 볼 수 있다.(이 remove는 index를 받아서 요소를 삭제하는 메서드이다. 하지만 내부적으로 fastRemove를 호출한다는 게 핵심)
하지만 여기서 끝나는 것이 아니라 커서의 위치를 바꿔주고 expectedModCount에 modCount를 할당해준다. 그래서 iterator의 다른 메서드에서 checkForComodification 메서드를 호출하더라도 예외가 발생하지 않는다.


이렇게 ConcurrentModificationException이 발생하는 이유와 해결 방법을 알아보았다.
그런데 해결방법이 너무 복잡하다. 모던 자바 인 액션에서는 선언적으로 이 문제를 해결하는 방법을 알려준다. 

```java
transactions.removeIf(transaction -> Character.isDigit(transaction.getReferenceCode().charAt(0)));
```
엄청 간단해졌다. 
이 코드도 내부적으로는 위의 복잡한 코드처럼 생겼다. 여기서는 Collection 인터페이스에 default 메서드로 정의된 removeif 메서드를 사용한다.

**Collection - removeIf 코드**

<p align="center">
<img width="552" alt="스크린샷 2025-07-29 오전 9 29 23" src="https://github.com/user-attachments/assets/e68d0b1c-daf5-429b-92e1-ecf14357165d" />
</p>

이렇게 for-each 문에서 ConcurrentModificationException이 발생하는 이유와 해결 방법을 알아보았다.    
앞으로 순회하면서 요소를 삭제할 때는 removeIf 메서드를 활용하도록 하자.
