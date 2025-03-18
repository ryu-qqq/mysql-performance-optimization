
# MySQL 성능 최적화 - 논리적 아키텍처

## 1. MySQL 잠금(Lock) 개요
MySQL에서 데이터를 안전하게 읽고 쓰기 위해 다양한 잠금(lock) 메커니즘이 존재한다. 잠금 수준에 따라 동시성과 성능이 크게 영향을 받으므로, 이를 정확히 이해하는 것이 중요하다.

### 1.1 잠금 세분화
MySQL에서는 **잠금 범위**를 어떻게 설정하느냐에 따라 성능과 동시성에 차이가 발생한다.

- **테이블 잠금 (Table Lock)**: 전체 테이블을 잠그는 방식으로, **MyISAM 엔진에서 주로 사용**되며, 단순하지만 동시성이 낮음.
- **행 잠금 (Row Lock)**: 특정 행만 잠그는 방식으로, **InnoDB 엔진에서 사용**되며, 동시성이 높음.


---

## 2. MySQL의 트랜잭션 격리 수준 (Isolation Level)
MySQL에서는 트랜잭션 간 충돌을 방지하고 데이터 정합성을 유지하기 위해 **격리 수준(Isolation Level)**을 제공한다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|------------|----------------------|--------------|
| **READ UNCOMMITTED** | O | O | O |
| **READ COMMITTED** | X | O | O |
| **REPEATABLE READ** (기본값) | X | X | O |
| **SERIALIZABLE** | X | X | X |

### 2.1 READ UNCOMMITTED (읽기 미완료 허용)
🚨 **Dirty Read(더러운 읽기) 발생 가능**
- 커밋되지 않은 데이터를 다른 트랜잭션에서 읽을 수 있음.
- 성능은 좋지만, 데이터 정합성이 보장되지 않음.

### 2.2 READ COMMITTED (커밋된 데이터만 읽기)
🚨 **Non-Repeatable Read(비일관성 읽기) 발생 가능**
- 다른 트랜잭션이 커밋한 데이터는 즉시 반영됨.
- 같은 트랜잭션에서 조회할 때 값이 바뀔 수 있음.


### 2.3 REPEATABLE READ (반복 읽기 보장) (MySQL 기본값)
🚨 **Phantom Read(팬텀 리드) 발생 가능**
- 트랜잭션 내에서 동일한 `SELECT` 쿼리를 실행하면 같은 결과를 보장하지만, **새로운 행이 삽입되면 반영됨.**


### 2.4 SERIALIZABLE (직렬 실행)
✅ **완전한 일관성 보장, BUT 성능 저하**
- 모든 트랜잭션을 순차적으로 실행하여 충돌을 방지함.
- 동시성이 낮아 성능이 크게 저하될 수 있음.

---

## 3. Phantom Read (팬텀 리드)와 Next-Key Lock (넥스트 키 락)

### 3.1 팬텀 리드(Phantom Read)란?
- **같은 트랜잭션 내에서 동일한 SELECT 문을 실행했을 때, 처음에는 없던 데이터가 두 번째 실행에서 갑자기 튀어나오는 현상.**

📌 **팬텀 리드 예제** (PC방 자리 예약 시스템)
1. **트랜잭션 1 (사용 가능한 자리 개수 조회)**
```sql
START TRANSACTION;
SELECT COUNT(*) FROM seats WHERE status = 'available'; -- 결과: 5개
```
2. **트랜잭션 2 (다른 사용자가 좌석 추가)**
```sql
INSERT INTO seats (seat_id, status) VALUES (6, 'available');
COMMIT;
```
3. **트랜잭션 1에서 다시 조회**
```sql
SELECT COUNT(*) FROM seats WHERE status = 'available'; -- 결과: 6개 (팬텀 리드 발생)
COMMIT;
```
🚨 **팬텀 리드 발생!** 같은 트랜잭션에서 동일한 쿼리를 실행했는데, 다른 트랜잭션이 추가한 데이터가 보이는 현상.

### 3.2 팬텀 리드 해결법
1. **SERIALIZABLE 격리 수준으로 변경** (성능 저하)
2. **SELECT ... FOR UPDATE 사용** (INSERT 방지는 불가능)
3. **Next-Key Lock 사용** (가장 효과적)

### 3.3 Next-Key Lock (넥스트 키 락)이란?
- **기존 행뿐만 아니라 행과 행 사이의 빈 공간(Gap)까지 포함하여 잠금.**
- **새로운 행이 INSERT 되는 것을 방지하여 팬텀 리드를 해결함.**
- **행을 포함한 전체 범위에 락을 걸어 UPDATE, DELETE도 보호 가능.**

📌 **Next-Key Lock 적용 예제**
```sql
START TRANSACTION;
SELECT * FROM seats WHERE status = 'available' FOR UPDATE;
```
✅ 이제 `status='available'` 조건을 만족하는 행뿐만 아니라, 그 범위의 행까지 잠김! 🚀

---

## 4. Next-Key Lock vs 일반 Row Lock 차이점

| 구분 | Row Lock (행 잠금) | Next-Key Lock (넥스트 키 락) |
|------|-----------------|--------------------------|
| **잠그는 범위** | 특정 행(Row)만 잠금 | **행(Row) + 그 앞뒤의 빈 공간(Gap)까지 잠금** |
| **목적** | **특정 행 보호** | **INSERT 막기 + 기존 데이터 수정(UPDATE, DELETE) 보호** |
| **INSERT 차단 여부** | ❌ NO | ✅ YES |
| **UPDATE 차단 여부** | ✅ YES | ✅ YES |
| **DELETE 차단 여부** | ✅ YES | ✅ YES |

✅ **즉, Next-Key Lock은 "Row Lock + Gap Lock"을 포함한 개념이다!**
✅ **MySQL InnoDB는 기본적으로 Next-Key Lock을 사용하여 팬텀 리드 방지 + 동시성 충돌 방지**
✅ **만약 동시성을 높이려면 Next-Key Lock을 비활성화하고 Record Lock만 사용할 수도 있음!**

---
