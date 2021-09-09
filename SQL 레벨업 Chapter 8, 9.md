### 2021.09.08 DB 스터디 (SQL 레벨업)

## 8장 SQL의 순서



0. 이번 장에서는 SQL로 레코드의 순서를 효율적으로 사용하는 방법을 배워본다

   <br>

   <br>

   <br>

1. 레코드에 순번 붙이기

   - 기본 키가 한 개의 필드일 경우

     - 체중(Weights) 테이블 

       - <img src="C:\Users\wooko\AppData\Roaming\Typora\typora-user-images\image-20210908013009215.png" alt="image-20210908013009215" />

     - 윈도우 함수

       - ```mysql
         -- 기본 키는 student_id 이다
         SELECT student_id, 
         ROW_NUMBER() OVER (ORDER BY student_id) AS seq 
         FROM Weights;
         ```

     - 상관 서브쿼리

       - ```mysql
         -- 기본 키는 student_id 이다
         SELECT student_id, SELECT(COUNT(*) 
                                   FROM Weights W2
                                   WHERE W2.student_id <= W1.student_id
                                  ) AS seq
         FROM Weights W1;
         ```

     - **정리**

       - **두 연산의 결과는 동일하지만, 윈도우 함수는 스캔 횟수가 한 번이므로 윈도우 함수를 추천한다**
         - **상관 서브쿼리는 SELECT을 두 번 하므로 스캔 횟수가 두 번이다**
       - **또한 윈도우 함수는 테이블 전체 스캔이 아닌 인덱스 스캔을 사용한다**
         - **SQL문에서 필요로하는 모든 칼럼이 인덱스 구성 칼럼에 포함되는 경우 테이블에 대한 접근이 발생하지 않는다**
       
       <br>

   - 기본 키가 여러 개의 필드로 구성되는 경우

     - 체중(Weights2) 테이블 2
       - <img src="C:\Users\wooko\AppData\Roaming\Typora\typora-user-images\image-20210908014308897.png" alt="image-20210908014308897" style="zoom:25%;" />

       - 윈도우 함수

         - ```mysql
           -- 기본 키가 여러 개의 필드로 구성되는 경우(ROW_NUMBER)
           SELECT class, student_id,
              ROW_NUMBER() OVER(ORDER BY class, student_id) AS seq
              FROM Weights2;
           ```

       - 상관 서브쿼리

         - ```mysql
           -- 기본 키가 여러 개의 필드로 구성된느 경우(상관 서브쿼리: 다중 필드 비교)
           SELECT class, student_id,
              (SELECT COUNT(*)
                  FROM Weights2 W2
                  WHERE (W2.class, W2.student_id)
                     <= (W1.class, W1.student_id)) AS seq
           FROM Weights2 W1;
           ```

         - 장점

           - 필드 자료형을 원하는대로 지정할 수 있어서 숫자와 문자열, 문자열과 숫자도 가능하다.
           - 암묵적인 자료형 변환도 발생하지 않으므로 기본 키 인덱스도 사용할 수 있다.
           - 또한 필드가 3개 이상일 때도 간단하게 확장 할 수 있다.

         <BR>

   - 그룹마다 순번을 붙이는 경우

     - 윈도우 함수

       - ```mysql
         -- 학급마다 순번 붙이기(ROW_NUMBER)
         SELECT class, student_id,
            ROW_NUMBER() OVER(PARTITION BY class ORDER BY student_id) AS seq
         FROM Weights2;
         
         ```

     - 상관 서브쿼리

       - ```mysql
         -- 학급마다 순번 붙이가(상관 서브쿼리)
         SELECT class, student_id,
            (SELECT COUNT(*)
               FROM Weights2 W2
               WHERE W2.class = W1.class
                 AND W2.student_id < W1.student_id) AS seq
         FROM Weights2 W1;
         ```

         <BR>

   - 순번과 갱신

     - 윈도우 함수

       - 셀렉트 쿼리를 SET에 넣으면 됨

       - ```mysql
         -- 순번 갱신(ROW_NUMBER)
         UPDATE Weights3
            SET seq = (SELECT seq
                         FROM (SELECT class, student_id,
                                 ROW_NUMBER()
                                    OVER(PARTITION BY class
                                       ORDER BY student_id) AS seq
                               FROM Weights3) SeqTbl
                       WHERE Weights3.class = SeqTbl.class
                         AND Weights3.student_id = SeqTbl.student_id);
         ```

     - 상관 서브쿼리

       - ```mysql
         -- 순번 갱신(상관 서브쿼리)
         UPDATE Wights3
           SET seq = (SELECT COUNT(*)
                         FROM Weights3 W2
                         WHERE W2.class = Weights3.class
                           AND W2.student_id <= Weights3.student_id);
         ```

         

     <BR>

     <BR>

     <BR>

