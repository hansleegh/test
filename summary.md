## 표준화 

## ERD PK

  ![alt text](acc_erd.png)

## ERD 모델링

  ![alt text](cvs_erd.png)

## 반정규화(De-normalization) 개념 및 전략
  - 개념 요약
    - 정규형 유지 + 조회 성능/비용 최적화를 위해 반복되는 고비용 조인 결과를 미리 펼쳐 저장(lookup flatten)하거나 집계값을 선계산(precompute)하는 기법
    - 장점: 조인 제거/축소 → 지연↓, 비용↓, 단순쿼리화
    - 단점: 중복↑, 갱신 복잡도↑(정합성 관리 필요)
  - 언제 쓰나
    - 대시보드/검색 API에서 n-way 조인 다빈도 + 엄격한 SLA(지연 한자리 ms~수십 ms)
    - 차원 테이블이 저변경(SCD/코드성)이고 참조 속성 적음
    - 카디널리티/조인 팬아웃으로 풀스캔·해시조인 비용이 큰 경우
    - 동일 패턴 질의가 반복되고 캐시 적중률이 낮음
    - 스토리지 비용 < 쿼리 비용인 환경(서버리스/쿼리당 과금 등)
  - 반정규화 전략 Top 6
    - Lookup Flatten: 자주 조인하는 차원 속성(이름/카테고리/코드명) 복제 컬럼으로 보관
    - Precomputed Aggregates: 일/시간/분 단위 집계 테이블(rollup, snapshot)
    - Materialized View(MV): 조인·집계 결과를 **MV+리프레시(주기/증분)**로 유지
    - Wide Table/JSON 컬럼: 읽기 중심 서비스에 속성 펼침(스키마-온-리드 비용 절감)
    - CDC 기반 증분 갱신: 소스 변경 캡처 → 타겟 반정규 테이블 부분 갱신
    - Covering Index 대안(시험 언급용): 완전한 반정규화는 아니지만, 조인 없는 커버링(INCLUDE/INVISIBLE 컬럼) > INCLUDE/INVISIBLE/생성열을 활용한 Covering Index는 정규형을 유지하면서도 테이블 접근을 제거해 조회 지연과 비용을 대폭 절감하는 반정규화의 실무적 대안 

    ![alt text](denorm_process.png)

## 이력관리 비교
  | 구분                   | **선분이력 구조 (Lineage/Period)**             | **SCD Type 2 (Slowly Changing Dimension)**           |
  | -------------------- | ---------------------------------------- | ---------------------------------------------------- |
  | **주요 목적**            | **기간 기반 이력 관리** (언제부터\~언제까지 유효했는가?)      | 변경 발생 시 **이전 값 보존 + 새 행 추가**                         |
  | **키(PK) 설계**         | `업무키(Business Key) + 유효시작일(VALID_FROM)`  | **대체키(Surrogate Key)** 주로 사용, 비즈니스 키 + 버전/플래그        |
  | **기간 컬럼**            | `VALID_FROM`, `VALID_TO` (NULL=현재)       | 보통 `START_DATE`, `END_DATE`, `IS_CURRENT` 컬럼         |
  | **구간 무겹침 제약**        | 중요! (CHECK/Trigger로 보장)                  | 보통 **버전 번호**나 `IS_CURRENT` 컬럼으로만 관리                  |
  | **현재 행 조회**          | `VALID_TO IS NULL`                       | `IS_CURRENT = 'Y'`                                   |
  | **Point-in-time 조회** | 날짜 비교(`BETWEEN VALID_FROM AND VALID_TO`) | 날짜 또는 버전 번호 기반 필터링                                   |
  | **갱신 방식**            | 기존 행 `VALID_TO` 갱신 후 **새 행 INSERT**      | 기존 행 `IS_CURRENT=N`, 새 행 `IS_CURRENT=Y` INSERT       |
  | **장점**               | - 구간 단위 정확 관리<br>- 회계/보험/계약 등 기간 로직 최적   | - 단순 구조<br>- BI/DW 툴 친화적<br>- surrogate key join 최적화 |
  | **단점**               | - PK/인덱스 복잡<br>- 구간 겹침 관리 필요             | - 기간 연산 지원 약함<br>- Point-in-time 조회 성능 낮을 수 있음       |
  | **사용 사례**            | 계약이력, 인사 배치 이력, 보험 상품 변경                 | 고객 주소 변경, 상품 카테고리 이력, 마스터 데이터 변경                     |

## 데이터품질 정의
  | 영역                     | 설명                       | 예시                  |
  | ---------------------- | ------------------------ | ------------------- |
  | **정합성 (Consistency)**  | 서로 다른 시스템·컬럼 간 데이터 값이 일치 | 고객ID, 계약ID 매핑 검증    |
  | **유효성 (Validity)**     | 정의된 도메인/형식/규칙에 맞는 값      | 생년월일이 YYYY-MM-DD 포맷 |
  | **완전성 (Completeness)** | 필수 항목 누락 없이 채워져 있음       | 주민등록번호 NULL 여부      |
  | **중복성 (Uniqueness)**   | 동일 엔티티 중복 입력 방지          | 동일 고객 두 번 등록 방지     |
  | **정확성 (Accuracy)**     | 실제 현실 세계의 참값과 일치         | 주소/전화번호 최신화         |
  | **적시성 (Timeliness)**   | 최신 정보로 업데이트 여부           | 배송 상태 갱신 시간         |

## CTQ (Critical to Quality) 정의
  ```yaml
  고객 요구: 고객 정보가 정확해야 한다
    └─ CTQ: 고객 주소 정확성
        ├─ Metric: 주소 오타율 ≤ 0.5%
        ├─ Metric: 우편번호-주소 정합성 오류율 ≤ 0.1%
    └─ CTQ: 고객 연락처 유효성
        ├─ Metric: 전화번호 정규식 미준수율 ≤ 0.2%
        ├─ Metric: 이메일 반송률 ≤ 1%
  ``` 

## RDBMS 성능 관련 주요 파라미터
  | DB         | 메모리 핵심                      | 병렬/플래너                      | WAL/로그·체크포인트                        | 특기사항                     |
  | ---------- | --------------------------- | --------------------------- | ----------------------------------- | ------------------------ |
  | Oracle     | SGA/PGA, Cache/Shared Pool  | Parallel Degree/Policy      | Redo/Log Buffer                     | NLS는 성능 무관               |
  | MSSQL      | max server memory           | MAXDOP / cost threshold     | TempDB 파일, 파일 성장                    | Query Store, 언어 설정 무관    |
  | PostgreSQL | shared\_buffers / work\_mem | random\_page/seq\_page cost | max\_wal\_size / checkpoint\_\*     | autovacuum, lc\_\* 무관    |
  | MySQL      | innodb\_buffer\_pool\_\*    | (플래너 자동)                    | innodb\_log\_file\_size / flush\_\* | character\_set/lc\_\* 무관 |

## 캐시 DB 활용 방안
  1. 자주 조회되지만 자주 변하지 않는 데이터
    - 예: 상품 정보(상품명, 카테고리, 가격), 코드 테이블(지역 코드, 국가 코드)
    - 변동이 적고 조회 빈도가 높아 캐싱 효과가 크다.
  2. 비즈니스적으로 Hot한 데이터 (핫 데이터)
    - 예: 실시간 인기 상품 랭킹, 자주 쓰이는 게시글/댓글 Top N, 세션 정보
    - 짧은 TTL(Time To Live) 기반으로 유지하면 DB 부하 완화 가능.
  3. 공통적으로 여러 서비스에서 반복 참조하는 데이터 (공통 Cache)
    - 예: 권한/역할 매핑, 환율, 설정값, 공통 마스터 데이터
    - 여러 서비스가 동일하게 참조 → Cache hit율 높음.

  ![alt text](redis_roundrobin_dpd.png)
  ![alt text](redis_roundrobin_sqd.png)

