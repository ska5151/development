---
name: mysql57-patterns
description: MySQL 5.7 프로덕션 백엔드를 위한 스키마, 쿼리, 인덱싱, 트랜잭션, 복제, 커넥션 풀 패턴(5.7 문법·기능 한정).
origin: ECC
---

# MySQL 5.7 패턴

이 스킬은 **MySQL 5.7**의 스키마 설계, 마이그레이션, 슬로우 쿼리 조사,
큐 스타일 트랜잭션, 커넥션 풀, 프로덕션 데이터베이스 구성을 다룰 때
사용합니다. 문법과 옵티마이저 동작은 패치 레벨(예: 5.7.8의 JSON 타입)에
따라 달라질 수 있으므로, 적용 전 `SELECT VERSION();`으로 정확한 빌드를
확인하세요.

**범위 밖:** MariaDB 전용 문법, MySQL 8.0의 `SKIP LOCKED`, `EXPLAIN ANALYZE`,
`SHOW REPLICA STATUS`, `INSERT ... AS row_alias` upsert 등은 이 문서에
포함하지 않습니다. 8.0 이상으로 올릴 때는 별도 가이드를 참고하세요.

## 활성화 시점

- MySQL 5.7 테이블, 인덱스, 제약 조건을 설계할 때
- 대규모 프로덕션 테이블에서 실행되기 전 마이그레이션을 검토할 때
- 슬로우 쿼리, 락 대기, 데드락, 커넥션 고갈을 디버깅할 때
- 키셋 페이지네이션, upsert, 전문 검색, JSON 컬럼(5.7.8+), 큐를 추가할 때
- 애플리케이션 커넥션 풀, 읽기 복제본, TLS, 슬로우 로그를 구성할 때

## 버전 확인

엔진과 빌드를 식별하는 것부터 시작합니다:

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'version_comment';
```

MySQL 5.7에서 자주 헷갈리는 제약:

- **Upsert:** `ON DUPLICATE KEY UPDATE`에서 삽입 값을 참조할 때는
  `VALUES(col)`을 사용합니다.
- **큐·워커:** `FOR UPDATE SKIP LOCKED`는 **MySQL 8.0+** 전용입니다.
  5.7에서는 아래 «큐 스타일 워커 클레임»의 대안 패턴을 사용하세요.
- **JSON:** 네이티브 `JSON` 타입과 관련 함수는 **5.7.8 이상**에서
  사용 가능합니다. 그 이전 패치는 `TEXT` + 애플리케이션 검증으로
  설계하세요.

## 스키마 기본값

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    account_id BIGINT UNSIGNED NOT NULL,
    status VARCHAR(32) NOT NULL,
    total DECIMAL(15, 2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at DATETIME NULL,
    PRIMARY KEY (id),
    KEY idx_orders_account_status_created (account_id, status, created_at),
    KEY idx_orders_active (account_id, deleted_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

기본 선택 사항:

| 사용 사례 | 권장 |
| --- | --- |
| 대리 기본 키 | `BIGINT UNSIGNED AUTO_INCREMENT` |
| UUID 조회 키 | 핫 테이블의 `VARCHAR(36)` 기본 키(또는 BINARY(16) 등 설계에 맞게) |
| 금액 및 정확한 수량 | `DECIMAL(p, s)` — 부동소수점(`FLOAT`/`DOUBLE`)은 금액에 부적합 |
| 사용자 대면 텍스트 | `utf8mb4` 테이블 및 인덱스 |
| 애플리케이션 타임스탬프 | 앱에서 UTC를 관리하는 `DATETIME` |
| 소프트 삭제 | `deleted_at DATETIME NULL` + 범위 지정 인덱스 |
| 확장 가능한 상태 값 | 룩업 테이블 또는 제약 있는 `VARCHAR` |

## 인덱싱

복합 인덱스 순서는 일반적으로 동등(equality) 술어 먼저, 그 다음 범위 또는
정렬 컬럼 순으로 따릅니다:

```sql
CREATE INDEX idx_orders_account_status_created
    ON orders (account_id, status, created_at);