2. 레코드에 순번 붙이기 응용

   - 중앙값 구하기

     - 중앙값(Median) 개념

       - 모집단에서 추출한 표본에서 가장 중앙에 있는 값을 의미한다, 평균(Mean, 기대값)하고는 다른 개념
       - 아웃라이어(Outlier)에 영향을 받지 않는다
         - 아웃라이어는 표본의 내부 중앙에서 극단적으로 멀리 떨어져 있는 특이값을 의미한다

     - 집합 지향적 방법

       - <img src="C:\Users\wooko\AppData\Roaming\Typora\typora-user-images\image-20210908123438043.png" alt="image-20210908123438043" style="zoom:25%;" />

       - ```mysql
         -- 중앙값 구하기(집합 지향적 방법): 모집합을 상위와 하위로 분할
         SELECT AVG(weight)
            FROM (SELECT W1.weight
                    FROM Weights W1, Weights W2
                    GROUP BY W1.weight
                    -- S1(하위 집합)의 조건
                    HAVING SUM(CASSE WHEN W2.weight >= W1.weight THEN 1 ELSE 0 END)
                             >= COUNT(*) / 2
                    -- S2(상위 집합)의 조건
                      AND SUM(CASE WHEN W2.weight <= W1.weight THEN 1 ELSE 0 END)
                             >= COUNT(*) / 2) TMP;
         ```

       - 단점

         - 코드가 복잡해서 무엇을 하고 있는 것인지 이해하기 힘들다.
         - 성능이 나쁘다. (W1과 W2간에 JOIN이 발생)

     - 절차 지향적 방법 1 - 세계의 중심을 향해

       - <img src="C:\Users\wooko\AppData\Roaming\Typora\typora-user-images\image-20210908123503433.png" alt="image-20210908123503433" style="zoom:25%;" />

       - SQL에서 자연수의 특징을 활용하면 ‘양쪽 끝부터 숫자 세기’를 할 수 있다

       - ```mysql
         -- 중앙값 구하기(절차 지향형 1): 양쪽 끝에서 레코드 하나씩 세어 중간을 찾음
         SELECT AVG(weight) AS median
         FROM (SELECT weight,
                   ROW_NUMBER() OVER(ORDER BY weight ASC, student_id ASC) AS hi,
                   ROW_NUMBER() OVER(ORDER BY weight DESC, student_id DESC) AS lo
                 FROM Weights) TMP
         WHERE hi In(lo, lo+1, lo-1);
         ```

     - 절차 지향적 방법 2 - 2 빼기 1은 1

       - 정렬이 아까에 비해 1회로 줄어들고, SQL 표준으로 중앙값을 구하는 가장 빠른 방법이다.

       - ```mysql
         -- 중앙값 구하기(절차 지향적 방법) 2 : 반환점 발견
         SELECT AVG(weight)
         FROM (SELECT weight,
                 2 * ROW_NUMBER() OVER(ORDER BY weight)
                   - COUNT(*) OVER() AS diff
               FROM Weights) TMP
         WHERE diff BETWEEN 0 AND 2;
         ```

         <br>

   - 순번을 사용한 테이블 분할(?)

     - 단절 구간 찾기

       - 테이블 번호가 빠져 있는레스토랑의 예약 테이블이 있다고 가정하자
       - 빠져 있는 테이블을 `gap_start` ~ `gap_end` 와 같은 테이블 형식으로 구하자

     - 집합 지향적 방법 - 집합의 경계선

       - ```mysql
         -- 비어있는 숫자 모음을 표시
         SELECT (N1.num + 1) AS gap_start,
             '~',
             (MIN(N2.num) - 1) AS gap_end
           FROM Numbers N1 INNER JOIN Numbers N2
             ON N2.num > N2.num
           GROUP BY N1.num
         HAVING (N1.num + 1) < MIN(M2.num);
         ```

     - 절차 지향적 방법 - '다음 레코드'와 비교

       - ```mysql
         -- 다음 레코드와 비교
         SELECT num + 1 AS gap_start
                '-',
                (num + diff -1) AS gap end
         FROM (SELECT num,
                      MAX(num)
                        OVER(ORDER BY num
                            ROWS BETWEEN 1 FOLLOWING
                                   AND 1 FOLLOWING - num
               FROM Numbers) TMP(num, diff)
         WHERE diff <> 1;
         ```

     <BR>

     <BR>

     <BR>