## AI 학습용 데이터 저장 설계

  ![alt text](tsdb_arch.png)

## 실행계획 명령
  | DBMS           | 실행계획 확인 명령                                                             | 실제 수행/분석 옵션                                                  | 비고                                                                   |
  | -------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------- |
  | **Oracle**     | `EXPLAIN PLAN FOR <SQL>`<br>`SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);` | `AUTOTRACE` (SQL\*Plus)<br>`V$SQL_PLAN`                      | 기본 `EXPLAIN PLAN`은 **실제 실행 안 함**. `AUTOTRACE`, `TRACE` 해야 실제 실행통계 확인 |
  | **PostgreSQL** | `EXPLAIN <SQL>`                                                        | `EXPLAIN ANALYZE <SQL>`                                      | `EXPLAIN`은 추정 계획만, `ANALYZE` 붙여야 실제 실행 및 row/time 확인                 |
  | **MySQL**      | `EXPLAIN <SQL>`<br>`DESCRIBE <SQL>`                                    | `EXPLAIN ANALYZE <SQL>` (8.0 이상)                             | 5.x/초기버전은 단순 플랜 추정치, 8.0부터는 실제 실행통계 포함 가능                            |
  | **SQL Server** | `SET SHOWPLAN_ALL ON;`<br>`SET SHOWPLAN_XML ON;`                       | `SET STATISTICS PROFILE ON;`<br>`SET STATISTICS IO/TIME ON;` | SSMS에서 “실행계획 보기(Estimated/Actual Execution Plan)” 구분                 |
  
## 실행계획분석
  | 연산자                       | 의미                          | 성능 포인트                       |
  | ------------------------- | --------------------------- | ---------------------------- |
  | **TABLE SCAN / SEQ SCAN** | 전체 테이블 순차 읽기                | 인덱스 미사용 → 성능 저하 위험           |
  | **INDEX SCAN / SEEK**     | 인덱스를 이용해 부분 범위 스캔           | 조건절 컬럼에 인덱스 필요               |
  | **INDEX RANGE SCAN**      | 범위 조건 (`BETWEEN`, `<`, `>`) | 선두 컬럼 사용 여부 중요               |
  | **INDEX FULL SCAN**       | 인덱스 전체 스캔                   | 조건 맞지 않거나 ORDER BY 최적화용      |
  | **NESTED LOOP JOIN**      | 작은 데이터 + 인덱스 적합             | 대량 데이터 시 느려짐                 |
  | **MERGE JOIN**            | 두 집합이 정렬된 상태                | Sort 비용 ↑ 가능                 |
  | **HASH JOIN**             | 대용량 조인에 효율적                 | Hash 영역 메모리 부족 시 Temp I/O 발생 |
  | **SORT / AGGREGATE**      | 정렬, 그룹핑 연산                  | Temp 사용 여부 확인                |
  | **FILTER**                | 조건절 필터링                     | 불필요한 필터는 제거 필요               |
  
  | 스캔 방식                    | 동작 원리                                   | 주요 사용 조건                                                        | 장점                        | 단점                   | 실행계획 키워드               |
  | ------------------------ | --------------------------------------- | --------------------------------------------------------------- | ------------------------- | -------------------- | ---------------------- |
  | **INDEX UNIQUE SCAN**    | 인덱스 키 값이 **정확히 1건** 매칭될 때, 인덱스만 접근      | `WHERE pk_col = 100` (PK, Unique 인덱스)                           | 가장 빠름, 결과가 1건             | = 조건이 아닌 경우 불가       | `INDEX UNIQUE SCAN`    |
  | **INDEX RANGE SCAN**     | 인덱스의 **범위 조건** 만족하는 부분만 스캔              | `WHERE col BETWEEN 10 AND 100`<br>`WHERE col LIKE 'A%'`         | 조건절 범위 탐색 효율적             | 범위 넓으면 비효율 발생        | `INDEX RANGE SCAN`     |
  | **INDEX FULL SCAN**      | 인덱스 전체를 순차적으로 읽음                        | - `ORDER BY` 최적화 (정렬 생략)<br>- 인덱스만으로 SELECT 가능 (Covering Index) | 테이블 FULL SCAN보다 효율적일 수 있음 | 불필요하게 인덱스 전체 읽을 수 있음 | `INDEX FULL SCAN`      |
  | **INDEX FAST FULL SCAN** | 인덱스 전체를 **병렬/랜덤 I/O** 로 빠르게 읽음 (정렬 불보장) | - COUNT(\*) 수행<br>- 인덱스 컬럼만 SELECT 시                            | 병렬 처리로 빠름                 | 정렬 불가능 → ORDER BY 불가 | `INDEX FAST FULL SCAN` |
  | **TABLE FULL SCAN**      | 테이블 전체를 읽음 (인덱스 무시)                     | - 인덱스 없음<br>- 대량 데이터가 조건절에 맞음<br>- 옵티마이저 판단상 유리                 | 소량 데이터시 비효율               | 대용량 Table I/O 부담 큼   | `TABLE ACCESS FULL`    |
  
  | 조인 방식                | 특징                       | 사용 적합 상황            |
  | -------------------- | ------------------------ | ------------------- |
  | **Nested Loop Join** | 한쪽 작은 집합 반복 + 다른쪽 인덱스 탐색 | 작은 데이터 조인           |
  | **Hash Join**        | 해시 테이블 생성 후 매칭           | 대용량 조인, 조인 키 인덱스 없음 |
  | **Merge Join**       | 두 집합 정렬 후 병합             | 대량 데이터 + 정렬 가능 시 효율 |
  
## 실행계획 해석 절차
  1. 접근 경로(Access Path) 확인 > Table Full Scan? Index Range Scan?
  2. 조인 방식(Join Method) 확인 > Nested Loop / Hash Join / Merge Join 선택 이유
  3. 조인 순서(Join Order) 확인 > 작은 집합부터 조인되는지?
  4. 비용/카디널리티(Cost / Rows) 확인 > 예측 행수와 실제 행수 차이 → 통계 불일치 가능
  5. 부하 연산 확인 > Sort, Group By, DISTINCT, Window Function 등
  - SQL성능분석/개선방안, 인덱스튜닝시 주의사항(IN절, 결합인덱스 등)
  
## SQL 성능 분석 및 개선 방안
  1. 실행계획 분석
    - EXPLAIN / EXPLAIN ANALYZE, AUTOTRACE, STATISTICS PROFILE 등으로 실행계획 확인
    - Table Full Scan 발생 여부 확인
    - 조인 순서, 조인 방식(Nested Loop/Hash/Merge) 검토
    - 카디널리티(예상 vs 실제 row 수 차이) → 통계 갱신 필요 여부
  2. SQL 개선 기법
    - 필요한 컬럼만 SELECT → 불필요한 I/O 줄이기
    - 불필요한 DISTINCT, ORDER BY, GROUP BY 제거
    - 조인 조건 최적화 (불필요한 CROSS JOIN 방지)
    - WHERE 조건절 → 함수/연산자 사용 자제 (ex. WHERE TO_CHAR(col) = '2025-08' → 인덱스 무시됨)
  3. 통계정보 최신화 (ANALYZE, DBMS_STATS)
    - 물리적 튜닝
    - 인덱스 생성/재구성
    - 파티셔닝 (대용량 테이블 범위조회 최적화)
    - 리라이팅(Query Rewrite) (서브쿼리 → 조인, IN → EXISTS 등)
  
