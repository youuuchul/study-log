# MySQL 치트시트 (코딩테스트용)

참고 자료 없이 쓸 수 있어야 하는 문법. MySQL 8 기준. 윈도우 함수와 CTE는 8.0부터 지원하니 시험 환경의 버전을 사전 테스트에서 `SELECT VERSION();`으로 확인한다.

## 1. 실행 순서와 기본 골격

```sql
SELECT   컬럼, 집계
FROM     테이블
JOIN     테이블 ON 조건
WHERE    행 조건
GROUP BY 그룹
HAVING   집계 조건
ORDER BY 정렬
LIMIT    n OFFSET m;
```

- 실행 순서: FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
- WHERE에는 집계 함수와 윈도우 함수를 쓸 수 없다. 집계 조건은 HAVING, 윈도우 결과 조건은 서브쿼리·CTE로 한 번 감싼다
- MySQL은 GROUP BY, HAVING, ORDER BY에서 SELECT 별칭을 쓸 수 있다. WHERE에서는 못 쓴다

## 2. 집계

```sql
COUNT(*)              -- 행 수 (NULL 포함)
COUNT(col)            -- NULL 제외
COUNT(DISTINCT col)   -- 중복 제거
SUM(col), AVG(col), MAX(col), MIN(col)

SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END)   -- 조건부 개수
SUM(status = 'paid')                               -- MySQL은 참이 1
AVG(status = 'paid')                               -- 비율
COUNT(DISTINCT CASE WHEN cond THEN user_id END)    -- 조건부 고유 수
```

- AVG, SUM은 NULL을 건너뛴다. 0으로 세야 하면 `AVG(IFNULL(col, 0))`
- MySQL의 `/`는 정수끼리 나눠도 소수가 나온다. 정수 나눗셈은 `DIV`, 나머지는 `MOD(a, b)` 또는 `%`
- 0으로 나누면 NULL. 필요하면 `NULLIF(분모, 0)`으로 명시
- 반올림 `ROUND(x, 1)`, 버림 `TRUNCATE(x, 1)`, 올림 `CEIL(x)`, 내림 `FLOOR(x)`

## 3. NULL과 조건

```sql
WHERE col IS NULL / IS NOT NULL      -- = NULL 은 항상 거짓
IFNULL(col, '없음')
COALESCE(a, b, c)                    -- 첫 번째 NULL 아닌 값
NULLIF(a, b)                         -- 같으면 NULL
IF(cond, 참, 거짓)
CASE WHEN a THEN x WHEN b THEN y ELSE z END
```

- `NOT IN (서브쿼리)`는 서브쿼리 결과에 NULL이 하나라도 있으면 전부 걸러진다. `NOT EXISTS`나 LEFT JOIN + IS NULL을 쓴다
- `BETWEEN a AND b`는 양 끝 포함
- `col <> 'x'`는 NULL 행을 포함하지 않는다

## 4. JOIN과 집합

```sql
-- 한쪽에만 있는 행
SELECT a.*
FROM a LEFT JOIN b ON a.id = b.a_id
WHERE b.a_id IS NULL;

-- 존재 여부
WHERE EXISTS (SELECT 1 FROM b WHERE b.a_id = a.id)

-- 셀프 조인 (연속일, 상하 관계)
FROM t AS cur JOIN t AS prev
  ON cur.user_id = prev.user_id
 AND cur.dt = DATE_ADD(prev.dt, INTERVAL 1 DAY)
```

- LEFT JOIN 뒤 오른쪽 테이블 조건을 WHERE에 쓰면 INNER JOIN이 된다. 조건은 ON에 둔다
- 1:N 조인 뒤 SUM·COUNT는 값이 불어난다. 먼저 집계한 서브쿼리를 조인하거나 COUNT(DISTINCT)를 쓴다
- `UNION`은 중복 제거, `UNION ALL`은 그대로 합침
- MySQL에는 FULL OUTER JOIN이 없다. LEFT JOIN과 RIGHT JOIN을 UNION으로 합친다

## 5. 날짜

```sql
DATE_FORMAT(dt, '%Y-%m-%d')          -- 2026-11-17
DATE_FORMAT(dt, '%Y-%m')             -- 월 단위 그룹
YEAR(dt), MONTH(dt), DAY(dt), HOUR(dt)
DATE(dt)                             -- 시간 제거
DAYOFWEEK(dt)                        -- 1=일요일 ~ 7=토요일
WEEKDAY(dt)                          -- 0=월요일 ~ 6=일요일

DATEDIFF(끝, 시작)                   -- 일수. 끝 - 시작
TIMESTAMPDIFF(MONTH, 시작, 끝)       -- 단위 지정. 시작이 먼저
DATE_ADD(dt, INTERVAL 7 DAY)
DATE_SUB(dt, INTERVAL 1 MONTH)
LAST_DAY(dt)                         -- 그 달의 마지막 날
CURDATE(), NOW()
```

