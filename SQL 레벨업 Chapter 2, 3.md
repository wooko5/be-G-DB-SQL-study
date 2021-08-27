### 2021.08.18 DB 스터디 (SQL 레벨업)

## 2장 SQL 기초



0. 너무 기초적인 select, update, delete, insert 등은 저보다 더 잘 설명해주시는 분들이 많기 때문에 이번 챕터에서는 다루지않고 제가 생각했을 때 잘 모르거나, 궁금한 부분들을 위주로 정리했습니다.

   <br>

   <br>

   <br>

1. 뷰(View)

   - 뷰는 사용자에게 접근이 허용된 자료만을 제한적으로 보여주기 위해 하나 이상의 기본 테이블로부터 파생된, 이름을 가지는 가상 테이블이다.

     - select구문을 데이터베이스 안에 저장할 수 있도록 해준다. 하지만 테이블과 다르게 내부에 데이터를 보유하지 않아서 `가상테이블`이라는 말을 갖게 되었다

   - 뷰 생성1

     - ```mysql
       CREATE VIEW [뷰이름](필드이름1, 필드이름2, ....) AS ... 
       ```

   - 뷰 생성2

     - ```mysql
       CREATE VIEW my_view(view_address, cnt) AS SELECT address, COUNT(*) FROM Address GROUP BY address;
       ```

   - 뷰 사용

     - ```mysql
       SELECT view_address, cnt
       FROM Address;
       ```

     <BR>

     <BR>

     <BR>

2. 서브쿼리

  - ```mysql
    SELECT name
    FROM Address1
    WHERE name IN(SELECT name FROM Address2);
    ```

    - 서브쿼리를 이용하면 Address2 테이블이 수정될 때마다 따로 수정하지 않아도 된다
    - 그러나 상수를 직접 입력하는 코드라면 테이블이 수정될 때마다 수정해야하는 번거로움이 존재함
    - `조인`을 통해 해결할 수 있는 문제라면 `서브쿼리`를 통해 조회를 2번 이상하는 것보다 `조인`을 이용하는 것을 `시간효율성` 측면에서 권장한다

  <br>

  <br>

  <br>

3. CASE WHEN

   - SQL 분기문
     - SQL에는 따로 switch나 if 조건문 같은 절차지향언어에서 사용하는 `문장단위` 분기문은 없다. 왜나하면 SQL은 절차지향적인 언어가 아니라 `집합지향적`인 언어이기 때문이다. 그래서 SQL에서는 `식`을 중심으로 조건 분기를 실행한다.
       - 집합지향적이다는 말은 SQL은 내부적으로는 절차지향적으로 작동하지만 사용자가 절차지향적 사고를 지양하고, 테이블과 레코드같은 데이터를 집합적으로 생각하기 위함에서 나온 말이다
     - 이를 위한 명령어가 `CASE WHEN ~ THEN` 이다

   - ```mysql
     SELECT name, address, 
     CASE WHEN address = '서울시' THEN '경기'
     CASE WHEN address = '인천시' THEN '경기'
     CASE WHEN address = '부산시' THEN '영남'
     CASE WHEN address = '속초시' THEN '관동'
     CASE WHEN address = '서귀포시' THEN '호남'
     ELSE NULL END as district
     FROM Address;
     ```

   - CASE식 작동원리

     - 절차지향적 언어의 조건 분기는 문장을 실행하고 딱히 반환되지 않지만, SQL의 조건 분기는 특정한 값(상수)를 반환한다

     <br>

     <br>

     <br>

4. SQL 집합연산

   - UNION

   - INTERSECT

   - EXCEPT

     <br>

     <br>

     <br>

5. 윈도우함수 

   - 집약 기능이 없는 GROUP BY 구문이라고 생각하면 쉽다

     ```mysql
     SELECT address, COUNT(*) OVER(PARTITION BY address)
     FROM Address
     ```

     <br>

     <br>

     <br>
     
     <br>
     
     <br>
     
     <br>


## 3장 SQL의 조건 분기



1. UNION을 사용한 쓸데 없이 긴 표현

   - 무조건은 아니지만 대부분의 경우에서 합집합인 UNION을 쓰면 내부적으로 여러 개의 SELECT문을 실행하는 실행계획으로 해석되기 때문에 테이블에 접근하는 횟수가 많아져서 I/O 비용이 크게 늘어납니다

     - 즉 정확한 판단이 없는 UNOIN 사용은 자제하자

   - UNION, CASE 비교 코드

     - ```mysql
       SELECT item_name, year, price_tax_ex AS price
       FROM Items
       WHERE year <= 2001
       UNION 
       SELECT item_name, year, price_tax_ex AS price
       FROM Items
       WHERE year >= 2002;
       ```

     - ```mysql
       SELECT item_name, year, 
       CASE WHEN year <= 2001 THEN price_tax_ex
            WHEN year >= 2002 THEN price_tax_in
       END AS price
       FROM Items;
       ```

     - 둘의 수행결과는 똑같지만 CASE의 수행시간은 두 배 빠르다. Items 테이블에 대한 접근이 1회 이기 때문이다

2. UNION을 사용하는 경우 

   - Merge 대상이 되는 SELECT 구문들에서 사용하는 테이블이 상이한 경우이다.
     - 즉 합치려는 대상이 모두 다른 테이블에서 온 경우

3. IN

   - IN은 매개변수로 단순한 스칼라 말고도 배열을 입력할 수도 있다

4. 팁

   - IN, CASE식으로 조건 분기를 표현할 수 있다면 테이블에 하는 스캔을 크게 감소시킬 가능성이 존재한다 
     - DB의 효율성 증대
   - 절차지향적 사고의 구문에서 식(Expression)으로의 전환을 연습해보자