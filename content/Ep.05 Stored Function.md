# Stored Function
- MySQL의 **Stored Function(스토어드 함수)**는 사용자가 정의한 함수로, 데이터베이스 서버에 저장되어 SQL 문에서 호출할 수 있는 기능
- 이를 통해 복잡한 SQL 로직을 모듈화하고 재사용할 수 있다.

### Stored Function의 특징
- 단일 값 반환: RETURN 문을 사용해 하나의 값을 반환
- SQL 문에서 호출 가능: SELECT, INSERT, UPDATE, DELETE 등의 SQL 문에서 사용할 수 있다
- 입력 매개변수: 하나 이상의 입력 매개변수를 받을 수 있으며, 반환 타입을 명시해야 한다.
- 보안성과 성능: 데이터베이스 서버에서 실행되므로 네트워크 오버헤드가 줄고, 권한 관리를 통해 보안성을 강화할 수 있다.
- 재사용성: 자주 사용하는 로직을 함수로 정의해 재사용할 수 있다.

### Stored Function 생성 방법
Stored Function은 CREATE FUNCTION 구문을 사용해 생성. 기본 구조는 다음과 같다

```
DELIMITER $$

CREATE FUNCTION function_name(parameters)
RETURNS return_type
[DETERMINISTIC | NOT DETERMINISTIC]
[CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA]
BEGIN
    -- 함수 로직
    RETURN value;
END$$

DELIMITER ;
```

**주요 구성 요소**
- function_name: 함수 이름.
- parameters: 입력 매개변수(이름과 데이터 타입).
- RETURNS return_type: 반환 값의 데이터 타입.
- 특성 설정:
  - DETERMINISTIC: 동일한 입력에 대해 항상 동일한 결과를 반환.
  - CONTAINS SQL, NO SQL, READS SQL DATA, MODIFIES SQL DATA: 함수가 사용하는 SQL 작업의 범위를 지정.
- 본문(BEGIN ... END): 함수의 로직을 작성하는 부분.

**예시**
두 숫자의 합계를 계산하는 함수
```
DELIMITER $$

CREATE FUNCTION add_numbers(a INT, b INT)
RETURNS INT
DETERMINISTIC
BEGIN
    RETURN a + b;
END$$

DELIMITER ;

-- 호출 예시
SELECT add_numbers(10, 20);
```

이번 에피소드에선 DETERMINISTIC, NOT DETERMINISTIC 속성에 대해서 알아본다

## DETERMINISTIC, NOT DETERMINISTIC

### DETERMINISTIC(확정적)
- 입력이 동일하면 언제 실행하든지 관계없이 항상 출력이 동일하다
  - 입력이 동일하다는 것은 함수의 인자뿐만 아니라 함수가 참조하는 데이터도 모두 동일하다는 뜻
- 예) 오늘 가입한 사용자의 수를 가져오는 함수
  - stored function이 실행되는 도중에도 사용자의 가입은 계속 진행되기 때문에 이 함수는 호출할 때마다 결과가 달라질 수도 있다.
  - 사용자 테이블의 레코드가 달라지는 지면 입력이 달라지는 것으로 해석된다.
  - 따라서 MySQL 서버에서 SELECT를 포함해서 하나의 SQL 문(statement)이 실행될 때 해당 문장이 시작된 시점의 데이터 스냅샷을 기준으로 동작 하도록 구현되어있다.
    - MySQL 서버에서 실행되는 Query 문장 하나는 동일한 데이터 상태를 보게 된다.
  - 결과적으로 하나의 문장 내에서는 Stored Function이 여러 번 호출된다고 하더라도 테이블의 데이터는 해당 시점의 스냅샷을 보게 된다.
    - 테이블의 데이터가 변하지 않는다고 볼 수 있다.

### NOT DETERMINISTIC(비 확정적)
- 동일한 입력값이 주어지더라도 호 시점에 따라 다른 결과를 반환할 수 있다.

### 예시

