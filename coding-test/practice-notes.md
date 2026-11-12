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