## 인덱스 튜닝 시 주의사항
  - IN 절
    - IN 조건은 내부적으로 OR 조건과 유사하게 처리 → 옵티마이저가 여러 인덱스를 활용하지 못하는 경우 있음
    - 대량 값이 들어가면 Full Scan이 더 효율적일 수 있음
    - 대안
      - IN 대신 EXISTS 사용 (특히 상관 서브쿼리)
      - 값이 많으면 임시 테이블(Temp Table) 조인 방식 고려
  - 결합 인덱스(Composite Index)
    - 결합 인덱스는 선두 컬럼부터 순서대로만 효율적 사용 가능
    - 예: (col1, col2, col3) 인덱스가 있을 때
      - WHERE col1=… → 인덱스 사용
      - WHERE col1=… AND col2=… → 인덱스 사용
      - WHER  E col2=… 단독 → 선두 컬럼 col1 없으면 인덱스 효율 ↓
    - 카디널리티 높은 컬럼을 앞쪽에 배치하는 것이 일반적
    - ORDER BY, GROUP BY 컬럼 순서와 일치 시 정렬 생략 가능
  - 주의해야 할 패턴
    - WHERE function(col) = '값' → 인덱스 무효화 (함수 적용 금지)
    - LIKE '%ABC' → 선두 % 패턴은 인덱스 무효화
    - 중복/불필요 인덱스 남발 → DML 성능 저하 (INSERT/UPDATE/DELETE 시 부하 증가)
    - 너무 많은 IN 조건 (예: col IN (1,2,…10000)) → 옵티마이저가 Index Range Scan 대신 Full Scan 선택
  
    | 구분         | IN                  | EXISTS                        |
    | ---------- | ------------------- | ----------------------------- |
    | 비교 방식      | 서브쿼리 결과 전체를 모아놓고 비교 | 행마다 조건 충족 여부를 즉시 확인           |
    | 서브쿼리 결과 크기 | 작은 경우 적합            | 큰 경우 유리                       |
    | 중복 제거      | 필요 (DISTINCT 수행)    | 필요 없음                         |
    | 최적화 포인트    | 외부 쿼리 테이블 작을 때 유리   | 외부 테이블이 크더라도 내부 조건 매칭이 빠르면 유리 |
    | 대표 용도      | 소규모 목록 비교           | **상관 서브쿼리** (row-by-row 확인)   |
  
  ## 대기 이벤트
  
  ![alt text](wait_event_analysis.png)
  
  ## 현행DB 오브젝트, 트랜잭션 분석 기반 용량 산정
    - 산정 개요 (What & How)
      - 정적 용량(Structure): 테이블/인덱스/LOB/파티션/메타/시스템 영역
      - 동적 용량(Workload): 트랜잭션/Redo(로그)/WAL/Temp/Sort/증분 성장
      - 증가율: 일/주/월 신규행, 평균 행 크기, 인덱스 팩터, LOB 평균 크기
      - 여유율(Headroom): 운영 SLA에 따른 버퍼(예: 30~50%) + 장애 시 리빌드/리커버리 여유
      - 핵심 공식(예)
        하루 데이터 증가량(GB) ≈ 신규행수 × (평균행크기 + 인덱스오버헤드 + LOB평균) / 1024³
        월 예상 증가량(GB) ≈ 하루 증가량 × 30 × (성수기 보정계수)
        로그/아카이브 증가량(GB/일) ≈ (트랜잭션당 평균 로그바이트 × TPS × 86,400) / 1024³
        필요 스토리지(12개월) ≈ {현재사용량 + (월증가량 × 12)} × (여유율)
    
    - 필수 지표 (수집 체크리스트)
      - 오브젝트 크기: 테이블/인덱스/LOB/파티션별 사이즈, 행수, 평균행크기
      - 액티비티: TPS, 커밋/롤백, 배치 피크, 성수기 스파이크
      - 로그/아카이브: Redo/WAL/트랜잭션 로그 생성량(일/시간)
      - Temp/Sort: 정렬/해시/집계 시 임시공간 사용량
      - 백업/DR: 백업셋/아카이브 보존정책에 따른 저장 필요분
      - 여유공간: 테이블스페이스/파일그룹/디스크 풀의 사용률·성장성
    
    - 바로 쓰는 수집 SQL
      + Oracle
      ```sql
      -- 테이블/인덱스/LOB 사이즈 (세그먼트 기준)
      SELECT owner, segment_type, segment_name, SUM(bytes)/1024/1024 MB
      FROM dba_segments
      GROUP BY owner, segment_type, segment_name
      ORDER BY MB DESC;
      -- 테이블별 행수(통계 기반)와 평균 행 크기 추정
      SELECT owner, table_name, num_rows,
            avg_row_len, blocks, bytes/1024/1024 MB
      FROM dba_tables
      WHERE owner NOT IN ('SYS','SYSTEM')
      ORDER BY MB DESC;
      -- REDO/아카이브(일 로그 생성량 근사)
      SELECT name, value
      FROM v$sysstat
      WHERE name IN ('redo size','redo writes');
      -- Temp 사용 상위 세션 (스냅샷 기간 관측 권장)
      SELECT * FROM v$tempseg_usage;
      -- TPS 근사(스냅샷): 커밋/롤백 증가량 / 관측초
      SELECT (SELECT value FROM v$sysstat WHERE name='user commits')
          + (SELECT value FROM v$sysstat WHERE name='user rollbacks') AS total_trx
      FROM dual;
      ```
      
      + PostgreSQL
      ```sql
      -- 오브젝트 사이즈
      SELECT n.nspname AS schema, c.relname AS object, c.relkind,
            pg_total_relation_size(c.oid) AS bytes,
            pg_relation_size(c.oid) AS tbl_bytes,
            pg_indexes_size(c.oid)  AS idx_bytes,
            pg_total_relation_size(c.reltoastrelid) AS toast_bytes
      FROM pg_class c
      JOIN pg_namespace n ON n.oid=c.relnamespace
      WHERE c.relkind IN ('r','m','p')  -- table, matview, partitioned table
      ORDER BY bytes DESC;
      -- 행수 추정(통계)
      SELECT schemaname, relname, n_live_tup AS approx_rows
      FROM pg_stat_user_tables
      ORDER BY approx_rows DESC;
      -- TPS/커밋·롤백, 시간당 WAL 생성량
      SELECT datname, xact_commit, xact_rollback, blks_read, blks_hit
      FROM pg_stat_database;
      -- WAL 크기(바이트) 누계: pg_current_wal_lsn() 차이로 기간당 생성량 산출
      SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') AS wal_bytes_total;
      ```

      + MySQL / MariaDB
      ```sql
      -- 테이블/인덱스 사이즈
      SELECT table_schema, table_name,
            data_length, index_length, (data_length+index_length) AS total_bytes
      FROM information_schema.tables
      WHERE table_schema NOT IN ('mysql','performance_schema','sys','information_schema')
      ORDER BY total_bytes DESC;
      -- InnoDB 상세 (행수 추정: 테이블 통계 참고)
      SHOW TABLE STATUS;
      -- TPS/커밋/롤백, 로그 생성량 근거 (버전별)
      SHOW GLOBAL STATUS LIKE 'Com_commit';
      SHOW GLOBAL STATUS LIKE 'Com_rollback';
      SHOW ENGINE INNODB STATUS;  -- redo/LSN, history list 관측
      ```
    
      + SQL Server
      ```sql
      -- 테이블별 데이터/인덱스/LOB 사이즈
      SELECT
        s.name AS schema_name, t.name AS table_name,
        SUM(a.total_pages)*8/1024.0 AS total_MB,
        SUM(a.used_pages)*8/1024.0 AS used_MB,
        SUM(a.data_pages)*8/1024.0 AS data_MB
      FROM sys.tables t
      JOIN sys.indexes i ON t.object_id=i.object_id
      JOIN sys.partitions p ON i.object_id=p.object_id AND i.index_id=p.index_id
      JOIN sys.allocation_units a ON p.partition_id=a.container_id
      JOIN sys.schemas s ON t.schema_id=s.schema_id
      GROUP BY s.name, t.name
      ORDER BY total_MB DESC;
      -- 행수
      SELECT s.name AS schema_name, t.name AS table_name, SUM(p.rows) AS rows
      FROM sys.tables t
      JOIN sys.schemas s ON t.schema_id=s.schema_id
      JOIN sys.partitions p ON t.object_id=p.object_id AND p.index_id IN (0,1)
      GROUP BY s.name, t.name
      ORDER BY rows DESC;
      -- 트랜잭션/로그 사용량
      SELECT * FROM sys.dm_tran_database_transactions;
      SELECT * FROM sys.dm_db_log_space_usage;  -- 로그 사용률
      ```
    
    - 산정 절차 (Step-by-step)
      - 현행 스냅샷(7~14일)
        - 위 쿼리 주기 실행(예: 5~15분 간격) → 피크/평균/최소 확보
        - WAL/Redo, Temp, 백업/아카이브 생성량 “하루” 기준으로 집계
      - 정적 용량 합산
        - 테이블 + 인덱스 + LOB + 파티션 + (SYSTEM/메타)
        - 압축/블록 크기, fillfactor(예: PG 테이블 100%가 아님) 반영
      - 동적 성장 추정
        - 신규행/일 × (평균행크기 + 인덱스오버헤드 + LOB평균)
        - 로그/WAL 생성량(GB/일), Temp 피크(GB/일) → 보수적 반영
      - 보존정책 반영
        - 아카이브/백업 보존기간 × 일 생성량 → 별도 저장 필요량
      - 12~18개월 용량 계획
        - 월증가량 × 12~18 + 현재사용량
        - 여유율(운영 30~50% + 재빌드/리커버리 버퍼)
      - 시나리오 테스트
        - 성수기 ×1.5~3.0 배, 신규 기능 런칭, 마이그레이션/리인덱스 이벤트
    
    - 계산 예시 (간단 수치 샘플)
      - 현재 사용량: 2.0 TB
      - 평균 증가: 데이터 20 GB/일, 로그 10 GB/일, 아카이브 보존 14일
      - 월 증가량: 20 GB × 30 = 600 GB
      - 12개월 필요 데이터: 2.0 TB + 0.6 TB × 12 = 9.2 TB
      - 아카이브 보존: 10 GB × 14 = 140 GB
      - 여유율 40%: (9.2 TB + 0.14 TB) × 1.4 ≈ 13.1 TB
      - 권장 14~16 TB (디스크 풀/증설 단위 고려)
    
    - 리포트/시트 구성(권장 컬럼)
      - schema | object | type(table/index/lob) | rows | avg_row_len | data_bytes | index_bytes | toast/lob_bytes | total_bytes | daily_new_rows | est_daily_growth_bytes | notes
    
    - 운영 팁
      - 통계 최신화 후 측정 (오차 ↓)
      - 대용량 LOB: 평균 크기·압축·외부 스토리지 여부 확인
      - 인덱스 리빌드/파편화 이벤트를 용량 가드레일에 포함
      - 파티셔닝: 오래된 파티션 압축/아카이브 전략으로 증분 억제
      - 백업/DR: 증분 백업률·압축률·카탈로그/메타 저장분 반영

  ![alt text](db_storage_capa_plan.png)

