# window 함수 활용법(feat. group by와 다른점)

[프로그래머스 - 연도별 대장균 크기의 편차 구하기](https://school.programmers.co.kr/learn/courses/30/lessons/299310) 문제를 풀다가 window 함수의 유용함을 알게 되어서 정리해보려고 한다.  

- 원래 풀이
```sql
SELECT
    ym.YEAR,
    (ym.max - ed.SIZE_OF_COLONY) AS YEAR_DEV,
    ed.ID
FROM
    ECOLI_DATA ed
        JOIN
    (
        SELECT
            YEAR(DIFFERENTIATION_DATE) AS YEAR,
            MAX(SIZE_OF_COLONY) AS max
        FROM
            ECOLI_DATA
        GROUP BY
            YEAR(DIFFERENTIATION_DATE)
    ) ym
    ON
    YEAR(ed.DIFFERENTIATION_DATE) = ym.YEAR
ORDER BY
    ym.YEAR ASC,
    YEAR_DEV ASC;
```

- window 함수 적용 풀이
```sql
SELECT 
    YEAR(DIFFERENTIATION_DATE) AS YEAR, 
    (MAX(SIZE_OF_COLONY) OVER (PARTITION BY YEAR(DIFFERENTIATION_DATE)) - SIZE_OF_COLONY) AS YEAR_DEV, 
    ID
FROM 
    ECOLI_DATA
ORDER BY 
    YEAR ASC, 
    YEAR_DEV ASC;
```

원래는 주어진 테이블과 주어진 테이블에서 각 년도별 최대 대장균 크기를 `JOIN`해서 풀었는데 window 함수를 적용하면 `JOIN`없이 간단하게 풀 수 있다.
지금부터 window 함수에 대해서 알아보겠다.


## Window 함수란?
Window 함수는 특정 데이터 범위(window)를 제한하여 결과를 기존 테이블에 반환하는 함수이다.

### Window 함수의 특징
- 원본 데이터를 유지 : `GROUP BY`는 그룹화된 데이터를 반환하지만, Window 함수는 원본 데이터의 행을 유지하면서 그룹화 후 반환된 데이터를 추가 열로 제공한다.
- `OVER`절로 정의 : `PARTITION BY`와 `ORDER BY`를 사용하여 데이터 그룹화와 정렬 기준을 정할 수 있다.

## Window 함수와 Group BY의 차이
둘 다 그룹화를 하여 결과를 얻는다는 점에서 비슷해서 헤갈릴 수 있기 때문에 표를 통해 정리해보겠다.  

| **특징**        | **Window 함수**                      | **Group By**              |
|---------------|------------------------------------|---------------------------|
| **데이터 유지 여부** | 원본 데이터를 유지하며 계산 결과를 추가             | 데이터를 요약하여 각 그룹당 하나의 행만 반환 |
| **주요 목적**     | 특정 범위(Window) 내에서 계산 및 분석          | 데이터 그룹화 및 집계              |
| **결과 형태**     | 기존 데이터 + 계산 결과                     | 그룹화된 데이터                  |
| **사용 위치**     | SELECT, ORDER BY                   | SELECT, HAVING            |

## Window 함수의 구조
```sql
<윈도우 함수>(<필드>) OVER ([PARTITION BY <그룹 기준>] [ORDER BY <정렬 기준>])
```

### 구성 요소
1. 윈도우 함수
    - SUM, AVG, COUNT, RANK, ROW_NUMBER 등 다양한 함수 사용 가능.
2. PARTITION BY
   - 데이터를 특정 기준으로 그룹화.
   - 생략하면 전체 데이터를 하나의 그룹으로 간주.
3. ORDER BY
   - 그룹 내에서 데이터를 정렬하는 기준.
   - 생략 가능

## Window 함수 활용 방법
### 데이터 예시
| employee_id | 	department_id  | 	salary  |
|-------------|-----------------|----------|
| 1           | 	10	            | 3000     |
 | 2           | 	10	            | 3000     |
 | 3           | 	10	            | 5000     |
 | 4           | 	20	            | 4000     |
 | 5           | 	20	            | 6000     |

### 부사별 급여 순위 계산
`PARTITION BY`와 `ORDER BY`를 조합하여 그룹별 순위를 매길 수 있다.
```sql
SELECT 
    employee_id, 
    department_id, 
    salary, 
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank_within_department
FROM employees;
```  

**결과**

| employee_id	 | department_id	  | salary	 | rank_within_department |
|--------------|-----------------|---------|------------------------|
| 3	           | 10	             | 5000    | 	1                     |
| 1	           | 10	             | 3000    | 	2                     |
 | 2            | 	10             | 	3000   | 	2                     |
 | 5            | 	20             | 	6000   | 	1                     |
| 4            | 	20             | 	4000   | 	2                     |


### 누적 합계 계산
`ORDER BY`를 활용해 특정 정렬 기준에 따른 누적 합계를 구할 수 있다.
```sql
SELECT 
    employee_id, 
    salary, 
    SUM(salary) OVER (ORDER BY salary ASC) AS cumulative_salary
FROM employees;
```
**결과**

| employee_id | salary | cumulative_salary |
|-------------|--------|--------------------|
| 1           | 3000   | 3000              |
| 2           | 3000   | 6000              |
| 4           | 4000   | 10000             |
| 3           | 5000   | 15000             |
| 5           | 6000   | 21000             |

### 전체 데이터에 대한 통계 계산
`PARTITION BY` 없이 `OVER()`를 사용하면 전체 데이터를 하나의 그룹으로 간주하여 계산한다.
```sql
SELECT 
    employee_id, 
    salary, 
    AVG(salary) OVER () AS average_salary
FROM employees;
```
**결과**

| employee_id | salary | cumulative_salary |
|-------------|--------|-------------------|
| 1           | 3000   | 4200              |
| 2           | 3000   | 4200              |
| 3           | 5000   | 4200              |
| 4           | 4000   | 4200              |
| 5           | 6000   | 4200              |

## GROUP BY와 Window 함수의 차이 실제로도 비교해보기
### GROUP BY 사용
```sql
SELECT 
    department_id, 
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;
```
**결과**

| department_id | avg_salary |
|---------------|------------|
| 10            | 3666.67    |
| 20            | 5000.00    |

- 행이 그룹화되며 부서별로 하나의 결과만 반환된다.

### Window 함수 활용
```sql
SELECT 
    employee_id, 
    department_id, 
    salary, 
    AVG(salary) OVER (PARTITION BY department_id) AS avg_salary
FROM employees;
```
**결과**

| employee_id | department_id | salary | avg_salary |
|-------------|---------------|--------|------------|
| 1           | 10            | 3000   | 3666.67    |
| 2           | 10            | 3000   | 3666.67    |
| 3           | 10            | 5000   | 3666.67    |
| 4           | 20            | 4000   | 5000.00    |
| 5           | 20            | 6000   | 5000.00    |

- 원본 데이터를 유지하며, 부서별 평균 값이 각 행에 추가된다.

## 결론
- `GROUP BY`는 데이터를 그룹화하여 그룹별 결과만 반환하는 반면, Window 함수는 원본 데이터를 유지하며 그룹별 계산 결과를 추가로 제공한다.
- 특히, `PARTITION BY`와 `ORDER BY`를 적절히 활용하면 기존의 복잡한 `GROUP BY`와 `JOIN`을 대체할 수 있다.