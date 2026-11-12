# SQL 연습 문제 노트

공개 연습 문제를 유형·풀이·함정 단위로 정리한다. 문제 원문은 옮기지 않는다. 실제 응시 문제는 적지 않는다.

## 전체 최댓값과 각 행의 차이 (프로그래머스 사전 테스트 예시, ECOLI_DATA)

유형: 집계값을 모든 행에 붙이기 (스칼라 서브쿼리 또는 빈 OVER)

요구
- 출력 컬럼 `ID`, `DEV`
- DEV = 전체에서 가장 큰 크기 - 각 행의 크기
- DEV 오름차순, 같으면 ID 오름차순

풀이 1: 스칼라 서브쿼리

```sql
SELECT ID,
       (SELECT MAX(SIZE_OF_COLONY) FROM ECOLI_DATA) - SIZE_OF_COLONY AS DEV
FROM ECOLI_DATA
ORDER BY DEV, ID;
```

풀이 2: 윈도우 함수 (MySQL 8 이상)

```sql
SELECT ID,
       MAX(SIZE_OF_COLONY) OVER () - SIZE_OF_COLONY AS DEV
FROM ECOLI_DATA
ORDER BY DEV, ID;
```

핵심
- `SELECT ID, MAX(SIZE_OF_COLONY) - SIZE_OF_COLONY`처럼 집계 함수와 일반 컬럼을 그냥 섞으면 오류다. 집계는 여러 행을 한 행으로 줄이기 때문
- 행 수를 유지한 채 집계값을 붙이는 방법은 둘: 값 하나를 돌려주는 서브쿼리, 또는 `OVER ()`
- `OVER ()`는 괄호가 비어 있으면 전체 행이 한 묶음. `PARTITION BY`를 넣으면 그룹별 최댓값과의 차이로 바뀐다
- ORDER BY에서는 SELECT 별칭(`DEV`)을 쓸 수 있다

함정
- 2차 정렬(ID)을 빼먹는다
- 별칭을 문제의 컬럼명과 다르게 쓴다
- 문제 설명에 나온 컬럼(PARENT_ID, GENOTYPE 등)을 다 쓰려 한다. 필요한 컬럼만 본다

같은 형태로 푸는 문제
- 평균 대비 차이: `v - AVG(v) OVER ()`
- 전체 대비 비중: `v / SUM(v) OVER ()`
- 그룹 내 최댓값과의 차이: `MAX(v) OVER (PARTITION BY g) - v`

## 같은 원리 자작 문제 (집계값을 행에 붙이기, 윈도우 함수)

테이블 `ROOM_BOOKING(BOOKING_ID, REGION, PRICE)`

```sql
CREATE TABLE ROOM_BOOKING (BOOKING_ID INT, REGION VARCHAR(10), PRICE INT);
INSERT INTO ROOM_BOOKING VALUES
  (1,'서울',120000),(2,'서울',80000),(3,'부산',100000),
  (4,'부산',100000),(5,'제주',200000),(6,'서울',120000);
```

문제
1. 각 예약의 금액과 전체 평균 금액의 차이 DIFF(소수 첫째 자리). DIFF 내림차순, 같으면 BOOKING_ID 오름차순
2. 같은 지역의 최고 금액과의 차이 GAP. REGION, GAP, BOOKING_ID 오름차순
3. 전체 매출에서 각 예약이 차지하는 비중 SHARE_PCT(%, 소수 둘째 자리). SHARE_PCT 내림차순, 같으면 BOOKING_ID 오름차순
4. 지역별 금액 상위 2위까지(동률 모두 포함). REGION 오름차순, PRICE 내림차순, BOOKING_ID 오름차순
5. 지역별 매출 SALES와 전체 대비 비중 SHARE_PCT(%, 소수 첫째 자리). SALES 내림차순

정답