SELECT id, total
FROM orders
WHERE account_id = ?
  AND status = 'pending'
  AND created_at >= ?
ORDER BY created_at DESC
LIMIT 50;
```

인덱스를 추가하거나 변경하기 전에 `EXPLAIN`을 사용하세요:

```sql
EXPLAIN
SELECT id, total
FROM orders
WHERE account_id = 123 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 50;
```

5.7에서는 `EXPLAIN FORMAT=JSON`으로 더 자세한 계획을 볼 수 있습니다.
`EXPLAIN ANALYZE`(실행 포함)는 **MySQL 8.0.18+** 전용이므로 5.7에서는
사용할 수 없습니다.

조사해야 할 신호:

| 필드 | 위험 신호 |
| --- | --- |
| `type` | 큰 테이블에서 `ALL` |
| `key` | 선택적 술어가 있는데 `NULL` |
| `rows` | 인터랙티브 경로에서 매우 높은 행 추정치 |
| `Extra` | `Using temporary`, `Using filesort`, 또는 광범위한 `Using where` |

맹목적으로 인덱스를 추가하지 마세요. 각 인덱스는 쓰기 비용, 마이그레이션 시간,
백업 크기, 그리고 버퍼 풀 압력을 증가시킵니다.

## 쿼리 패턴

### Upsert (5.7)

MySQL 5.7에서는 `VALUES(col)`이 표준입니다.

```sql
INSERT INTO user_settings (user_id, setting_key, setting_value)
VALUES (?, ?, ?)
ON DUPLICATE KEY UPDATE
    setting_value = VALUES(setting_value),
    updated_at = CURRENT_TIMESTAMP;
```

### 키셋 페이지네이션

```sql
SELECT id, name, created_at
FROM products
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

커서와 일치하는 인덱스로 뒷받침하세요:

```sql
CREATE INDEX idx_products_created_id ON products (created_at, id);
```

대규모 테이블에서 깊은 `OFFSET` 페이지네이션은 사용하지 마세요. 서버가
페이지를 반환하기 전에 행을 스캔하고 버리게 됩니다.

### JSON 필드 (5.7.8+)

JSON 컬럼은 확장 데이터에 사용하고, 관계형 필터링이나 제약이 많이 필요한
필드에는 사용하지 마세요.

```sql
CREATE TABLE events (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    payload JSON NOT NULL,
    event_type VARCHAR(64)
        GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(payload, '$.type'))) STORED,
    KEY idx_events_type (event_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

자주 쿼리되는 JSON 경로의 경우, 생성된 컬럼(generated column)을 노출하고 그
컬럼에 인덱스를 만드세요. 외래 키, 소유권, 테넌시, 라이프사이클 필드는
관계형으로 유지하세요.

### 전문 검색 (Full-Text Search)

```sql
ALTER TABLE articles ADD FULLTEXT KEY ft_articles_title_body (title, body);