3. 시퀀스 객체, IDENTITY 필드 , 채번 테이블

   - **표준 SQL에는 순번을 다루는 기능으로 시퀀스 객체, IDENTITY 필드가 존재하지만 지원하지 않는 데이터베이스도 존재한다**

     - **오라클은 IDENTITY 필드를 지원하지 않는다**

     - **MySQL은 시퀀스 객체를 지원하지 않는다**

     - **고로 두 기능은 최대한 사용하지 않는 것을 이 책에서 추천함**

       <BR>

   - 시퀀스 객체

     - 개념 및 특징
       - 테이블 또는 뷰처럼 스키마 내부에 존재하는 객체 중의 하나를  `시퀀스 객체`라고 한다
       - 만들어진 시퀀스 객체는 SQL 구문 내부에 접근해서 수열을 생성할 수 있다
       - 시퀀스 객체를 가장 많이 사용하는 장소는 INSERT 구문의 내부이다
         - 시퀀스 객체로 만들어진 순번을 기본 키로 사용해서 레코드를 INSERT 한다

     - 시퀀스 객체 SQL

       - ```mysql
         CREATE SEQUENCE testseq
         START WITH 1 -- 초기값
         INCREMENT BY 1 -- 증가값
         MAXVALUE 100000 -- 최대값
         MINVALUE 1 -- 최소값
         CYCLE; -- 최대값에 도달했을 때 순환 유무
         ```

     - 문제점

       - 표준화가 늦어서, 구현에 따라 구문이 달라 이식성이 없고, 사용할 수 없는 구현도 있다
       - 시스템에서 자동으로 생성되는 값이므로 실제 엔터티 속성이 아니다
       - 성능 문제
         - 실제 1, 2번 문제의 경우 실무에서 무시하고 사용하지만 3번은 잘 관리해야 한다
       
     - 시퀀스 객체가 생성하는 순서의 특성

       - 유일성
         - `1,2,2,3,4,5,5,5 // 중복 값이 X`
       - 연속성
         - `1.2.4.5.6.8 // 숫자가 비면 X.`
       - 순서성
         - `1,2,5,4,6,7,8 // 정렬이 되어있음.`

     - **시퀀스 객체의 락(Lock) 메커니즘**

       - **시퀀스 객체에 배타 Lock을 적용**
       - **NEXT VALUE을 검색**
       - **CURRENT VALUE를 1만큼 증가**
       - **시퀀스 객체에 배타 Lock 해제**
         - **동시에 여러 사용자가 시퀀스 객체에 접근하는 경우 락 충돌로 인해 성능 저하 문제가 발생한다. 또한 어떤 사용자가 연속적으로 시퀀스 객체를 사용하는 경우에도 1,4 단계를 반복하므로 오버 헤드가 발생한다.**
       
     - 시퀀스 객체로 발생하는 성능 문제의 대처

       - `CACHE`와 `NO ORDER` 객체
         - `CACHE`는 새로운 값이 필요할 때마다 메모리에 읽어들일 필요가 있는 값의 수를 설정하는 옵션
         - `NOORDER` 은 순서성을 담보하지 않아서 오버 헤드를 줄이는 효과를 지닌 옵션

       <BR>

   - IDENTITY 필드

     - 개념 및 특징

       - `IDENTITY 필드`는 테이블의 필드로 정의하고, 테이블에 INSERT 가 발생할 때마다 자동으로 순번을 붙여주는 기능이다
       - IDENTITY 필드는 '자동 순번 필드'라고 불린다

     - 문제점

       - IDENTITY 필드는 특정한 테이블과 연결되므로 테이블에 종속적이다

       - IDENTITY 필드는 구현에 따라 CACHE, NOORDER를 지정할 수 없어서 제한적/전면적 사용이 불가능하다

       - 걍 장점이 없음 쓰레기임

         <BR>

   - 채번 테이블

     - 개념 및 특징
       - 애플리케이션쪽에서 레코드에 순번을 부여하기 위해 만든 테이블
       - 시퀀스 객체를 테이블로 유사하게 구현한 것으로, 시퀀스 객체의 락 메커니즘을 이용함
         - 락(Lock) 메커니즘은 동시성 실행 제어를 위해 'A'라는 사용자가 시퀀스 객체를 사용하고 있다면 다른 사용자가 시퀀스 객체에 접근하지 못 하게 블록하는 제어를 의미한다

     - 문제점

       - 성능이 제대로 나오지 않아서 현재는 거의 쓰지 않는다

         <BR>

   - **정리**

     - **SQL 초기에 배제했던 절차 지향성이 윈도우 함수라는 형태로 부활**

     - **윈도우 함수를 사용하면 코드를 간결하게 기술할 수 있으므로 가독성 향상**

     - **윈도우 함수는 조인 또는 테이블 접근을 줄이므로 성능 향상을 가져옴**

     - **시퀀스 객체 또는 `IDENTITY` 필드는 성능 문제를 일으키는 원인이 되므로 사용할 때 주의가 필요**