```sql
-- 1
SELECT BOOKING_ID,
       ROUND(PRICE - (SELECT AVG(PRICE) FROM ROOM_BOOKING), 1) AS DIFF
FROM ROOM_BOOKING
ORDER BY DIFF DESC, BOOKING_ID;

-- 2
SELECT BOOKING_ID, REGION,
       MAX(PRICE) OVER (PARTITION BY REGION) - PRICE AS GAP
FROM ROOM_BOOKING
ORDER BY REGION, GAP, BOOKING_ID;

-- 3
SELECT BOOKING_ID,
       ROUND(PRICE / SUM(PRICE) OVER () * 100, 2) AS SHARE_PCT
FROM ROOM_BOOKING
ORDER BY SHARE_PCT DESC, BOOKING_ID;

-- 4
WITH ranked AS (
  SELECT BOOKING_ID, REGION, PRICE,
         DENSE_RANK() OVER (PARTITION BY REGION ORDER BY PRICE DESC) AS rnk
  FROM ROOM_BOOKING
)
SELECT REGION, BOOKING_ID, PRICE
FROM ranked
WHERE rnk <= 2
ORDER BY REGION, PRICE DESC, BOOKING_ID;

-- 5
SELECT REGION,
       SUM(PRICE) AS SALES,
       ROUND(SUM(PRICE) / SUM(SUM(PRICE)) OVER () * 100, 1) AS SHARE_PCT
FROM ROOM_BOOKING
GROUP BY REGION
ORDER BY SALES DESC;
```

기대 결과
- 1: (5, 80000.0), (1, 0.0), (6, 0.0), (3, -20000.0), (4, -20000.0), (2, -40000.0)
- 2: 부산 3·4번 0, 서울 1·6번 0, 서울 2번 40000, 제주 5번 0
- 3: (5, 27.78), (1, 16.67), (6, 16.67), (3, 13.89), (4, 13.89), (2, 11.11)
- 4: 부산 3·4번, 서울 1·6·2번, 제주 5번
- 5: 서울 320000 44.4, 부산 200000 27.8, 제주 200000 27.8

검증: 기대 결과는 SQLite에서 같은 논리의 쿼리로 확인했다(나눗셈만 `* 100.0`으로 바꿈). MySQL에서는 실행하지 못했다.

## 자작 문제: JOIN과 날짜 (숙박 예약)

테이블
- `HOTEL(HOTEL_ID, HOTEL_NAME, REGION, GRADE)`
- `USERS(USER_ID, JOIN_DATE)`
- `BOOKING(BOOKING_ID, HOTEL_ID, USER_ID, CHECKIN_DATE, CHECKOUT_DATE, PRICE, STATUS, BOOKED_AT)`: STATUS는 PAID 또는 CANCELLED. 체크아웃 날짜의 밤은 숙박하지 않는다
- `REVIEW(REVIEW_ID, BOOKING_ID, SCORE)`: 예약 1건에 리뷰는 최대 1개

```sql
CREATE TABLE HOTEL (HOTEL_ID INT, HOTEL_NAME VARCHAR(20), REGION VARCHAR(10), GRADE INT);
INSERT INTO HOTEL VALUES
  (1,'서울스테이','서울',4),(2,'한강호텔','서울',5),(3,'해운대뷰','부산',4),
  (4,'광안리인','부산',3),(5,'제주오션','제주',5),(6,'한라리조트','제주',4);

CREATE TABLE USERS (USER_ID INT, JOIN_DATE DATE);
INSERT INTO USERS VALUES
  (1,'2026-08-20'),(2,'2026-09-01'),(3,'2026-09-10'),(4,'2026-09-28'),(5,'2026-10-05');

CREATE TABLE BOOKING (BOOKING_ID INT, HOTEL_ID INT, USER_ID INT, CHECKIN_DATE DATE,
                      CHECKOUT_DATE DATE, PRICE INT, STATUS VARCHAR(10), BOOKED_AT DATETIME);
INSERT INTO BOOKING VALUES
  (1,1,1,'2026-09-05','2026-09-07',240000,'PAID','2026-08-25 10:00:00'),
  (2,2,2,'2026-09-12','2026-09-13',300000,'PAID','2026-09-03 21:30:00'),
  (3,3,1,'2026-09-28','2026-10-02',400000,'PAID','2026-09-20 09:15:00'),
  (4,5,3,'2026-10-03','2026-10-06',600000,'PAID','2026-09-30 23:50:00'),
  (5,1,4,'2026-10-06','2026-10-08',220000,'CANCELLED','2026-10-01 12:00:00'),
  (6,4,2,'2026-10-10','2026-10-11',90000,'PAID','2026-10-02 08:00:00'),
  (7,3,4,'2026-10-07','2026-10-09',200000,'PAID','2026-10-04 19:00:00'),
  (8,2,5,'2026-11-01','2026-11-03',620000,'PAID','2026-10-20 14:00:00'),
  (9,5,1,'2026-11-14','2026-11-16',420000,'PAID','2026-11-02 11:00:00'),
  (10,1,3,'2026-11-20','2026-11-21',110000,'PAID','2026-11-05 16:40:00');

CREATE TABLE REVIEW (REVIEW_ID INT, BOOKING_ID INT, SCORE INT);
INSERT INTO REVIEW VALUES (1,1,5),(2,2,4),(3,3,3),(4,4,5),(5,6,2),(6,7,4);
```

