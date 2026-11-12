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