<br>

<br>

<br>

<BR>

<BR>

<BR>


## 9장 갱신과 데이터 모델



0. 이번 장에서는 갱신을 효율적으로 수행하는 SQL을 공부한다

   <br>

   <br>

   <br>

1. 갱신은 효율적으로

   - NULL 채우기

     - ```mysql
       UPDATE OmitTbl
       	SET val = (SELECT val
       		FROM OmitTbl	OT1
       		WHERE OT1.keycol = (SELECT MAX(seq)
       		FROM OmitTbl OT2
       		WHERE OT2.keycol = OmitTbl.keycol
       		AND OT2.seq < OmitTbl.seq
       		AND OT2.val IS NOT NULL))
       WHERE val IS NULL;
       ```

       <br>

   - 반대로 NULL을 작성 

     - ```mysql
       UPDATE OmitTbl
       	SET val = CASE WHEN val
       		= (SELECT val
       		FROM OmitTbl 01
       		WHERE 01.keycol = OmitTbl.keycol
       		AND 01.seq
       		= (SELECT MAX(seq)
       		FROM OmitTbl 02
       		WHERE 02.keycol = OmitTbl.keycol
       		AND 02.seq < OmitTbl.seq))
        	THEN NULL
        	ELSE val END;
       ```

       <br>

       <br>

       <br>

