# JPA getSingleResult 사용 시 값이 없을 떄

회원가입 로직을 짜고 테스트 코드를 작성하다가 중복을 확인 하는 부분에서 `NoResultException`이 발생하였다.  
순수 JPA로 개발을 하고 있는데 문제의 상황에 놓인 코드는 다음과 같다.  

```java
	@Override
	public Optional<Account> findByUsername(String username) {
		EntityManager em = EntityManagerHolder.getEntityManager();

		String jpql = "SELECT a FROM Account a WHERE a.username = :username";
			Account account = em.createQuery(jpql, Account.class)
				.setParameter("username", username)
				.getSingleResult();
			return Optional.ofNullable(account);
	}
```
처음에 기대한 결과는 찾는 값이 없을 떄 null이 반환될 것을 기대하였고, null을 안전하게 처리하기 위해 **Optional**로 감싸서 처리하였다.  
그리고 해당 코드를 테스트해봤는데 `NoResultException`이 발생하였다.  
실제로 `getSingleResult()`의 메서드를 확인해보면 아래와 같은 경우에 Exception을 던지도록 설계하라고 나와있다.

``` java

public interface TypedQuery<X> extends Query {
    
        ...

     /**
     * Execute a SELECT query that returns a single result.
     * @return the result
     * @throws NoResultException if there is no result (결과가 없을 때)
     * @throws NonUniqueResultException if more than one result (둘 이상의 결과가 있을 때)
     * @throws IllegalStateException if called for a Jakarta
     *         Persistence query language UPDATE or DELETE statement
     * @throws QueryTimeoutException if the query execution exceeds
     *         the query timeout value set and only the statement is
     *         rolled back
     * @throws TransactionRequiredException if a lock mode other than
     *         <code>NONE</code> has been set and there is no transaction
     *         or the persistence context has not been joined to the
     *         transaction
     * @throws PessimisticLockException if pessimistic locking
     *         fails and the transaction is rolled back
     * @throws LockTimeoutException if pessimistic locking
     *         fails and only the statement is rolled back
     * @throws PersistenceException if the query execution exceeds 
     *         the query timeout value set and the transaction 
     *         is rolled back 
     */
    X getSingleResult();   
    
    ...
}
```

그래서 이 경우에 결과값이 없을 때 null값을 던지도록 유도하려면 아래와 같이 구현해야한다.
```java
	@Override
	public Optional<Account> findByUsername(String username) {
		EntityManager em = EntityManagerHolder.getEntityManager();

		String jpql = "SELECT a FROM Account a WHERE a.username = :username";
		try {
			Account account = em.createQuery(jpql, Account.class)
				.setParameter("username", username)
				.getSingleResult();
			return Optional.of(account);
		} catch (NoResultException e) {
			return Optional.empty();
		}
	}
```

다른 방법으로는 `getResultList()`로 처리하여 List로 반환받는 방법도 있다.  

나의 경우에는 중복 아이디는 저장할 수 없어서 데이터베이스에는 하나의 레코드만 존재한다.  
그리고 리스트의 경우에는 테이블 전체를 확인할 것 이기 때문에 하나의 값만 찾는 `getSingleResult()`를 사용하는 것이 더 낫다고 판단했다.  

인터페이스에 자바독으로 메서드 명세를 작성해두었기 때문에 잘 모르겠으면 확인해보고 코드를 짜는 습관을 들여야겠다.