문제 (JOIN)
- J1. 지역별 PAID 예약 건수 CNT와 매출 SALES. SALES 내림차순, 같으면 REGION 오름차순
- J2. 예약이 한 건도 없는 호텔의 HOTEL_ID, HOTEL_NAME (상태 무관). HOTEL_ID 오름차순
- J3. 모든 호텔의 PAID 예약 건수 BOOKING_CNT(없으면 0)와 평균 평점 AVG_SCORE(소수 첫째 자리, 리뷰가 없으면 NULL). BOOKING_CNT 내림차순, 같으면 HOTEL_ID 오름차순
- J4. 리뷰가 없는 PAID 예약의 BOOKING_ID, HOTEL_NAME. BOOKING_ID 오름차순

문제 (날짜)
- D1. 예약 시각(BOOKED_AT) 기준 월별 PAID 건수 CNT와 매출 SALES. 월은 `2026-09` 형식의 YM. YM 오름차순
- D2. 지역별 PAID 예약의 평균 숙박 일수 AVG_NIGHTS(소수 첫째 자리). REGION 오름차순
- D3. 가입일로부터 7일 이내에 첫 PAID 예약을 한 유저 수 USER_CNT와 전체 유저 대비 비율 RATE_PCT(%, 소수 첫째 자리). 예약일은 BOOKED_AT의 날짜
- D4. 2026-10-01부터 2026-10-07까지의 밤 중 하루라도 숙박하는 PAID 예약의 BOOKING_ID, CHECKIN_DATE, CHECKOUT_DATE. BOOKING_ID 오름차순
- D5. D1의 월별 매출에 전월 대비 증감률 GROWTH_PCT(%, 소수 첫째 자리)를 붙인다. 첫 달은 NULL. YM 오름차순

정답