2. 레코드에서 필드로의 갱신
   - 필드를 하나씩 갱신

     - 명확하지만 항목별로 서브쿼리를 필요로하기 때문에 비효율적이다.

     - ```sql
       UPDATE ScoreCols
       	SET score_en = (SELECT score
       		FROM ScoreRow SR
       		WHERE SR.studend_id = ScoreCols.studend_id
       		AND subject = '영어'),
       		score_nl = (SELECT score
       		FROM ScoreRow SR
       		WHERE SR.studend_id = ScoreCols.studend_id
       		AND subject = '국어'),
       		score_mt = (SELECT score
       		FROM ScoreRow SR
       		WHERE SR.studend_id = ScoreCols.studend_id
       		AND subject = '수학');
       ```

       <br>

   - 다중 필드 할당

     - 여러 개의 필드를 리스트화하고 한번에 갱신하기 때문에 성능도 좋고, 코드도 간결하다

     - MySQL에서는 SET구에 리스트를 넣는 기능이 아직 지원하지 않는다 (2014년 기준, 오라클은 지원함)

     - ```sql
       UPDATE ScoreCols
       	SET (score_en, score_nl, score_mt)
       		= (SELECT MAX(CASE WHEN subject = '영어'
       				THEN score
       				ELSE NULL END) AS score_en,
       				MAX(CASE WHEN subject = '국어'
       				THEN score
       				ELSE NULL END) AS score_nl,
       				MAX(CASE WHEN subject = '수학'
       				THEN score
       				ELSE NULL END) AS score_mt
       		FROM ScoreRows SR
       WHERE SR.student_id = ScoreCols.student_id);
       ```

       <br>

   - NOT NULL의 제약이 있는 경우

     - UPDATE 구문 사용

       - 테이블 사이에 일치하지 않는 레코드는 제거

       - ```sql
         UPDATE ScoreColsNN
         	SET score_en = COALESCE((SELECT score
         		FROM ScoreRow SR
         		WHERE SR.studend_id = ScoreCols.studend_id
         		AND subject = '영어'), 0), -- 학생은 존재하지만 과목이 없는 경우, 처리가능
         		score_nl = COALESCE((SELECT score
         		FROM ScoreRow SR
         		WHERE SR.studend_id = ScoreCols.studend_id
         		AND subject = '국어'), 0),
         		score_mt = COALESCE((SELECT score
         		FROM ScoreRow SR
         		WHERE SR.studend_id = ScoreCols.studend_id
         		AND subject = '수학'), 0);
         WHERE EXISTS (SELECT * -- 처음부터 학생이 존재하지 않는 경우, 처리 가능
         			FROM ScoreRows
         			WHERE student_id = ScoreColsNN.studend_id);
         ```

     - MERGE 구문 사용(오라클, DB2 지원)

       - 결합조건을 ON 구 하나에 묶을 수 있다.

       - ScoreRows 테이블 풀 스캔 1회 + 정렬 1회로 고정된다.

       - 아까보다 괜찮은 선택지로 고려해볼만 하다.

       - ```sql
         MERGE INTO ScoreColsNN
         	USING (SELECT student_id,
         		COALESCE(MAX(CASE WHEN subject = '영어'
         				THEN score
         				ELSE NULL END), 0) AS Score_en,
         		COALESCE(MAX(CASE WHEN subject = '국어'
         				THEN score
         				ELSE NULL END), 0) AS Score_nl,
         		COALESCE(MAX(CASE WHEN subject = '수학'
         				THEN score
         				ELSE NULL END), 0) AS Score_mt
         			FROM ScoreRows
         			GROUP By studend_id) SR
         			ON	(ScoreColsNN.student_id = SR.student_id)
         	WHEN MATCHED THEN
         	UPDATE SET ScoreColsNN.score_en = SR.score_en,
         			coreColsNN.score_nl = SR.score_nl,
         			ScoreColsNN.score_mt = SR.score_mt;
         ```

         <br>

         <br>

         <br>

3. 필드에서 레코드로 변경

   - <img src="C:\Users\wooko\Desktop\블로그에 올릴 사진\필드에서레코드로변경.jpg" alt="필드에서레코드로변경" style="zoom:25%;" />

     - ```mysql
       UPDATE ScoreRows
       	SET Score = (SELECT CASE ScoreRows.subject
       		WHEN '영어' THEN score_en
       		WHEN '국어' THEN score_nl
       		WHEN '수학' THEN score_nt
       		ELSE NULL
       		END
       	FROM ScoreCols
       	WHERE student_id = ScoreRow.student_id);
       ```

       <br>

       <br>

       <br>