SELECT id, title, MATCH(title, body) AGAINST (? IN NATURAL LANGUAGE MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST (? IN NATURAL LANGUAGE MODE)
ORDER BY score DESC
LIMIT 20;
```

오타 허용, 복잡한 랭킹, 교차 테이블 패싯(facet), 또는 내장 전문 검색 동작을
넘어서는 언어별 분석이 필요한 경우 외부 검색 엔진을 사용하세요.

## 트랜잭션

트랜잭션은 짧게 유지하고 일관된 순서로 행을 잠그세요:

```sql
START TRANSACTION;

SELECT id, balance
FROM accounts
WHERE id IN (?, ?)
ORDER BY id
FOR UPDATE;

UPDATE accounts SET balance = balance - ? WHERE id = ?;
UPDATE accounts SET balance = balance + ? WHERE id = ?;

COMMIT;
```

데드락 및 락 대기 체크리스트:

- 모든 코드 경로에서 결정적인 순서로 행을 잠그세요.
- 외부 API 호출은 트랜잭션을 열기 전에 수행하고, 트랜잭션 내부에서는 하지 마세요.
- `UPDATE`, `DELETE`, 잠금 읽기에서 사용되는 술어에 인덱스를 추가하세요.
- 데드락 시 롤백하고, 제한된 재시도 예산 내에서 트랜잭션 전체를 재시도하세요.
- 데드락 직후 `SHOW ENGINE INNODB STATUS\G`를 캡처하세요. 이후 이벤트로
  덮어쓰여집니다.

### 큐 스타일 워커 클레임 (5.7)

`SKIP LOCKED`가 없으므로, 동시 워커는 같은 후보 행에 대해 잠금 대기가
발생할 수 있습니다. 허용 가능한 경우 한 행을 `FOR UPDATE`로 선점합니다:

```sql
START TRANSACTION;

SELECT id
FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE;

UPDATE jobs
SET status = 'processing', started_at = CURRENT_TIMESTAMP
WHERE id = ?;

COMMIT;
```

동시성이 더 필요하면 예를 들어 (1) `status` 전이를 원자적으로 하는
`UPDATE ... WHERE status = 'pending' ORDER BY ... LIMIT 1`과 영향 받은
행 수 확인, (2) 별도 큠 테이블·`GET_LOCK` 조합, (3) MySQL 8.0으로
업그레이드 후 `SKIP LOCKED` 도입 등을 검토하세요.

## 커넥션 풀

SQLAlchemy 예시:

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+mysqlconnector://app:secret@db.internal/app",
    pool_size=10,
    max_overflow=5,
    pool_timeout=30,
    pool_recycle=240,
    pool_pre_ping=True,
    connect_args={"connect_timeout": 5},
)
```

Node.js `mysql2` 예시:

```javascript
import mysql from 'mysql2/promise';

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
  enableKeepAlive: true,
  keepAliveInitialDelay: 30000,
});

const [rows] = await pool.execute(
  'SELECT id, total FROM orders WHERE account_id = ? LIMIT 50',
  [accountId],
);
```

애플리케이션 풀 리사이클은 서버의 `wait_timeout` 아래로 유지하세요. 서버가
`wait_timeout = 300`을 사용한다면 `pool_recycle`을 240초 정도로 두는 것이
일관적이며, `pool_pre_ping`은 여전히 네트워크 및 페일오버 이벤트에서
복구하는 데 도움이 됩니다.

## 진단

유용한 1차 점검 명령:

```sql
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS\G;
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
```

통제된 환경에서 슬로우 로그 활성화:

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

실행 계획 분석은 `EXPLAIN` / `EXPLAIN FORMAT=JSON`으로 수행하세요.
`EXPLAIN ANALYZE`는 MySQL 8.0.18+에서만 제공되며 실제 데이터를 읽습니다.

## 복제

읽기 복제본은 지연될 수 있습니다. 쓰기 직후 본인 쓰기 읽기(read-your-own-write)
경로, 체크아웃 플로우, 권한 확인, 또는 멱등성 키(idempotency-key) 읽기를
복제본으로 라우팅하지 마세요.

MySQL 5.7에서는 복제 상태 확인에 다음을 사용합니다:

```sql
SHOW SLAVE STATUS\G;
```

`SHOW REPLICA STATUS`는 **MySQL 8.0.22+** 용어입니다. 5.7에서는 지원되지
않습니다.

TCP 연결이 살아있는지뿐 아니라 복제본의 SQL 스레드 상태, IO 스레드 상태,
그리고 지연(lag)을 모니터링하세요.

## 보안

```sql
CREATE USER 'app'@'%' IDENTIFIED BY 'use-a-secret-manager';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';

ALTER USER 'app'@'%' REQUIRE SSL;

SELECT user, host
FROM mysql.user
WHERE user = '';

DROP USER IF EXISTS ''@'localhost';
DROP USER IF EXISTS ''@'%';
```

`DROP USER IF EXISTS`는 **MySQL 5.7.8+**에서 사용 가능합니다. 그보다
낮은 패치에서는 존재 여부를 확인한 뒤 `DROP USER`만 호출하세요.

보안 검토 포인트:

- 애플리케이션 사용자에게 `ALL PRIVILEGES` 또는 `*.*`를 부여하지 마세요.
- 트래픽이 호스트나 네트워크를 가로지를 때 애플리케이션 사용자에 대해 TLS를
  요구하세요.
- 자격 증명은 예시, 스크립트, 저장소 파일이 아닌 플랫폼 시크릿 매니저에
  저장하세요.
- 마이그레이션/관리자 사용자를 런타임 애플리케이션 사용자와 분리하세요.
- 성능을 튜닝하기 전에 공용 네트워크 노출과 바인드 주소를 감사하세요.

## 구성

전용 데이터베이스 호스트의 시작점 예시(MySQL 5.7):

```ini
[mysqld]
innodb_buffer_pool_size = 4G
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

max_connections = 300
thread_cache_size = 50

wait_timeout = 300
interactive_timeout = 300
innodb_lock_wait_timeout = 10

slow_query_log = ON
long_query_time = 1
log_queries_not_using_indexes = ON

log_bin = mysql-bin
binlog_format = ROW
expire_logs_days = 7
```

`binlog_expire_logs_seconds`는 **MySQL 8.0+** 변수입니다. 5.7에서는
`expire_logs_days`로 바이너리 로그 보존 기간을 설정합니다.

구성 값은 보편적인 프리셋이 아니라 검토를 위한 출발점으로 다루세요. 워크로드,
하드웨어, 백업 정책, 복구 목표를 기준으로 메모리, 커넥션, 로그 보존, 내구성
설정의 크기를 정하세요.

## 안티 패턴

| 안티 패턴 | 위험 | 더 나은 패턴 |
| --- | --- | --- |
| 핫 경로에서 `SELECT *` | 과다 페치와 깨지기 쉬운 클라이언트 | 명시적인 컬럼 선택 |
| 깊은 `OFFSET` 페이지네이션 | 선형 스캔과 느린 페이지 | 키셋 페이지네이션 |
| 외래 키 조인에 인덱스 없음 | 느린 조인과 락이 무거운 삭제 | FK 컬럼을 의도적으로 인덱싱 |
| 긴 트랜잭션 | 락 대기와 큰 언두 히스토리 | 작은 작업 단위로 커밋 |
| `mysql.user`에 직접 DML | 그랜트 테이블 손상 위험 | `CREATE USER`, `ALTER USER`, `DROP USER` 사용 |
| 관리자 권한이 있는 애플리케이션 사용자 | 큰 영향 반경 | 최소 권한 런타임 사용자 |
| `wait_timeout`을 초과하는 풀 리사이클 | 오래된 풀 커넥션 | 타임아웃 아래로 리사이클하고 pre-ping |
| 쓰기 직후의 복제본 읽기 | 오래된 사용자 대면 상태 | read-after-write 플로우를 프라이머리에 고정 |
| 5.7에서 `SKIP LOCKED` 가정 | 문법 오류 또는 잘못된 이식 | 5.7 문서화 패턴 또는 8.0 업그레이드 계획 |

## 출력 기대 사항

이 스킬이 리뷰에 사용될 때 반환할 내용:

1. MySQL 5.7(및 필요 시 패치 레벨) 가정.
2. 정확성, 락, 보안, 마이그레이션과 관련된 최고 위험 이슈.
3. 안전한 경로를 위한 정확한 SQL 또는 코드 변경 사항.
4. 검증 계획: `EXPLAIN` / `EXPLAIN FORMAT=JSON`, 마이그레이션 드라이런,
   락/데드락 체크, 롤백 기준.
5. 8.0으로 올릴 때 달라지는 문법·기능(`SKIP LOCKED`,
   `EXPLAIN ANALYZE`, 복제 용어, 바이너리 로그 보존 변수 등)이 리스크인지
   짧게 언급.

## 관련 자료

- 스킬: `postgres-patterns` - PostgreSQL 특화 스키마 및 쿼리 패턴
- 스킬: `database-migrations` - 마이그레이션 계획과 롤아웃 안전성
- 스킬: `backend-patterns` - API 및 서비스 계층 패턴
- 스킬: `security-review` - 시크릿 처리, 인증, 최소 권한
- 에이전트: `database-reviewer` - 더 넓은 데이터베이스 리뷰 워크플로