- 형식 문자: `%Y` 4자리 연도, `%m` 2자리 월, `%d` 2자리 일, `%H` 24시, `%i` 분, `%s` 초. `%M`은 월 이름, `%D`는 서수라서 다르다
- DATEDIFF와 TIMESTAMPDIFF는 인자 순서가 반대다
- 대여 기간처럼 양 끝을 포함하면 `DATEDIFF(end, start) + 1`
- 기간 필터: `dt >= '2026-10-01' AND dt < '2026-11-01'`. DATETIME에 BETWEEN을 쓰면 마지막 날의 00:00:00 이후가 빠진다
- 특정 기간과 겹치는 구간: `start <= 기간끝 AND end >= 기간시작`

## 6. 문자열

```sql
CONCAT(a, '-', b)
CONCAT_WS('-', a, b, c)
SUBSTRING(s, 1, 3)        -- 1부터 시작
LEFT(s, 2), RIGHT(s, 2)
LENGTH(s)                 -- 바이트. 한글은 CHAR_LENGTH(s)
UPPER(s), LOWER(s), TRIM(s)
REPLACE(s, 'a', 'b')
LIKE '%키워드%'  /  LIKE '김_'
s REGEXP '^A|B$'
GROUP_CONCAT(col ORDER BY col SEPARATOR ',')
```

## 7. 윈도우 함수와 CTE

```sql
ROW_NUMBER() OVER (PARTITION BY g ORDER BY v DESC)   -- 1,2,3,4
RANK()       OVER (...)                              -- 1,2,2,4
DENSE_RANK() OVER (...)                              -- 1,2,2,3
LAG(v, 1)  OVER (PARTITION BY g ORDER BY dt)         -- 이전 행
LEAD(v, 1) OVER (PARTITION BY g ORDER BY dt)         -- 다음 행
SUM(v) OVER (PARTITION BY g ORDER BY dt)             -- 누적 합
SUM(v) OVER (PARTITION BY g)                         -- 그룹 합계 (비중 계산)
AVG(v) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- 7일 이동 평균
NTILE(4) OVER (ORDER BY v)                           -- 4분위
PERCENT_RANK() OVER (ORDER BY v)
FIRST_VALUE(v) OVER (PARTITION BY g ORDER BY dt)
```

```sql
WITH ranked AS (
  SELECT g, item, v,
         ROW_NUMBER() OVER (PARTITION BY g ORDER BY v DESC) AS rn
  FROM t
)
SELECT g, item, v
FROM ranked
WHERE rn <= 3;
```

- 그룹별 상위 N개는 위 형태가 기본. 동률을 모두 포함하라면 RANK나 DENSE_RANK
- CTE는 쉼표로 이어 쓴다: `WITH a AS (...), b AS (...) SELECT ...`
- ORDER BY가 있는 SUM OVER는 누적, 없으면 파티션 전체 합
- 윈도우 함수가 없는 버전이면 그룹별 최댓값은 서브쿼리로 푼다

```sql
SELECT *
FROM t
WHERE (g, v) IN (SELECT g, MAX(v) FROM t GROUP BY g);
```

재귀 CTE (0~23시처럼 빈 구간을 채울 때)

```sql
WITH RECURSIVE h AS (
  SELECT 0 AS hr
  UNION ALL
  SELECT hr + 1 FROM h WHERE hr < 23
)
SELECT h.hr, COUNT(t.id) AS cnt
FROM h LEFT JOIN t ON HOUR(t.dt) = h.hr
GROUP BY h.hr
ORDER BY h.hr;
```

## 8. 분석 패턴

전월 대비 증감

```sql
WITH m AS (
  SELECT DATE_FORMAT(order_dt, '%Y-%m') AS ym, SUM(amount) AS amt
  FROM orders
  GROUP BY ym
)
SELECT ym, amt,
       ROUND((amt - LAG(amt) OVER (ORDER BY ym)) / LAG(amt) OVER (ORDER BY ym) * 100, 1) AS growth_pct
FROM m;
```

재구매 고객