4. 같은 테이블의 다른 레코드로 갱신
   - 상관 서브쿼리 이용

     - 테이블에 여러번 접근해야 한다.

     - ```sql
       INSERT INTO Stocks2
       SELECT brand, sale_date, price,
       	CASE SIGN(price -
       			(SELECT price
       			FROM Stocks S1
       			WHERE brand = Stocks.brand
       			AND sale_date =
       				(SELECT MAX(sale_date)
       				FROM Stocks S2
       				WHERE brand = Stocks.brand
       				AND sale_date < Stock.sale_date)))
       	WHEN -1 THEN '아래화살표'
       	WHEN 0 THEN '오른쪽화살표'
       	WHEN 1 THEN '위쪽화살표'
       	ELSE NULL
       	END
       FROM Stocks S2;
       ```

       <BR>

   - 윈도우 함수 사용

     - 매우 간단하고 효율적이 된다.

     - ```sql
       INSERT INTO Stocks2
       SELECT brand, sale_date, price,
       	CASE SIGN(price -
       		MAX(price) OVER (PARTITION BY brand
       			ORDER BY sale_date
       			ROWS BETWEEN 1 PRECEDING
       			AND  1 PRECEDING))
       	WHEN -1 THEN '아래화살표'
       	WHEN 0 THEN '오른쪽화살표'
       	WHEN 1 THEN '위쪽화살표'
       	ELSE NULL
       	END
       FROM Stocks S2;
       ```

       <BR>

   - INSERT-SELECT와 UPDATE 어떤 것이 좋을까?

     - INSERT-SELECT 장점

       - 고속 처리 가능
       - MySQL처럼 자기참조를 허가하지 않는 DB에서도 사용가능

     - INSERT-SELECT 단점

       - 같은 크기와 구조를 가진 데이터를 두 개 만들어야하므로 저장소를 2배 이상 소비
         - 하지만 최근에 저장소 가격이 낮아진 걸 생각하면 나쁘지 않은 단점

     - ~~**정리**~~

       - ~~**성능과 동기성의 트레이드 오프를 잘 생각해서 결정해보자**~~

         <br>

         <br>

         <br>

5. 갱신이 초래하는 트레이드오프

   - 문제 상황

     - 주문일(order_date)과 배송 예정일(delivery_date) 차이가 3일 이상 차이난다면 주문자에게 연락을 하고싶다 이런 경우 어떻게 해결할까?

       <br>

   - SQL을 사용하는 방법

     - ```sql
       -- 주문일과 배송 예정일의 차이
       SELECT O.order_id,
       	O.order_name,
       	ORC.delivery_date - O.order_date AS diff_days
       FROM Orders O
       		INNER JOIN OrderReceipts ORC
       		ON O.order_id = ORC.order_id
       WHERE ORC.delivery_date - O.order_date >= 3;
       ```

       <br>

   - 모델 갱신을 사용하는 방법

     - `SQL을 사용하는 방법`은 결과로는 동일하지만 효율적인 방법은 아님

       - sql에 의지하지 않고 `모델 갱신`을 통해 해결할 수 있다
       - 쿼리를 통해 찾는게 아니라 필드를 하나 추가해 해결할 수도 있을 것

     - Orders 테이블에 배송 지연 플래그를 추가해서 검색쿼리는 해당 플래그 필드만을 검색하면 됨

       - <img src="C:\Users\wooko\AppData\Roaming\Typora\typora-user-images\image-20210908131037232.png" alt="image-20210908131037232" style="zoom:25%;" />

         <br>

         <br>

         <br>

6. 모델 갱신의 주의점
   - 높아지는 갱신 비용

     - 검색 부하를 갱신 부하로 미루는 꼴이 될 수 있음(플래그 갱신 때문에)

       <br>

   - 갱신까지의 시간 랙(Time Rag) 발생

     - `데이터의 실시간성`이라는 문제 발생 

     - 예를 들어, Orders 테이블에 배송 지연 플래그와 다른 테이블의 배송 예정일 필드가 실시간으로 동기화되지 않는다면 차이 발생

       <br>

   - 모델 갱신 비용 발생

     - RDB의 데이터 모델 갱신은 대대적인 수정이 요구됨

     - 프로젝트 마지막 단계에서 모델을 변경하는 것은 프로그램의 품질과 개발 일정에 큰 위험이 된다

       <br>

       <br>

       <br>

7. 데이터 모델을 지배하는 자가 시스템을 지배한다(총정리)

   - 현명한 데이터 구조와 멍청한 코드의 조합이 멍청한 데이터 구조와 현명한 코드의 조합보다 좋다.
   - 엔지니어의 사명은 전략적 실패를 만회하는 전술을 찾는 것이 아닌 올바른 전략을 고려하는 것





질문사항

1. auto increment와 indentity 필드의 차이