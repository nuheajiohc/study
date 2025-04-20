# JPA save와 saveAll 비교(feat. @Transactional의 영향)

`save()`와 `saveAll()`이 같은 저장 기능을 수행하지만 성능에는 차이가 있고 `@Transactional` 사용 여부에 따라서도 다른 결과를 나타낸다.

이번 글에서는 이 둘의 내부 구조와 함께, @Transactional 유무에 따른 성능 차이를 실험과 함께 분석해보겠다.

## save()와 saveAll() 내부 구조

`JpaRepository`의 기본 구현체인 `SimpleJpaRepository`의 내부를 보면 다음과 같다.

**save()**

```java

@Transactional
@Override
public <S extends T> S save(S entity) {

	Assert.notNull(entity, "Entity must not be null.");

	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
}
```

**saveAll()**

```java

@Transactional
@Override
public <S extends T> List<S> saveAll(Iterable<S> entities) {

	Assert.notNull(entities, "Entities must not be null!");

	List<S> result = new ArrayList<>();

	for (S entity : entities) {
		result.add(save(entity));
	}

	return result;
}
```

`saveAll()`은 단순히 `save()`를 반복 호출하며 결과를 리스트로 반환하는 구조다.  
로직이 같아 보이지만 실제 성능도 같을까?


## 실험 환경

- H2 DB (in-memory)
- Book엔티티: id, name 필드만 존재
- 엔티티 100,000개 저장하기

```java

@Entity
public class Book {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	private String name;

	public Book(String name) {
		this.name = name;
	}
}

public interface BookRepository extends JpaRepository<Book, Long> {
}
```

## 실험 내용
`@Transactional` 유무에 따라 `save()`와 `saveAll()`성능을 각각 비교한다.

### @Transcatinal + save() 10만 번 호출

```java

@Test
@Transactional
void testSaveIndividuallyWithTransactional() {
	List<Book> list = new ArrayList<>();
	for (int i = 0; i < 100_000; i++) {
		list.add(new Book("book " + i));
	}

	long start = System.currentTimeMillis();
	for (Book book : list) {
		bookRepository.save(book);
	}
	long end = System.currentTimeMillis();

	System.out.println("save(): " + (end - start) + "ms");
}
```

### 트랜잭션 없이 save() 10만 번 호출

```java

@Test
void testSaveIndividually() {
	List<Book> list = new ArrayList<>();
	for (int i = 0; i < 100_000; i++) {
		list.add(new Book("book " + i));
	}

	long start = System.currentTimeMillis();
	for (Book book : list) {
		bookRepository.save(book);
	}
	long end = System.currentTimeMillis();

	System.out.println("save(): " + (end - start) + "ms");
}
```

### @Transactional + saveAll() 한 번 호출

```java

@Test
@Transactional
void testSaveAllWithTransactional() {
	List<Book> list = new ArrayList<>();
	for (int i = 0; i < 100_000; i++) {
		list.add(new Book("book " + i));
	}

	long start = System.currentTimeMillis();
	bookRepository.saveAll(list);
	long end = System.currentTimeMillis();

	System.out.println("saveAll(): " + (end - start) + "ms");
}

```

### 트랜잭션 없이 saveAll() 한 번 호출

```java

@Test
void testSaveAll() {
	List<Book> list = new ArrayList<>();
	for (int i = 0; i < 100_000; i++) {
		list.add(new Book("book " + i));
	}

	long start = System.currentTimeMillis();
	bookRepository.saveAll(list);
	long end = System.currentTimeMillis();

	System.out.println("saveAll(): " + (end - start) + "ms");
}
```

## 실험 결과

각 메서드로 약 10번 정도 실행한 결과 아래와 같은 결과가 나왔다.

트랜잭션 o save() : 1197 1174 1210 1272 1227 1215 917 917 1231 1218 1185 1235  
트랜잭션 x save() : 1749 1465 1435 1858 1748 1834 1838  
트랜잭션 o saveAll() : 926 988 1024 1029 994 1015 1046 989 1091 951 733 1059 695 1008 955 711 707 989  
트랜잭션 x saveAll() : 1114 1158 1168 1095 778 816 881 787 1105 1148 692 826 1112

> 각 실험마다 한번씩 결과가 튀는 것이 나오는데 아마도 이것은 내부적으로 캐싱이 발생해서 그런것으로 예상된다.  
> 그래서 그런 경우를 배제하고 표로 나타내면 아래와 같은 결과라고 생각할 수 있다.

| 방식        | 트랜잭션 | 평균 실행 시간 (ms) |
|-----------|------|---------------|
| save()    | o    | 1150 ~ 1250   |
| save()    | x    | 1700 ~ 1850   |
| saveAll() | o    | 900  ~ 1050   |
| saveAll() | x    | 1100 ~ 1150   |

실험 결과만 보자면 트랜잭션을 가진 saveAll() 이 가장 빠르고 트랜잭션이 없는 saveAll(), 트랜잭션이 있는 save()는 비슷하고, 가장 느린 것은 트랜잭션이 없는 save()로 볼 수 있을 것
같다.  
하지만 실제로는 트랜잭션 유무와 상관없이 saveAll()이 똑같이 제일 빠르고, 그 다음이 트랜잭션이 있는 save(), 가장 느린 건 트랜잭션이 없는 save()이다.  
조금의 차이가 있는 이유는 아래에서 분석을 진행하며 설명해보겠다.