## 인덱스·파티셔닝·백업 전략
  - 인덱스 전략
    - 공통 원칙 (세 DB 공통)
      - 고선택도(카디널리티↑) 컬럼, 조인·WHERE·정렬·그룹 기준에 우선 적용
      - 복합 인덱스는 접두(leading) 컬럼 설계를 명확히(쿼리 패턴 기준)
      - 빈번한 갱신 컬럼·저선택도 컬럼(예: Y/N)은 주의(Oracle Bitmap 제외)
      - 커버링(필요 컬럼을 인덱스만으로 충족)과 통계 최신화로 Plan 안정성 확보
    - PostgreSQL
      - 종류: B‑tree(기본), GIN(다중값/전문검색/JSONB), GiST(범위/공간), BRIN(대용량·칼럼 정렬성 활용), Hash(특정 용도), Partial/Expression, INCLUDE(커버링)
      - 팁: 부분 인덱스(Partial)로 쓰기부하↓, BRIN으로 시계열 대테이블 스캔 최소화
      - 예시
      ```sql
      -- 커버링 + 조건 최적화
      CREATE INDEX ix_orders_date_status ON orders(order_date, status) INCLUDE (amount);
      -- 부분 인덱스(취소 제외)
      CREATE INDEX ix_orders_active ON orders(id) WHERE status <> 'CANCELLED';
      -- JSONB 검색(GIN)
      CREATE INDEX ix_doc_tags_gin ON docs USING GIN (tags jsonb_path_ops);
      ```
    - MySQL (InnoDB)
      - 종류: B‑tree(PK/세컨더리), FULLTEXT, SPATIAL, Functional Index(8.0+), Invisible Index
      - 팁: Leftmost Rule(복합 인덱스 접두 컬럼 일치) 준수, 보조 PK로 AUTO_INCREMENT PK 단일 권장, 과도한 세컨더리 인덱스는 쓰기 지연 유발
      - 예시
      ```sql
      -- 함수 기반 인덱스(8.0+)
      CREATE INDEX ix_user_email_lc ON users ((LOWER(email)));
      -- 보이지 않는 인덱스로 안전 제거 검증
      ALTER TABLE orders ALTER INDEX ix_old INVISIBLE;
      ```
    - Oracle
      - 종류: B‑tree(대부분), Bitmap(저카디널리티/분석), Function‑Based, Reverse Key(순차 PK 핫스팟 완화), Partitioned Index(LOCAL/GLOBAL), Invisible Index, Index Compression
      - 팁: DML 많은 OLTP는 Bitmap 지양, 대규모 조회·집계엔 유효. Global vs Local 명확히(파티션 전략과 함께)
      - 예시
      ```sql
      -- 함수 기반 + 가상칼럼
      ALTER TABLE USERS ADD (EMAIL_LC GENERATED ALWAYS AS (LOWER(EMAIL)));
      CREATE INDEX IX_USERS_EMAIL_LC ON USERS(EMAIL_LC);
      -- Reverse Key: 시퀀스 핫스팟 완화
      CREATE INDEX IX_ORDERS_PK_REV ON ORDERS(ORDER_ID) REVERSE;
      ```
  
  - 파티셔닝 전략
    - 공통 선택 기준
      - 시계열 Range(일/월 기준) 기본값, 핫·콜드 분리/아카이브 용이
      - 균등 분포가 어려우면 Hash/Key로 균형화, 다차원은 Composite
      - 파티션 프루닝이 쿼리 패턴과 맞물리도록 파티션 키 = 필터 컬럼
    - PostgreSQL (Declarative)
      - 방식: Range / List / Hash, 서브파티션 가능, 인덱스는 파티션별(Local)만(글로벌 인덱스 미지원)
      - 운영: ATTACH/DETACH로 롤링 윈도우 관리, BRIN과 궁합 좋음(시계열 대용량)
      - 예시
      ```sql
      CREATE TABLE sales (
        id bigserial, sale_date date, region text, amount numeric
      ) PARTITION BY RANGE (sale_date);
      CREATE TABLE sales_2025_08 PARTITION OF sales
        FOR VALUES FROM ('2025-08-01') TO ('2025-09-01');
      ```
    - MySQL (InnoDB)
      - 방식: RANGE / LIST / HASH / KEY, 서브파티션 지원, 인덱스는 파티션 로컬
      - 제약: 글로벌 세컨더리 인덱스 없음(쿼리 설계로 보완), 파티션 키가 인덱스 접두에 포함되도록 설계
      - 예시
      ```sql
      CREATE TABLE orders (
        id bigint primary key,
        order_date date,
        status varchar(20), amount decimal(12,2),
        KEY ix_date_status (order_date, status)
      ) PARTITION BY RANGE (TO_DAYS(order_date)) (
        PARTITION p202508 VALUES LESS THAN (TO_DAYS('2025-09-01'))
      );
      ```
    - Oracle (Enterprise Partitioning)
      - 방식: Range / List / Hash / Interval, Composite(Range-Hash, Range-List), 참조 파티셔닝(Reference) 지원
      - 인덱스: LOCAL / GLOBAL 모두 가능(운영 유연성↑), 파티션 교체(Exchange), 온라인 리빌드 성숙
      - 예시
      ```sql
      CREATE TABLE SALES (
        ID NUMBER, SALE_DT DATE, REGION VARCHAR2(20), AMOUNT NUMBER
      ) PARTITION BY RANGE (SALE_DT) INTERVAL (NUMTOYMINTERVAL(1,'MONTH'))
      ( PARTITION P202508 VALUES LESS THAN (DATE '2025-09-01') );
      -- Local partitioned index
      CREATE INDEX IX_SALES_DT_LOC ON SALES(SALE_DT) LOCAL;
      ```
  
  - 백업·복구 전략 (RPO/RTO 중심)
    -  공통 설계
      - 초기Full + 주기Incremental + 트랜잭션로그 아카이브(PITR)
      - 백업무결성검증(restore test), 암호화, 보존정책, 오프사이트/클라우드 복제
      - 대규모는 스냅샷(스토리지 레벨) + 논리백업(스키마이식) 병행
    - Oracle
      - 도구: RMAN(물리 Full/Level 0·1 증분, 블록체인지 추적), Data Pump(expdp/impdp, 논리)
      - ARCHIVELOG + RMAN + FRA로 PITR, 대규모 DR은 Data Guard
      - 예시(개념):
      ```bash
      -- RMAN 증분 + 카탈로그
      RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;
      RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;
      -- 복구: SET UNTIL TIME ...; RESTORE ...; RECOVER ...;
      ```
    - PostgreSQL
      - 도구: pg_dump/pg_dumpall(논리), pg_basebackup(물리), 아카이브 모드+WAL로 PITR
      - 운영형: pgBackRest / Barman(증분·병렬·압축·보존)
      - 예시(개념):
      ```bash
      # 물리 베이스백업 + WAL 보관
      pg_basebackup -h primary -D /backup/base -X stream -C -S bkp_slot
      # 복구(PITR): recovery.signal + restore_command + target_time
      ```
    - MySQL
      - 도구: mysqldump(논리), MySQL Enterprise Backup / Percona XtraBackup(물리, Hot)
      - Binary Log 기반 PITR, GTID 권장
      - 예시(개념):
      ```bash
      # 물리 증분(XtraBackup)
      xtrabackup --backup --target-dir=/bkp/full
      xtrabackup --backup --target-dir=/bkp/inc1 --incremental-basedir=/bkp/full
      # 복구 후 binlog 적용으로 PITR
      mysqlbinlog --start-datetime="2025-08-15 00:00:00" binlog.000123 | mysql
      ```

    | DBMS       | 물리 백업                        | 논리 백업                  | PITR 지원        | 운영 도구/특징               |
    | ---------- | ---------------------------- | ---------------------- | -------------- | ---------------------- |
    | Oracle     | RMAN Full/증분                 | Data Pump(expdp/impdp) | ARCHIVELOG 기반  | Data Guard, FRA        |
    | PostgreSQL | pg_basebackup + WAL         | pg\_dump/pg\_dumpall   | WAL Archive 기반 | pgBackRest(증분/병렬/압축/보존정책 + WAL 관리 + PITR), Barman     |
    | MySQL      | XtraBackup/Enterprise Backup | mysqldump              | Binlog + GTID  | Hot Backup 지원          |
    | SQL Server | Full/Differential/T-Log      | BCP, SSMS Export       | T-Log 기반       | AlwaysOn/AG, Filegroup |
    
    + PG: 주 1회 Full + 일일 Diff + 시간단위 Inc (pgBackRest), WAL 아카이브, 보존정책 관리
    + MySQL: 주 1회 Full(XtraBackup) + 일일 Incremental + binlog 보관, GTID
    + Oracle: RMAN Full/증분 + ARCHIVELOG + (옵션) Data Guard + FRA
    + MSSQL: Full(주간) + Diff(일일) + T-Log(15~30분), 정기 복구 리허설
    
    ### Oracle Data Guard, FRA
    | 항목     | **Data Guard**                              | **FRA**                               |
    | ------ | ------------------------------------------- | ------------------------------------- |
    | 목적     | DR/고가용성 (Standby 운영)                        | 백업/복구 스토리지 통합 관리                      |
    | 방식     | Primary→Standby Redo 전송                     | RMAN·Archived Log·Flashback 저장        |
    | 적용 범위  | 멀티 서버(Primary + Standby)                    | 단일 DB 인스턴스 스토리지                       |
    | 장점     | 장애 시 Failover/Switchover                    | 백업 관리 자동화, Flashback 지원               |
    | 시험 포인트 | 보호 모드 (Protection/Availability/Performance) | `db_recovery_file_dest`, Flashback DB |
    
    ![alt text](orcl_dgfra_arch1.png)
    
    ### PostgreSQL
    | 구분              | **pgBackRest**                         | **Barman**                 | **pg\_basebackup**               |
    | --------------- | -------------------------------------- | -------------------------- | -------------------------------- |
    | **백업 방식**       | 물리 (Full / Differential / Incremental) | 물리 (Full + WAL 기반 증분)      | 물리 (Full only)                   |
    | **증분 백업**       | 지원 (Diff/Inc, 블록 단위)                   | 지원 (WAL 기반)                | 미지원 (항상 Full)                    |
    | **병렬 처리**       | 지원 (멀티스레드 백업/복구/압축)                    | 일부 지원 (rsync 멀티 프로세스)      | 미지원                              |
    | **압축/암호화**      | 지원 (lz4, zstd, gzip / GPG 암호화)         | 지원 (gzip/bzip2, 일부 암호화)    | 미지원                              |
    | **WAL 관리**      | 자체 WAL 아카이브 관리, 보존 정책 지원               | WAL 아카이브/스트리밍 관리 중심        | 단순 아카이브 (별도 archive\_command 필요) |
    | **PITR (시점복구)** | 지원 (WAL 자동 적용)                         | 지원 (WAL 기반 복구)             | 지원 (WAL과 조합 시)                   |
    | **보존 정책**       | Full/Diff/Inc 세트 단위 자동 관리              | Retention Policy 제공        | 없음 (운영자 직접 관리)                   |
    | **저장소**         | 로컬/NFS/오브젝트 스토리지(S3, Azure Blob 등)     | 로컬/원격 서버 중심                | 로컬 디렉토리만                         |
    | **운영 난이도**      | 중간 (설정 다소 복잡, 안정적)                     | 중간 (전용 서버/아키텍처 필요)         | 낮음 (간단하지만 기능 한계)                 |
    | **대표 활용**       | 대규모 운영환경(수 TB) / 엔터프라이즈 백업 표준          | 원격 DR센터 백업, WAL 스트리밍 중심    | 단일 서버 기본 백업, PoC/소규모 환경          |
    | **장점**          | 엔터프라이즈급 기능, 병렬/증분/압축 최강                | WAL 관리/DR에 특화, 원격 보관 강점    | 기본 내장, 설정 단순                     |
    | **단점**          | 설정 복잡, 초기 러닝커브                         | 기능 제한, 확장성 pgBackRest보다 낮음 | 증분/병렬 없음, 대용량 비효율                |
    
    ![alt text](pg_bkrst_arch1.png)
    
    ### MySQL
    | 항목         | **Percona XtraBackup** | **MySQL Enterprise Backup (MEB)** |
    | ---------- | ---------------------- | --------------------------------- |
    | 라이선스       | 오픈소스(무료)               | 상용(Enterprise Edition 구독 필요)      |
    | Hot Backup | ✅ InnoDB 지원            | ✅ InnoDB + MyISAM 등 엔진 지원         |
    | 증분/차등      | 지원                     | 지원                                |
    | 압축/암호화     | 지원(lz4/zstd/gzip, GPG) | 지원(내장)                            |
    | 병렬 처리      | 지원                     | 지원                                |
    | PITR       | Binlog과 연계             | Binlog과 연계                        |
    | 지원 대상      | InnoDB 중심              | InnoDB + MyISAM + 기타 엔진           |
    | 관리 편의      | CLI 기반, 스크립트 자동화       | GUI·Oracle Enterprise 관리 툴 연계     |
    | 비용         | 무료                     | 유료                                |
    
    ![alt text](mysql_pcxtr_arch1.png)
    
    ### SQL Server
    | 항목     | AlwaysOn AG                      | Filegroup Backup                     |
    | ------ | -------------------------------- | ------------------------------------ |
    | 목적     | 고가용성 + DR                        | 대용량 DB 효율적 백업/복구                     |
    | 단위     | 데이터베이스 단위                        | Filegroup 단위                         |
    | 모드     | Sync (HA), Async (DR)            | Full / Partial Restore               |
    | 장점     | 무중단 Failover, Secondary 읽기 부하 분산 | 빠른 복구, 불필요 재백업 최소화                   |
    | 시험 포인트 | 자동 Failover, Secondary 백업        | Partial Restore, Read-Only Filegroup |
    
    ![alt text](mssql_aoagf_arch1.png)
    
    - pg_dump 방식과 restore 방식등 다르다.
    - 각 RDBMS별 백업/복구툴, Command에 대해서 기본적인 접근, 숙지, 검색가능하게 
    - 백업과 복구도구(RMAN, pgdump, Backup/Restore 등)
    - pg_dump/pg_restore, RMAN, mysqldump: DB백업 및 복구 도구
    - PostgreSQL, MySQL, Oracle 등의 인덱스, 파티셔닝, 백업전략 비교

  - 조합 가이드(상황별 처방)
    - 시계열 대용량(로그/거래)
      - PG: Range 파티션 + BRIN(스캔 최소화) + pgBackRest + WAL PITR
      - MySQL: Range/KEY 파티션 + 읽기복제 + XtraBackup + binlog PITR
      - Oracle: Interval Range + Local Index + RMAN 증분 + Data Guard
    - 쓰기많고 조회패턴 명확한 OLTP
      - 인덱스는 최소/정확, 복합은 접두 정렬 맞추기
      - 읽기 확장: read replica(또는 Active Data Guard) + 캐시(예: Redis)
    - 스키마 이식·선택적 마이그레이션
      - 논리백업(pg_dump, mysqldump, DataPump) + CDC이행(변경분적용)

