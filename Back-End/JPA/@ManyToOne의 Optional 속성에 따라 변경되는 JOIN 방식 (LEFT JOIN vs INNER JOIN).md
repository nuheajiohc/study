# @ManyToOne의 Optional 속성에 따라 변경되는 JOIN 방식 (LEFT JOIN vs INNER JOIN)

## 1. 상황과 문제점

### 예시 Entity

```java
public class CategoryEntity extends BaseEntity {
	
    ...
	
	@Column(name = "name", nullable = false, length = 20)
	private String name;

    ...

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "parent_id")
	private CategoryEntity parentCategoryEntity;

    ...
}
```

### 예시 쿼리 메소드

```java
public interface CategoryJpaRepository extends JpaRepository<CategoryEntity, Long> {

	boolean existsByParentCategoryEntityIdAndName(Long parentCategoryId, String name);
}
```

위와 같은 Repository 메소드를 작성했을 때 기대했던 쿼리는 다음과 같았다.

``` sql
SELECT COUNT(*) FROM category WHERE parent_id = ? AND name = ?
```

하지만 실제로 생성된 쿼리는 예상하지 못한 LEFT JOIN을 포함한 쿼리였다.

![query_result_leftJoin](query_result_leftJoin.png)

> 처음에는 단일 테이블 쿼리가 실행될 것이라고 생각했는데 JOIN 쿼리가 발생해서 "parent_id" 때문에 이렇게 변환됐나보다 싶었다.  
> 그런데 앞에 LEFT가 붙어있어서 왜 이런 결과가 나오는지 궁금하여 알아보았다.

## 2. JOIN 방식이 달라지는 이유

이 현상은 @ManyToOne에서의 optional 속성 때문이었다.

- optional = true (기본값): Hibernate가 엔티티의 연관관계가 null이 허용된다고 판단하여 LEFT JOIN을 선택한다.
- optional = false: 엔티티의 연관관계가 절대 null이 될 수 없음을 의미하여 Hibernate가 INNER JOIN을 선택한다.

optional = false를 적용하면 생성되는 쿼리는 다음과 같이 변경된다.

![query_result_innerJoin](query_result_innerJoin.png)

## 3. optional = false와 nullable = false의 차이

헷갈리기 쉬운 두 속성에 대해서도 정리해보겠다. 

| 속성	|의미	| 대상   |
|---|---|------|
|optional = false	|Java 객체가 null이면 안 됨|	JPA 레벨|
|nullable = false	|DB 컬럼이 NULL 허용 안 함 (NOT NULL)	|DDL (DB 테이블 생성) 레벨|

nullable = false는 DB 구조를 정의하는 시점에서 의미를 가지며, optional = false는 JPA가 쿼리를 생성할 때 영향을 준다.
즉, JPA 사용시점에 쿼리 결과에 영향을 미치는 것은 optional 뿐이다.

## 4. JOIN 없이 원하는 쿼리를 만드는 방법

이렇게 정리해뒀지만 `existsByParentCategoryEntityIdAndName`의 쿼리 메소드의 결과로 아래와 같은 단일 테이블이 조회되길 기대했다.

``` sql
SELECT COUNT(*) FROM category WHERE parent_id = ? AND name = ?
```

왜냐하면 엔티티를 조회 후 외래키가 매핑된 연관객체의 기본키에 접근할 떄는 추가 쿼리 없이 조회가 가능하기 때문이다.  
그래서 여기에서도 JOIN 없이 쿼리가 발생할 줄 알았다.  
하지만 쿼리메소드로 작성할 경우 Spring Data JPA는 최적화를 수행하지 않고 항상 JOIN을 생성한다.  

그렇기 떄문에 Join없이 사용하려면 아래와 같이 `@Query`어노테이션을 사용해야한다.

### 쿼리
``` java
	@Query("SELECT COUNT(c) > 0 FROM CategoryEntity c WHERE c.parentCategoryEntity.id = :id AND c.name = :name")
	boolean existsByParentCategoryEntityIdAndName(Long id, String name);
```
### 결과

![query_result_no_join](query_result_no_join.png)