## 분석

### 트랜잭션의 기본 원리

`@Transactional`은 AOP 기반으로 **CGLIB 프록시 객체**를 만들어 트랜잭션 시작, 커밋, 롤백 로직을 앞뒤로 감싼다.
전파 방식의 기본값은 `REQUIRED`로 기존 트랜잭션이 있으면 참여하고 없으면 새로 만든다.

하지만 여기서 짚고 넘어가야 할 점이 있다.   
예를 들어 클래스 안 두 메서드가 존재하고 하나를 호출해서 사용한다고 가정해보자.  
이 때 호출 되는 메서드에 `@Transactional`이 붙어있다고 해도 트랜잭션은 먹히지 않는다.  
코드로 확인해보자.

**원본 클래스**

```java

@Service
public class MyService {

	public void outer() {
		this.inner(); // 이거 프록시 안 거쳐서 트랜잭션 안 먹힘
	}

	@Transactional
	public void inner() {
		System.out.println("💥 트랜잭션 실행됨?");
	}
}
```

**CGLIB가 생성한 프록시 클래스**(Transactional 애노테이션이 존재하므로 이걸로 빈 등록됨)

완전 똑같지는 않을텐데 이런 느낌으로 구현되어 있을 것이다.

```java
public class MyService$$EnhancerBySpringCGLIB extends MyService {

	private final TransactionInterceptor txInterceptor;

	public MyService$$EnhancerBySpringCGLIB(TransactionInterceptor txInterceptor) {
		this.txInterceptor = txInterceptor;
	}

	@Override
	public void inner() {
		// 프록시에서 트랜잭션을 감쌈!
		Method method = MyService.class.getMethod("inner");
		if (txInterceptor != null && txInterceptor.applies(method)) {
			txInterceptor.invoke(() -> super.inner());
		} else {
			super.inner();
		}
	}

	// outer()는 오버라이딩 X → 그대로 상속됨
}
```

프록시 객체는 오버라이딩된 메서드만 감싸기 때문에 위 코드처럼 내부 메서드 호출 시 트랜잭션이 적용되지 않는다.

그렇기 때문에 `outer()`를 호출한 후 트랜잭션을 적용하기 위해 `inner()` 메서드를 호출해도 원본 클래스에 `inner()`메서드는 트랜잭션이 없기 때문에 트랜잭션이 적용되지 않는다.


### save/saveAll과 트랜잭션 전파

- `@Transactional`이 없는 `save` 실험을 한 테스트는 반복문을 돌 때마다 각자의 트랜잭션이 생긴다. 왜냐하면 호출한 메서드에 트랜잭션이 없기 때문에 트랜잭션 합류가 불가능 하기 때문이다.  
- `@Transactional`이 있는 `save` 실험을 한 테스트는 테스트 메서드의 트랜잭션에 `save` 메서드의 트랜잭션이 합류되기 떄문에 하나의 트랜잭션으로 로직을 처리한다.  
- 두 개의 `saveAll` 실험 메서드는 테스트 메서드의 트랜잭션 유무에 상관없이 `SimpleJpaRepository` 클래스의 `saveAll`메서드에 `@Transactional`이 존재하기 때문에 하나의 트랜잭션에 전부 합류하여 처리하게 된다.

### 그렇다면 하나의 트랜잭션으로 처리되는데 왜 조금씩 결과가 다르게 나왔을까?

먼저 트랜잭션이 없는 `save` 메서드가 가장 느린 건 자명한 사실이다.

트랜잭션이 있는 `save` 메서드는 트랜잭션 전파를 위한 오버헤드가 발생한다. 그래서 `saveAll`보다 느리다.

문제는 `saveAll`이다. 같은 결과가 나와야 할 것 같은데 조금의 시간차이가 발생한다.  
그 이유는 테스트 메서드에 트랜잭션을 걸면 커밋/롤백이 되기 전까지의 시간을 기록한다.  
하지만 트랜잭션을 걸지 않으면 `saveAll` 메서드에서 처음으로 트랜잭션이 발생하기 때문에 커밋/롤백이 끝난 후의 시간을 기록한다.  
그렇기 때문에 트랜잭션이 있을 때 시간이 좀 더 짧았던 것이다.

이것은 엄연히 말하면 테스트를 잘못한 거라서 실제로는 같은 결과가 나온다고 생각하면 된다.

또한 saveAll 메서드는 save와 saveAll 메서드가 하나의 클래스 내부에 있어서 프록시 객체를 호출 하는 것이 아니라 실제 save 메서드를 호출하기 때문에 트랜잭션 전파 오버헤드가 발생하지 않는다.  
트랜잭션이 적용이 안된다는 말이 아니라 적용이 되는데 일반 메서드처럼 호출이 된다는 말이다.


## 결론
- `save()`로 많은 양의 데이터를 삽입할 시 성능이 낮아지기 때문에 대량 삽입은 `saveAll()`을 사용하자.
- 트랜잭션이 있어도 `save()` 반복 호출은 프록시 객체 호출로 인해 `saveAll()`보다 느리다.
- `saveAll()`도 auto-increment인 데이터를 삽입할 때는 성능 이슈가 있으므로 jdbcTemplate 같은 배치 삽입을 지원하는 전략을 사용하자.