## 트랜잭션 격리 수준(Isolation Level) 개요
  | 격리수준                 | Dirty | Non-repeatable | Phantom | 비고                                |
  | -------------------- | ----- | -------------- | ------- | --------------------------------- |
  | **READ UNCOMMITTED** | 허용    | 허용             | 허용      | Oracle 미지원(비권장)                   |
  | **READ COMMITTED**   | 차단    | 허용             | 허용      | **기본값: Oracle/PG/SQL Server**(전통) |
  | **REPEATABLE READ**  | 차단    | 차단             | 가능      | **기본값: MySQL(InnoDB)**            |
  | **SERIALIZABLE**     | 차단    | 차단             | 차단      | 가장 엄격(직렬화)                        |
  | **SNAPSHOT**(DB옵션)   | 차단    | 차단             | 차단      | MVCC 스냅샷 기반, 시스템마다 명칭/동작 상이       |

  + Dirty Read: 커밋 안 된 값을 읽음
  + Non-repeatable Read: 같은 행을 두 번 읽을 때 값이 달라짐(중간 커밋 반영)
  + Phantom Read: 같은 조건을 두 번 조회할 때 “행의 수”가 달라짐(삽입/삭제로 유령행 발생)
  + Lost Update: 서로의 갱신이 덮어씌워짐
  + Write Skew: 개별 행 제약은 만족하지만, 조합 제약이 깨지는 스냅샷 계열 특유의 이상

  ``` sql
  -- 1) 기본: READ COMMITTED (문 단위 스냅샷)
  SELECT COUNT(*) FROM t;          -- 예: 100
  -- 다른 세션이 INSERT COMMIT
  SELECT COUNT(*) FROM t;          -- 101 (문 사이 최신 커밋 보임)

  -- 2) 트랜잭션 스냅샷
  SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  SELECT COUNT(*) FROM t;          -- 100
  -- 다른 세션 INSERT COMMIT
  SELECT COUNT(*) FROM t;          -- 여전히 100 (트랜잭션 시점 고정)
  COMMIT;
  ```