```sql
-- J1
SELECT h.REGION, COUNT(*) AS CNT, SUM(b.PRICE) AS SALES
FROM BOOKING b
JOIN HOTEL h ON b.HOTEL_ID = h.HOTEL_ID
WHERE b.STATUS = 'PAID'
GROUP BY h.REGION
ORDER BY SALES DESC, h.REGION;

-- J2
SELECT h.HOTEL_ID, h.HOTEL_NAME
FROM HOTEL h
LEFT JOIN BOOKING b ON h.HOTEL_ID = b.HOTEL_ID
WHERE b.BOOKING_ID IS NULL
ORDER BY h.HOTEL_ID;

-- J3
SELECT h.HOTEL_ID, h.HOTEL_NAME,
       COUNT(b.BOOKING_ID)    AS BOOKING_CNT,
       ROUND(AVG(r.SCORE), 1) AS AVG_SCORE
FROM HOTEL h
LEFT JOIN BOOKING b ON h.HOTEL_ID = b.HOTEL_ID AND b.STATUS = 'PAID'
LEFT JOIN REVIEW r  ON b.BOOKING_ID = r.BOOKING_ID
GROUP BY h.HOTEL_ID, h.HOTEL_NAME
ORDER BY BOOKING_CNT DESC, h.HOTEL_ID;

-- J4
SELECT b.BOOKING_ID, h.HOTEL_NAME
FROM BOOKING b
JOIN HOTEL h       ON b.HOTEL_ID = h.HOTEL_ID
LEFT JOIN REVIEW r ON b.BOOKING_ID = r.BOOKING_ID
WHERE b.STATUS = 'PAID' AND r.REVIEW_ID IS NULL
ORDER BY b.BOOKING_ID;

-- D1
SELECT DATE_FORMAT(BOOKED_AT, '%Y-%m') AS YM, COUNT(*) AS CNT, SUM(PRICE) AS SALES
FROM BOOKING
WHERE STATUS = 'PAID'
GROUP BY YM
ORDER BY YM;

-- D2
SELECT h.REGION,
       ROUND(AVG(DATEDIFF(b.CHECKOUT_DATE, b.CHECKIN_DATE)), 1) AS AVG_NIGHTS
FROM BOOKING b
JOIN HOTEL h ON b.HOTEL_ID = h.HOTEL_ID
WHERE b.STATUS = 'PAID'
GROUP BY h.REGION
ORDER BY h.REGION;

-- D3
WITH first_booking AS (
  SELECT USER_ID, MIN(DATE(BOOKED_AT)) AS first_dt
  FROM BOOKING
  WHERE STATUS = 'PAID'
  GROUP BY USER_ID
)
SELECT SUM(DATEDIFF(f.first_dt, u.JOIN_DATE) <= 7) AS USER_CNT,
       ROUND(SUM(DATEDIFF(f.first_dt, u.JOIN_DATE) <= 7) / COUNT(*) * 100, 1) AS RATE_PCT
FROM USERS u
LEFT JOIN first_booking f ON u.USER_ID = f.USER_ID;

-- D4
SELECT BOOKING_ID, CHECKIN_DATE, CHECKOUT_DATE
FROM BOOKING
WHERE STATUS = 'PAID'
  AND CHECKIN_DATE <= '2026-10-07'
  AND CHECKOUT_DATE > '2026-10-01'
ORDER BY BOOKING_ID;

-- D5
WITH m AS (
  SELECT DATE_FORMAT(BOOKED_AT, '%Y-%m') AS YM, SUM(PRICE) AS SALES
  FROM BOOKING
  WHERE STATUS = 'PAID'
  GROUP BY YM
)
SELECT YM, SALES,
       ROUND((SALES - LAG(SALES) OVER (ORDER BY YM)) / LAG(SALES) OVER (ORDER BY YM) * 100, 1) AS GROWTH_PCT
FROM m
ORDER BY YM;
```

기대 결과
- J1: 서울 4건 1270000, 제주 2건 1020000, 부산 3건 690000
- J2: 6 한라리조트
- J3: (1, 2, 5.0), (2, 2, 4.0), (3, 2, 3.5), (5, 2, 5.0), (4, 1, 2.0), (6, 0, NULL)
- J4: 8 한강호텔, 9 제주오션, 10 서울스테이
- D1: 2026-08 1건 240000, 2026-09 3건 1300000, 2026-10 3건 910000, 2026-11 2건 530000
- D2: 부산 2.3, 서울 1.5, 제주 2.5
- D3: 3명, 60.0
- D4: 3번, 4번, 7번
- D5: 2026-08 NULL, 2026-09 441.7, 2026-10 -30.0, 2026-11 -41.8

풀이 포인트
- J2·J4: 없는 것 찾기는 LEFT JOIN 뒤 오른쪽 키가 IS NULL
- J3: PAID 조건을 WHERE에 쓰면 예약 없는 호텔이 사라진다. ON에 둔다. `COUNT(*)`는 예약이 없어도 1이므로 `COUNT(b.BOOKING_ID)`
- D2: 숙박 일수는 `DATEDIFF(체크아웃, 체크인)`. 체크아웃 날은 숙박하지 않으므로 +1 하지 않는다
- D3: 첫 예약은 유저별 MIN으로 먼저 구한다. 분모는 예약이 없는 유저도 포함해야 하므로 USERS 기준 LEFT JOIN
- D4: 기간 겹침은 `시작 <= 기간끝 AND 끝 > 기간시작`. 체크아웃 날을 포함하는 문제라면 `>=`
- D5: 윈도우 함수는 집계가 끝난 CTE 위에서 쓴다

검증: 기대 결과는 SQLite에서 같은 논리로 확인했다(날짜 함수와 나눗셈은 SQLite 문법으로 바꿈). MySQL에서는 실행하지 못했다.