```sql
SELECT user_id
FROM orders
GROUP BY user_id
HAVING COUNT(*) >= 2;
```

월 코호트 리텐션

```sql
WITH first_m AS (
  SELECT user_id, MIN(DATE_FORMAT(order_dt, '%Y-%m-01')) AS cohort
  FROM orders
  GROUP BY user_id
),
act AS (
  SELECT DISTINCT o.user_id, f.cohort,
         TIMESTAMPDIFF(MONTH, f.cohort, DATE_FORMAT(o.order_dt, '%Y-%m-01')) AS n
  FROM orders o JOIN first_m f ON o.user_id = f.user_id
)
SELECT cohort, n,
       COUNT(*) AS users,
       ROUND(COUNT(*) / FIRST_VALUE(COUNT(*)) OVER (PARTITION BY cohort ORDER BY n) * 100, 1) AS retention_pct
FROM act
GROUP BY cohort, n
ORDER BY cohort, n;
```

퍼널 전환율

```sql
SELECT
  COUNT(DISTINCT CASE WHEN event = 'view' THEN user_id END)     AS view_users,
  COUNT(DISTINCT CASE WHEN event = 'cart' THEN user_id END)     AS cart_users,
  COUNT(DISTINCT CASE WHEN event = 'purchase' THEN user_id END) AS purchase_users,
  ROUND(COUNT(DISTINCT CASE WHEN event = 'purchase' THEN user_id END)
      / COUNT(DISTINCT CASE WHEN event = 'view' THEN user_id END) * 100, 1) AS cvr_pct
FROM events;
```

- 단계 순서를 지켜야 하는 퍼널은 사용자별 단계 최초 시각을 구한 뒤 `view_at <= cart_at <= purchase_at`을 확인한다

연속 N일 (gaps and islands)

```sql
WITH d AS (
  SELECT DISTINCT user_id, DATE(login_dt) AS dt FROM logins
),
g AS (
  SELECT user_id, dt,
         DATE_SUB(dt, INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY dt) DAY) AS grp
  FROM d
)
SELECT user_id, MIN(dt) AS start_dt, COUNT(*) AS days
FROM g
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

구간 분류

```sql
SELECT CASE WHEN price < 10000 THEN '1만 미만'
            WHEN price < 30000 THEN '1만~3만'
            ELSE '3만 이상' END AS price_band,
       COUNT(*) AS cnt
FROM products
GROUP BY price_band;

-- 만 원 단위 구간
SELECT FLOOR(price / 10000) * 10000 AS price_group, COUNT(*) AS cnt
FROM products
GROUP BY price_group
ORDER BY price_group;
```

비중

```sql
SELECT category, SUM(amount) AS amt,
       ROUND(SUM(amount) / SUM(SUM(amount)) OVER () * 100, 1) AS share_pct
FROM orders
GROUP BY category;
```

## 9. MySQL 8.x에서 주의할 것

- GROUP BY가 엄격하다. SELECT의 컬럼은 집계 함수 안에 있거나 GROUP BY에 있어야 한다
- 예약어는 별칭으로 못 쓴다: `rank`, `row_number`, `dense_rank`, `lag`, `lead`, `groups`. `AS rnk`처럼 바꾸거나 백틱으로 감싼다
- `QUALIFY`가 없다. 윈도우 결과로 거르려면 CTE나 서브쿼리로 감싼 뒤 WHERE
- 집계값을 모든 행에 붙일 때는 스칼라 서브쿼리 또는 `MAX(v) OVER ()`. 예시는 `practice-notes.md`

## 10. 자주 틀리는 곳

- GROUP BY에 없는 컬럼을 SELECT에 넣으면 버전·설정에 따라 오류이거나 임의의 값이 나온다
- 그룹별 최댓값의 "행 전체"가 필요할 때 `SELECT name, MAX(price)`로 쓰면 name은 최댓값 행의 것이 아니다
- ORDER BY 동률 처리 지시를 놓친다. 2차 정렬 컬럼 확인
- 문자열로 된 숫자를 정렬하면 사전순이 된다. `ORDER BY CAST(col AS UNSIGNED)`
- DATE 컬럼을 그대로 출력하면 시간까지 나오는 환경이 있다. 문제 예시 형식에 맞춰 DATE_FORMAT
- `COUNT(*)`와 `COUNT(col)`의 차이 (NULL)
- LEFT JOIN 후 `COUNT(*)`는 매칭이 없어도 1이다. `COUNT(오른쪽.id)`를 쓴다