## 프로젝트 단계별 통계정보 정책 수립 전략
  - 프로젝트에서 통계정보는 설계 → 개발/테스트 → 통합 테스트 → 운영의 단계별로 수립·관리한다.
    - 설계: 데이터특성·조회패턴·성능요구사항 기반 통계정보수집·갱신주기, 수집방식(자동/수동), 파티션·인덱스설계 연계한 정책수립
    - 개발/단위테스트: 샘플데이터 이용해 초기통계정보 수집하여 실행계획검증과 성능튜닝 수행 
    - 통합테스트: 운영데이터 규모와 유사한 데이터셋으로 통계정보 재수집, 실제쿼리부하에 따른 옵티마이저 동작 검증하여 통계정보확정
    - 운영이행전: 통계정보 최신상태 갱신하여 쿼리계획안정성 확보, 
    - 운영이행후: 자동통계갱신(Auto Statistics)과 주기적 수동수집 병행하여 성능저하 예방 
  - 정책에는 수집시점·방법·대상테이블·샘플링비율·파티션별 통계수집여부 등을 명시해 일관성 유지

  ![alt text](prjt_dbstat_strtgy.png)

## 온프레미스에서 Azure로의 단계적 마이그레이션 전략
  1. Azure Migrate로 서버·DB·애플리케이션 종속성 분석/전환우선순위 도출 
  2. DB이행은 Azure DMS|GoldenGate|pglogical과 같은 CDC를 사용해 다운타임 최소화 
  3. 대용량 배치는 Azure Data Factory로 초기 적재, 변경분은 CDC(Kafka + Debezium)로 스트리밍 복제
  4. 아키텍처설계 시 공통데이터는 단일 중앙DB로 모으고, 서비스별전용데이터는 개별인스턴스 분리하여 Azure에서 분산구조로 배치. 데이터 파티셔닝은 날짜·업무키 기반 Range 또는 Hash 파티션을 적용 읽기성능 최적화
  3. EAI는 온프레미스 ESB|API Gateway <> Azure API Management 연계, Hybrid 환경에서 서비스 호출 가능
  6. 데이터 복제 방식은
    - 동기 복제: 강한 일관성, 지연 증가 가능
    - 비동기 복제: 성능 우수, eventual consistency 허용
    - CDC 복제: Binlog/WAL/Redo 기반 실시간 변경전송, 월단위 스냅샷/백업은 Blob Storage/ADLS로 보관해 재해 복구 지원
  7. 마이그레이션 절차 예시:
    1. 자산 분석(Azure Migrate) > 2. 신규 Azure 환경 구성(VNet, Subnet, Peering) > 3. 초기 데이터 적재(ADF) > 4. 실시간 변경 복제(DMS, GoldenGate, Kafka CDC) > 5. 점진적 서비스 전환(Read → Write) > 6. 최종 스위치오버
  
  ![alt text](onprem2azure_mig_arch1.png)