![image](https://github.com/user-attachments/assets/43647c14-d59a-4d95-88c5-d7a254206448)

- func1과 func2 함수 모두 똑같지만, func1은 deterministic func2는 not deterministic 로 정의
- 두 함수는 단순히 function1__called 세션 변수와 function2__called 세션 변수의 값을 1식 증가시키는 연산만 수행

<br>

![image](https://github.com/user-attachments/assets/5f08c5e1-2baf-4f64-bf70-33c137c558dd)

- 이제 tab이라는 테이블 id 컬럼의 값이 function1과 function2 함수의 function1__called와 function2__called라는 세션 변수의 값이 결과 값이랑 동일한 레코더를 찾는 쿼리 실행
- 여기에서 두 쿼리의 실행 결과 값은 중요하지 않고 쿼리의 실행결과 function1__called와 function2__called라는 세션 변수의 값이 어떻게 바뀌었는지가 중요하다.
- 두 함수의 호출 횟수가 많이 차이가 나는 걸 확인할 수 있다.
- not deterministic 함수의 호출 횟수가 많다 -> 함수가 그만큼 많이 실행되었고 이 함수가 사용된 쿼리 처리 시에 실행시간이나 CPU 또는 메모리 사용량이 한 4배 정도 더 추가되었다는 것을 예상

<br>

![image](https://github.com/user-attachments/assets/a033cbbb-968a-409a-8901-7a108317cc81)

**func1**
- primary key 인덱스를 const 타입으로 읽었다는 것을 확인 할 수 있다.
- primary key 를 const 타입으로 접근하는 것은 테이블의 레코드를 1건만 읽었다는 것을 의미
  - 매우 빠르게 처리되는 실행 계획이다
- 그리고 실행 계획 하단에 Note 에 명시된 query 에서도 where 절의 function1 함수 호출이 없어진 것을 알 수 있다
  - 이는 최적화 되어, 함수 호출 부분 자체가 사라졌다는 것을 의미

**func2**
- 실행 계획의 type 값이 ALL 이라는 것을 확인 할 수 있다
- 쿼리가 full table scan으로 처리 되었다는 것을 의미
- Note 를 보면 where 절의 func2 함수 호출 코드가 그대로 남아 있는 것도 확인 할 수 있다.
- 즉, not deterministic 함수의 호출 횟수가 많은 이유는 쿼리의 **실행 계획**에서 확인 한 것 처럼 인덱스를 사용하지 못하고 **full table scan 이 사용**되었기 때문

### 분석
- Not Deterministic 함수는 입력이 동일해도 실행하는 시점에 따라서 다른 값을 반환할 수 있다는 것을 기억하자.
- 즉, 레코드 건수가 10건인 테이블에서 where 조건절의 id 값이 stored function 호출 결과 값과 동일한 레코드를 찾으려고 하는 것
- Not Deterministic 속성의 function2 함수는 레코드를 매번 비교할 때마다 function2의 함수 결과 값이 달라진다고 정의되어 있다는 것.
  - MySQL 서버의 옵티마이저는 function2 함수의 결과 값을 상수로 취급하지 못한다.
  - 테이블의 레코드를 한 건씩 읽어서 비교할 때마다 function2 함수를 실행해서 결과 값을 매번 새롭게 가져와서 비교를 수행하게 된다.
- 결과적으로 Not Deterministic으로 정의된 함수를 Where 조건절의 비교값으로 사용하게 되면 MySQL 서버는 해당 칼럼에 인덱스가 있다 하더라도 인덱스 최적화를 사용하지 못한다
  - 풀테이블 스캔으로 처리

### NOT DETERMINISTIC Built-in Function
- RAND()
- UUID()
- SYSDATE()
- NOW() 등등
- 위와 같은 함수를 사용한 표현식 또한 NOT DETERMINISTIC 속성을 상속 받게 된다
  - WHERE col = (RAND() * 1000)

### NOT DETERMINISTIC 함수 예외적인 경우
- NOW() 와 SYSDATE()
- 둘다 모두 NOT DETERMINISTIC 함수이다

![image](https://github.com/user-attachments/assets/47dd381b-90ac-4ebf-be7a-cd686875a190)

- NOW() 하나의 문장 안에서는 deterministic처럼 인식되도록 구현되어 있는 반면 SYSDATE()는 not deterministic처럼 레코드 건 단위로 매번 새로 호출되고 그 결과 값이 달라지는 것을 확인할 수 있다.
  - NOW()의 결과 값과 비교하는 경우에는 인덱스를 사용해서 최적화 가능하다. SYSDATE()의 결과 값과 비교하는 경우에는 인덱스를 사용할 수 없고 풀테이블스캔을 사용하게 된다.
- NOW 함수와 SYSDATE 함수의 이런 차이를 인지하지 못하는 사용자가 너무 많았고, 혼용되어서 사용되어 왔다.
  - 따라서 sysdate-is-now라는 시스템 설정이 추가되었다.
  - MySQL Server의 설정파일에 이 옵션이 명시가 되면 MySQL Server는 SYSDATE()도 NOW()와 동일하게 deterministic으로 인식하도록 만들어준다.
  - SYSDATE()가 not deterministic 방식으로 작동될 필요가 있는 경우가 거의 없다고 한다.
  - sysdate-is-now 설정이 표준이 되는 것도 권장된다고 한다

 ### Stored Function 주의사항
 - NOT DETERMINISTIC이 기본 속성이기 때문에 Stored Function을 생성할 때 Deterministic 옵션을 별도로 명시하지 않으면 NOT DETERMINISTIC 속성을 가지게 된다.
 - 함수가 NOT-DETERMISTIC이어야 하는 경우는 잘 없다고 한다.
 - 따라서 서비스별로 요건을 검토해서 NOT-DETERMINISTIC일 필요가 없다면, 반드시 Stored Function에 DETERMINISTIC 키워드를 명시해서 사용.
