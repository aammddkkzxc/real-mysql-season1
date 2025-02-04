![image](https://github.com/user-attachments/assets/9ed831e2-4f9d-4239-a011-0052eea85413)# Lateral Derived Table

- Derived Table(파생 테이블)은 쿼리의 FROM 절에 서브쿼리를 통해 생성되는 임시 테이블
- 일반적으로 Derived Table은 선행테이블의 칼럼을 참조할 수 없으나, Lateral Derived Table은 가능
- LATERAL 키워드를 사용하여 선언, 참조한 값을 바탕으로 동적으로 결과 생성

## 기본 동작 방식

![image](https://github.com/user-attachments/assets/16d95b5f-e35a-4e9e-b84b-fe754a6d91f8)

**쿼리의 역할**
- employees 테이블:
  - 모든 직원의 정보를 가져온다
  - e.emp_no는 직원 번호를 나타낸다
- LATERAL JOIN:
  - 각 직원(e.emp_no)에 대해, sales 테이블에서 해당 직원의 판매 기록을 동적으로 조회
- 서브쿼리는 다음 두 가지를 계산
  - sales_count: 해당 직원의 판매 건수 (COUNT(*)).
  - total_sales: 해당 직원의 총 판매 금액 (SUM(total_price)), 값이 없으면 0으로 대체 (IFNULL 사용).
- LEFT JOIN:
  - 직원이 판매 기록이 없는 경우에도 직원 정보를 유지합니다. 이 경우 sales_count와 total_sales는 각각 0 또는 NULL로 표시
- 결과 컬럼:
  - 최종적으로 직원 번호(e.emp_no), 판매 건수(s.sales_count), 총 판매 금액(s.total_sales)을 반환
- 이와 같이 레터럴로 조인이 실행되는 경우 일반적으로 서브 쿼리의 WHERE절에 조인 조건이 사용
  - emp_no 컬럼에 대한 참조가 where 절에 명시된 것을 알 수 있다.
  - 조인 조건을 명시하는 부분인 ON 절에는 문법을 준수하기 위해 true 값을 명시해 쿼리를 수행

**실행 계획**
- Lateral Derived Table의 경우 외부에 있는 선행 테이블인 Employees테이블에 대해 의존적이므로 실행 계획의 Select 타입에 DependentDerived라고 표시된 것을 알 수 있다.
-  Lateral 키워드를 사용해서 선행 테이블의 컬럼을 참조하는 경우 결과 데이터가 선행테이블에 의존적
-  쿼리 실행 시 처리 순서에 이러한 부분이 영향을 받는다는 점을 인지해서 사용해야 한다.

```
일반 join 과 비교
SELECT e.emp_no,
       COUNT(s.emp_no) AS sales_count,
       IFNULL(SUM(s.total_price), 0) AS total_sales
FROM employees e
LEFT JOIN sales s ON e.emp_no = s.emp_no
GROUP BY e.emp_no;
```

![image](https://github.com/user-attachments/assets/7f60dca2-af87-46ad-80fe-76b1b566b661)

위 예제에서는 단순히 직원별 판매 건수와 총 판매 금액을 계산하는 작업이므로 일반적인 조인을 사용하는 것이 더 직관적이고 성능 면에서도 유리할 가능성이 크다

## LATERAL 활용 예제 1 - 종속 서브 쿼리의 다중 값 반환
- 부서별 가장 먼저 입사한 직원의 입사일과 직원 이름을 조회한다고 가정

![image](https://github.com/user-attachments/assets/5f7a2efb-b2f6-45ed-b5c3-65584166d1a1)

- select 절에서 서브쿼리를 사용하는 경우 서브쿼리의 결과가 하나의 스칼라 값이어야 가능하다. -> 에러 발생

![image](https://github.com/user-attachments/assets/a7eebdd0-9619-4ffa-b104-fcca6c56685c)

- 두개의 서브 쿼리로 나누면 에러 없이 가능하다

![image](https://github.com/user-attachments/assets/5d811086-4629-4858-9fd7-2a3fa2a841f1)

- 다만 이 경우는 동일한 조건(de.dept_no = d.dept_no)과 테이블 조인을 포함하고 있다
- 각 부서(departments 테이블의 행)에 대해 서브쿼리가 각각 독립적으로 실행되므로, 동일한 데이터를 불필요하게 여러 번 조회하게 된다.

![image](https://github.com/user-attachments/assets/0fa8c006-a473-49e6-bfe2-f19b3557e8b5)

- departments 테이블
  - 메인 테이블로, 각 부서의 이름(d.dept_name)을 기준으로 데이터를 조회
- LATERAL JOIN:
  - 각 부서(departments의 행)에 대해 서브쿼리를 실행
  - 서브쿼리는 해당 부서에 속한 직원들 중 가장 빠른 입사일(e.hire_date)과 해당 직원의 이름(e.first_name, e.last_name)을 반환
- 서브쿼리
  - dept_emp와 employees를 조인하여 부서별 직원 데이터를 가져온다
  - WHERE de.dept_no = d.dept_no: 현재 부서(d.dept_no)에 해당하는 직원만 필터링
  - ORDER BY e.hire_date LIMIT 1: 입사일 기준으로 정렬 후 가장 빠른 한 명의 데이터를 반환
- 결과
  - 각 부서의 이름, 가장 빠른 입사일, 그리고 해당 직원의 이름이 출력
- 생각해 봐야 할 점
  - LATERAL JOIN은 메인 테이블(departments)의 각 행마다 서브쿼리를 실행하므로, 부서 수가 많아질수록 서브쿼리가 반복적으로 실행
  - 서브쿼리에서 ORDER BY e.hire_date LIMIT 1이 사용되므로, 각 부서에 대해 정렬 작업이 발생

## LATERAL 활용 예제 2 - SELECT 절 내 연산 결과 반복 참조

 - 일별 매출 데이터를 조회하는 쿼리

![image](https://github.com/user-attachments/assets/c2570b8a-e9a2-4d2e-99b4-ead50ebf92ed)

- SELECT문 내에서 연산 결과를 참조하기 위해 동일한 연산을 중복 기재해서 사용하게 된다

![image](https://github.com/user-attachments/assets/656dc625-4fc3-4f43-8c55-89e2aad0f83b)

- 쿼리의 중복 연산을 제거하고 가독성 향상

## LATERAL 활용 예제 3 - 선행 데이터를 기반으로 한 데이터 분석

- 퍼널 분석에 유용할 수 있다.
- 처음 서비스에 가입하고 나서 일주일 내로 결제 완료한 사용자의 비율
  - 2024년 1월에 가입한 유저들을 대상으로 분석
  - 사용자 관련 이벤트 데이터를 저장하는 user_events 테이블을 활용

![image](https://github.com/user-attachments/assets/97ff87e0-f8b7-413d-b632-01ce8e33ec1b)

<br>

- LATERAL 사용X

![image](https://github.com/user-attachments/assets/b6175621-24e6-4114-93f1-4518898fe313)

- 두 번째 drived table 부분에서 비효율이 발생한다
- 유저 이벤츠 테이블에서 결제에 해당하는 전체의 데이터들에 대해 그룹핑을 수행하기 때문
- 즉, 그룹핑 대상 데이터에 2024년 1월에 가입한 유저들에 대한 데이터뿐만 아니라 분석 대상이 아닌 다른 유저들의 데이터도 모두 포함이 되는 것

<br>

- LATERAL 사용O
  
![image](https://github.com/user-attachments/assets/e20c14b8-c7df-4080-b334-e152be1051d7)

- 선행 테이블의 컬럼을 참조하여 User Events 테이블에서 분석 대상 유저들 각각에 대해 가입 후 일주일 내에 결제한 이력이 하나라도 있는지를 확인
- 두 번째 테이블에서는 분석 대상 유저들에 대해서만 추가적으로 조건을 확인하므로 앞선 쿼리보다는 좀 더 효율적으로 처리되는 것을 알 수 있다.
- 데이터 분석 과정이 세분화 되고 조건이 복잡해 질 수록 성능에서 더 우수해진다

## LATERAL 활용 예제 4 - TOP N 데이터 조회

- 뉴스 카테고리별로 조회수가 가장 높은 뉴스 기사 3개를 추출하는 쿼리

<br>

- LATERAL 사용X

![image](https://github.com/user-attachments/assets/1ba20633-703e-428f-9ef8-c6d1ef10e1ad)

- ROW_NUMBER() 함수 사용
  - ROW_NUMBER() 윈도우 함수는 각 카테고리(a.category_id)별로 기사를 조회 수(a.views) 기준 내림차순으로 정렬하고, 순위를 매긴다
  - PARTITION BY는 카테고리별로 데이터를 나누고, ORDER BY는 조회 수를 기준으로 순위를 정한다
- 서브쿼리 사용
  - 서브쿼리에서 각 카테고리별로 순위가 매겨진 데이터를 생성
  - 최종적으로 WHERE x.article_rank <= 3 조건을 사용해 각 카테고리에서 상위 3개의 기사를 필터링
- 성능 분석
  - 실행 계획에 따르면, 이 쿼리는 ROW_NUMBER() 함수와 정렬 작업 때문에 "Using temporary"와 "Using filesort"를 사용
  - 이는 대량의 데이터에서 성능 저하를 유발할 수 있다.
  - 결과적으로 실행 시간이 2.88초로 비교적 느리다.

<br>

- LATERAL 사용O
  
![image](https://github.com/user-attachments/assets/f2e76cc3-1f02-4802-9f1d-87e898ba8226)

- LATERAL 서브쿼리 사용:
  - LATERAL은 각 행에 대해 동적으로 서브쿼리를 실행하는 방식
  - 여기서 각 카테고리(c.id)에 대해 관련된 기사(articles)를 조회하고, 조회 수 기준 내림차순으로 정렬한 뒤 상위 3개만 선택합니다 (LIMIT 3).
- 카테고리별 처리:
  - LATERAL을 통해 카테고리별로 필요한 데이터만 처리하므로 불필요한 계산을 줄일 수 있다.
- 성능 분석:
  - 실행 계획에 따르면 "Backward index scan"을 사용하여 효율적으로 데이터를 검색
  - 인덱스를 활용해 정렬된 데이터를 빠르게 가져오므로 성능이 크게 향상
  - 실행 시간은 0.00초로 매우 빠름