## 데이터베이스 오브젝트 분석, 복제구성, DB서버 부하분산 방안
  ![alt text](db_workload_balance_arch1.png)


## 데이터베이스 모니터링 및 성능 점검 전략
  - 데이터베이스 모니터링은 운영 안정성 확보와 성능 저하 사전 예방을 위해 주기별로 관찰 지표를 구분한다.
  - 일 단위 필수 모니터링 지표
    - CPU 사용률, 메모리 사용률, 디스크 I/O 대기율
    - 장애·오류 로그 발생 건수
    - 주요 테이블 Lock 발생 빈도
    - 장애 상황 조기 탐지 목적의 알람성 지표 위주
  -  일 단위 모니터링 불필요/선택 지표 (필요 시에만 점검)
    - 장기 쿼리 실행 계획 변화
    - 전체 인덱스 사용률 및 미사용 인덱스 목록
    - 전체 테이블/파티션별 통계정보 변경 내역 → 빈번히 변하지 않으므로 주 단위나 장애 발생 시 분석
  - 주 단위/정기 점검 지표
    - AWR(Oracle) / pg_stat_statements(PostgreSQL)를 통한 Top SQL 분석
    - 실행계획 안정성(Plan Stability) 및 통계정보 최신화 여부
    - I/O Wait Event 상위 항목 분석(DB 부하 패턴)
    - 커넥션수 추이, 트랜잭션 처리량(TPS/QPS) 비교
    - 인덱스 조각화, 테이블 파티션 밸런스
  - 점검 방법
    - Oracle: AWR/ASH 리포트, V$ 뷰, OEM 대시보드
    - PostgreSQL: pg_stat_statements, pg_stat_activity, auto_explain
    - 부하상황별 모니터링:
      - CPU 부하시 → SQL실행시간·병렬처리수 확인
      - I/O 부하시 → Wait Event·스토리지 IOPS 확인
      - 메모리 부하시 → Buffer Cache Hit Ratio, Temp Usage 분석

## 1.21. Azure VPN/ExpressRoute, Vnet, App Service, AKS, Azure SQL 등 계층적 구성 설명
  - 데이터복제, 서버분산, App/Web/DB 계층 Azure 서비스 맵핑
  - Azure Vnet/Subnet: 네트워크논리분할, 보안설정
  - Azure App Service/AKS: 웹앱/마이크로서비스 호스팅
  - Azure SQL/PostgreSQL/Blob Storage: 주요 저장소 역할 및 특성
  - Azure Database Migration Service: 온프레미스 DB를 Azure로 마이그레이션하는 도구
  - [사용자] -> [VPN/ExpressRoute] -> [AzureVnet] -> [AppService/AKS] -> [AzureSQL/CosmosDB] -> [BlobStorage/Redis/LogAnalytics]
  
## AI기반 데이터 모델링(백터DB등)
### Chunking
### Embedding(DenseVector생성)
### VectorDB

## Monolithic 아키텍처에서 Micro Service 아키텍처로 분산시에 데이터 복제, 연동 방안
![alt text](onprem2azure_mig_arch2.png)

![alt text](msa_seq_diagram1.png)

## 보안 및 암호화 설계
  - AES256 등 필수암호화는 '개인정보보호법'기준에 따라 적용
  - 조건절에서 성능 이슈 발생시 대체키 설계 고려
  - 단방향 암호화된 비밀번호는 복호화 불가, 분실시 재설정 방식 권장
  - DB 계정 보안 등 업무상 고려해야 할 사항
  - DB 암호화에 대한 개념, 암호화된 컬럼의 performance 개선에 대한 방법론
  - |데이터보호|단방향암호화/AES|복호화불가암호화 또는 양방향암호화|
  - 자주사용하지 않는 개인정보 마스킹. 항목이 아닌 것들이 좀 들어가는 있어서 좀 헷갈림.
  - DB계정보안 및 테스트환경 구성에 대한 실무적 고려사항
  - 자주 사용되지 않는 고객정보 보관 및 암호화/마스킹 방안
  - DB계정분리
  - DB암호화 정책
  - TDE/Always Encrypted: DB단위/컬럼단위 암호화기법
  - RBAC: 역할 기반 접근 제어로 DB 계정 보안 강화
  - Key Vault: 암호화키 및 보안정보 중앙관리

