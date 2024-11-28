# MySQL에서 SELECT 별칭은 GROUP BY, ORDER BY, HAVING 절에서 사용 가능하다

## 상황
프로그래머스에서 [카테고리 별 상품 개수 구하기](https://school.programmers.co.kr/learn/courses/30/lessons/131529?language=mysql) 문제를 풀다가 의문이 들었다.  
다음과 같이 쿼리를 작성했는데 통과가 되었다.
```sql
`SELECT` LEFT(PRODUCT_CODE, 2) CATEGORY, COUNT(*)
FROM PRODUCT 
`GROUP BY` CATEGORY 
`ORDER BY` 1 ASC;
```
`GROUP BY` 절에서 `CATEGORY`라는 별칭을 사용했음에도 문제가 없었다. 왜 그럴까?

## SQL 실행 순서
내가 알는 SQL 실행순서는 아래와 같았다.
1. FROM : 가장 먼저 테이블을 선택하고 `JOIN`이 있다면 함께 진행
2. WHERE : `FROM`에서 가져온 데이터 필터링
3. GROUP BY : 데이터를 그룹화하여 집계 가능하게 만듦
4. DISTINCT : `DISTINCT`가 걸린 칼럼의 중복 행들을 제거
5. SELECT : 조회할 행들이 결정됐으므로 선택한 칼럼들을 반환
6. ORDER BY : 반환 후 정렬 기준으로 정렬
7. LIMIT : 일부만 반환하고 싶다면 `LIMIT`을 사용해서 반환 개수 조절

> 위 순서에서 알 수 있듯, GROUP BY는 SELECT보다 먼저 실행된다.  
그런데도 MySQL에서는 `SELECT` 절의 별칭을 GROUP BY에서 사용할 수 있다. 왜일까?
> 
## GROUP BY에 별칭이 허용되는 이유
[MySQL 공식문서 - 칼럼 별칭](https://dev.mysql.com/doc/refman/8.4/en/problems-with-alias.html)에 따르면 **MySQL은 `SELECT` 절에서 정의된 별칭을 `GROUP BY`, `ORDER BY`, `HAVING` 절에서 사용할 수 있도록 허용**하고 있다.  
이러한 동작은 MySQL 고유의 유연성 때문이며, 다른 DBMS(예: Oracle)에서는 허용되지 않는다.

예를 들어, 동일한 쿼리를 Oracle에서 실행하면 다음과 같은 오류가 발생한다:
```
ORA-00904: "CATEGORY": invalid identifier
```
이는 SQL 표준에서 `SELECT` 별칭은 `SELECT` 절이 끝나야 정의된다는 규칙을 엄격히 따르기 때문이다.

MySQL은 `SELECT` 절에서 별칭이 정의될 때, 이를 내부적으로 계산식으로 확장해 `GROUP BY`, `ORDER BY`, `HAVING` 절에서 참조할 수 있도록 처리한다.


## WHERE절에서는 별칭을 사용할 수 없는 이유
한편, `WHERE` 절에서는 `SELECT` 절의 별칭을 사용할 수 없다.  
이는 SQL의 실행 순서 때문인데, `WHERE` 절은 `SELECT` 절보다 먼저 실행되기 때문에 `SELECT`에서 정의된 별칭을 `WHERE`가 참조할 수 없다.  
예를 들어, 아래 쿼리는 오류가 발생한다:
```sql
`SELECT` LEFT(PRODUCT_CODE, 2) CATEGORY, COUNT(*) 
    FROM PRODUCT 
    `WHERE` CATEGORY ="A1"
    `GROUP BY` CATEGORY 
    `ORDER BY` 1 ASC;

// 실행결과 
ORA-00904: "CATEGORY": invalid identifier
```
해결하려면 서브쿼리를 사용해야한다.  
이 내용도 [MySQL 공식문서 - 칼럼 별칭](https://dev.mysql.com/doc/refman/8.4/en/problems-with-alias.html)에서 확인할 수 있다.

> `SELECT` 절 내부에서도 별칭을 다시 참조할 수 없다.  
이유는 `SELECT` 절이 모두 처리된 이후에 별칭이 정의되기 때문이다.

## GROUP BY에서 별칭 허용에 대한 추측
GROUP BY와 `SELECT` 절은 밀접하게 연관되어 있다.  
예를 들어, GROUP BY를 사용한 후 `SELECT` 절에 `*`를 사용하면 다음과 같은 상황이 발생한다:
- MySQL은 GROUP BY로 그룹화된 데이터를 기준으로 SELECT를 수행한다.
- 하지만 `SELECT` 절에 포함된 컬럼이 `GROUP BY`에 포함되지 않으면, 오류가 발생할 수 있다.

이처럼 ``GROUP BY``는 ``SELECT`` 절과 관련이 깊으며, MySQL은 ``GROUP BY``에서 ``SELECT``의 별칭을 허용해 쿼리문을 간결하게 사용할 수 있게 만든 것 같다.


## 결론
MySQL에서는 ``SELECT``에서 정의한 별칭을 ``GROUP BY``, ``ORDER BY``, ``HAVING`` 절에서 사용할 수 있도록 허용한다.  
이는 MySQL의 유연한 처리 방식 덕분이며, 코드의 가독성과 간결성을 높여준다.   
그러나 SQL 표준을 엄격히 따르는 DBMS에서는 이러한 동작이 허용되지 않으므로, 환경에 따라 작성 방식이 달라질 수 있음을 유의해야 한다.