## 고객요구사항 기반 DB분석 및 개선
  - 요구사항 분석 템플릿: 요구사항정의서를 만드는 것이다. 구조적으로는 뭐가 필요하고 성능적으로 뭐가 필요하고 현재 문제가 뭐고 해결하려면 뭐가 필요하다.예시를 2개는 준다. 지문에 있는 것을 따와야하고 생각을 너무 적으면 안된다. 그 밑에 적혀있는 것은 객관식 하나하나의 문제다.
  - 요구사항정의서를 작성할 수 있어야합니다. 
  - 아키텍처 구조 및 성능 관점에서 현재의 문제점과 개선방안을 도출할 수 있어야 합니다.
  - SQL작성가이드에서는 기존의 SQL 목록과 튜닝가이드 목록을 제시한다. 
  - 성능테스트결과서에서는 XXX 가 나온다. 대충 보았을때는 다 맞는 것 같은데 다 꼬아져있다. 중상으로 가기때문에. 문제가 다 꼬아져있다.

  ### 요구사항 정의서 예시
  | 분류     | 요구사항명                  | 요구사항내용                                 | 해결/개선방안                              | 반영 산출물          |
  | ------ | ---------------------- | -------------------------------------- | ------------------------------------ | --------------- |
  | **구조** | 파티션 설계 최적화  | 대용량 로그 테이블에서 전체 스캔 발생, 조회 성능 저하 | 날짜 기반 파티셔닝(연·월·일) 적용, 파티션 프루닝 활용     | 논리 ERD, 파티션 설계서 |
  | **구조** | 금칙어 정비                 | 기존 시스템 변경 없이 속성명에 포함된 ‘직원번호’ 식별자 제거 필요 | 금칙어 ‘직원번호’ 제거, 표준용어 ‘직원’으로 대체        | 용어 표준 변경 정의서    |
  | **구조** | 선분이력 설계 검토             | 종료일시를 PK에 포함할 경우 기간 중복 발생 가능           | 종료일시는 일반 속성으로 분리, PK는 ‘코드 + 시작일’ 구성  | 이력 테이블 설계서      |
  | **성능** | SQL 병목 제거   | 특정 SQL의 실행 지연(10초 이상)           | 인덱스 신규 설계, 파티션 조건 추가, SQL 리팩토링       | SQL 성능 분석 보고서   |
  | **성능** | 실행 계획 분석(EXPLAIN)      | SQL의 실제 실행 계획 확인 필요                    | `EXPLAIN ANALYZE` 활용하여 병목 구간 분석 및 개선 | 실행 계획 비교 리포트    |
  | **운영** | Oracle → PostgreSQL 전환 | Timestamp 등 데이터 타입 차이로 인한 변환 이슈        | 호환 가능한 데이터 타입 매핑, 스키마 변환 자동화 도구 활용   | 마이그레이션 매뉴얼      |
  | **운영** | DB 분산 및 경량화 | 모든 업무 데이터가 통합 DB에 집중되어 운영 부하 가중 | 업무·지역 단위 DB 분리, 공통 DB + API 참조 구조 도입 | 분산 DB 아키텍처 설계서  |
  | **보안** | 암호화 정책 개선   | 암호화 컬럼 조건절 사용 시 성능 저하           | 대체키 설계, 마스킹 뷰 활용                     | 암호화 정책 설계서      |
  | **보안** | 비밀번호 정책                | 복호화 불가한 안전한 암호화 필요                     | SHA-256 단방향 암호화 적용, 비밀번호 재설정 절차 안내   | 비밀번호 정책 가이드     |
  | **백업** | 백업 공간 확보    | 월 1회 점검으로 백업 중 임시 테이블스페이스 부족 가능 | 자동 확장 정책 수립, 수시 모니터링                 | 백업 운영 매뉴얼       |


[ DATA Engineering ] 데이터 엔지니어링 관련 KICE 학습 참조
# 데이터 아키텍처 및 엔지니어링 전략
ㅇLakehouse, data fabric, data mesh 개념 설명
ㅇFact Table과 다중 계층 테이블 찾기
ㅇBusiness META, technical  META 구분하기
ㅇRAG. KAFKA 개념 정확히 알고 있어야 하며, 설명 중 잘못된 부분을 확인 할 수 있어야 함. 
ㅇ데이터 프로파일링
ㅇ데이터 스토어 특징 비교표 : RDB, key-value(JSON형식), Hadoop, Object Storage
ㅇ데이터 관리 사례 기반 주제 
   - 파티셔닝 방법으로 맞는 것 (파티셔닝 컬럼 찾기)
   - SQL문장 완성하기
   - 코드 보고 고객 요구사항 정리하기 (마케팅 자료 추출, ㅇ월-ㅇ월 자료..)
## ✅ 데이터 모델링 및 ELT 프로세스
### 1. 데이터 모델링 프로세스
- **요구사항 분석 → 개념 모델(CDM) → 논리 모델(LDM) → 물리 모델(PDM)** 순
- 개념 모델: 비즈니스 용어 기반 ERD
- 논리 모델: 정규화, 제약조건 명시
- 물리 모델: DBMS 종속적 구현
### 2. ELT 과정
- Extract → Load → Transform
- 대용량 병렬 처리에 유리
- Snowflake, BigQuery 같은 클라우드 DWH에 적합
---
## ✅ 데이터 레이크하우스 및 DW 구조 비교
### Lakehouse 특징
- ACID 트랜잭션 + 오픈 포맷 (Parquet 등)
- 분석 및 머신러닝 통합
- DW의 정형성과 DL의 유연성 결합
### 비교 요약
| 항목 | DW | Data Lake | Lakehouse |
|------|----|-----------|-----------|
| 스키마 | Schema-on-write | Schema-on-read | Hybrid |
| 활용 | 정형 분석 | 비정형 저장 | 분석 + ML 통합 |
| 성능 | 고속 쿼리 | 유연성 중심 | 최적화된 쿼리 + 확장성 |
---
## ✅ 개인정보 보호법 기반 설계 평가
- `개인정보의 안전성 확보조치 기준`을 기준으로 암호화 대상 판단
- 주요 항목:
  - 주민등록번호, 계좌번호 등은 필수 암호화
  - 저장 시 AES256, 전송 시 TLS 등 적용 권장
- 설계 평가 시, DB 컬럼 단위 암호화 정책 수립 필요
---
## ✅ 데이터 적재 설계 (테이블 유형별)
| 테이블 유형 | 적재 방식 |
|-------------|-----------|
| 마스터 테이블 | 전체 적재 or 갱신일 기준 증분 |
| 이력 테이블 | CDC 기반 증분 적재 |
| 트랜잭션 테이블 | 일별 분할 적재 or 파티션 병렬 |
---
## ✅ 고객 디멘전 연결 모델링
### 유형별 이력 관리 모델
1. Type 1: 덮어쓰기
2. Type 2: 이력 유지 (시작/종료일, 버전 관리)
3. Type 3: 한 개 이전 값만 저장
### 선택 시 고려사항
- 고객 속성 변화 추적 필요 시 Type 2 적용
- 대체키 필요 (Surrogate Key)
---
## ✅ 다양한 데이터 스토어 구조
| 유형 | 설명 | 예시 |
|------|------|------|
| 컬럼지향 (Columnar) | 읽기 최적화 | Parquet, ORC, ClickHouse |
| 키-값 저장소 | 빠른 조회 | Redis, RocksDB |
| HDFS/Hadoop 기반 | 분산 처리 | Hive, HBase |
| 객체 스토리지 | 저비용 대용량 | S3, GCS |
---
## ✅ OLAP 및 다차원 모델
- 팩트 테이블: 이벤트 중심 (매출, 주문 등)
- 디멘전 테이블: 고객, 제품, 시간 등
- 계층 구조 예: 시간(년-월-일), 지역(국가-도-시)
---
## ✅ 데이터 거버넌스 기반 품질 진단 전략
- 품질 진단 지표: 정확성, 유일성, 정합성, 최신성
- 대응 전략:
  - 단일 변경: 빠르나 리스크 큼
  - 병행 운영: 안정성 확보
  - 메타 중심 매핑: 유연성과 확장성
---
## ✅ RTT 지연 시 영향도
- RTT 0.2초 이상: 타이핑 반응 저하
- RTT 0.4초 이상: 마우스 드래그 지연
- ETL 설계, IDE 사용, BI 도구 성능에 부정적 영향
---
## ✅ 데이터 중복 점검 SQL
```sql
SELECT pk_col1, pk_col2, COUNT(*)
FROM target_table
GROUP BY pk_col1, pk_col2
HAVING COUNT(*) > 1;
```
- NULL 여부, 형식 일관성 점검 포함 필요
---
## ✅ 문제 해결 방안 제안 및 선택
- 상황: 30개 ETL, 10개 리포트에서 표준 불일치
- 방안:
  1. 전면 교체 – 리스크 큼
  2. 병행 운영 – 안정적이지만 이중 관리
  3. **메타데이터 기반 매핑 (선택)**
### 선택 이유:
1. 리스크 최소화 (운영 영향 적음)
2. 빠른 적용 가능 (ETL 변경 없음)
3. 향후 확장에 유리 (유지보수 간편)
