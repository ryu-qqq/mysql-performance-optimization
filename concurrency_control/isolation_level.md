# 트랜잭션 Isolation Levels (격리 수준)

Isolation(고립성)은 트랜잭션이 수행되는 동안 다른 트랜잭션과의 영향을 최소화하여 **데이터 정합성을 유지하는 역할**을 한다. 하지만 완벽한 Isolation을 적용하면 성능이 저하될 수 있기 때문에, **격리 수준(Isolation Level)**을 조정하여 **정합성과 성능의 균형**을 맞춘다.

---

## **1. Read Uncommitted (읽기 미커밋)**
- **특징:** 다른 트랜잭션이 **커밋되지 않은 변경 사항도 읽을 수 있음**.
- **발생 가능 문제:** Dirty Read, Non-Repeatable Read, Phantom Read, Read Skew, Write Skew
- **장점:** 성능이 가장 좋음.
- **단점:** 데이터 정합성이 보장되지 않음.

### **예제 (Dirty Read)**
```
T1: UPDATE a SET balance = 180 WHERE id = 1;  -- (커밋 안 함)
T2: SELECT balance FROM a WHERE id = 1;  -- (변경된 180을 읽음)
T1: ROLLBACK;  -- (T1 변경 사항 취소됨)
```
→ **T2는 존재하지 않는 데이터(180만원)를 기반으로 작업하게 됨.**

---

## **2. Read Committed (읽기 커밋됨)**
- **특징:** 다른 트랜잭션이 **커밋된 데이터만 읽을 수 있음**.
- **발생 가능 문제:** Non-Repeatable Read, Phantom Read, Read Skew, Write Skew
- **장점:** Dirty Read 방지
- **단점:** 같은 데이터를 다시 조회할 때 값이 바뀔 가능성이 있음.

### **예제 (Non-Repeatable Read)**
```
T1: SELECT balance FROM a WHERE id = 1;  -- (200만원 읽음)
T2: UPDATE a SET balance = 180 WHERE id = 1; COMMIT;
T1: SELECT balance FROM a WHERE id = 1;  -- (180만원으로 변경됨)
```
→ **T1이 같은 데이터를 두 번 읽었을 때 값이 다름.**

---

## **3. Repeatable Read (반복 읽기)**
- **특징:** **트랜잭션이 시작될 때 읽은 데이터가 트랜잭션이 끝날 때까지 동일함.**
  → `SELECT`로 읽은 데이터는 **다른 트랜잭션이 변경할 수 없음**.
- **발생 가능 문제:** Phantom Read, Write Skew
- **장점:** Non-Repeatable Read 방지
- **단점:** 새로운 행이 추가되거나 삭제되는 경우는 방지하지 못함.

### **예제 (Phantom Read)**
```
T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- (결과: 10건)
T2: INSERT INTO orders (status) VALUES ('PENDING'); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- (결과: 11건)
```
→ **T1이 같은 조건으로 데이터를 조회했을 때, 결과가 달라짐.**

---

## **4. Serializable (직렬화 가능)**
- **특징:** 트랜잭션을 완전히 분리하여 **동시에 실행되지 않도록 보장**.
  → **완벽한 Isolation을 제공하지만 성능 저하가 큼**.
- **발생 가능 문제:** 없음. **(모든 이상 현상 방지)**
- **장점:** 데이터 정합성이 완벽하게 보장됨.
- **단점:** 성능 저하가 심함 (Locking이 많아짐).

---

## **5. Snapshot Isolation (스냅샷 아이솔레이션)**
- **특징:** 트랜잭션이 **시작될 때의 데이터 스냅샷을 기반으로 실행**.
    - 트랜잭션이 진행되는 동안 **다른 트랜잭션이 커밋한 변경 사항은 보이지 않음**.
    - **MVCC (Multi-Version Concurrency Control)**을 사용하여 구현됨.
- **발생 가능 문제:** Write Skew 가능성 존재.
- **장점:** Dirty Read, Non-Repeatable Read 방지.
- **단점:** 일부 상황에서 Write Skew 문제가 발생할 수 있음.

### **예제 (Write Skew 발생 가능성)**
```
T1: SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- (2명)
T2: SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- (2명)
T1: UPDATE doctors SET on_call = false WHERE id = 1; COMMIT;
T2: UPDATE doctors SET on_call = false WHERE id = 2; COMMIT;
```
→ **스냅샷을 기반으로 트랜잭션이 독립적으로 실행되었으나, 최종적으로 모든 의사가 on-call에서 제외됨.**

### **해결 방법**
- **Serializable Isolation**을 적용하면 해결 가능하지만 성능이 저하될 수 있음.
- 애플리케이션 수준에서 Write Skew를 감지하고 처리해야 할 수도 있음.

---

## **정리**
| 격리 수준        | Dirty Read | Non-Repeatable Read | Phantom Read | Read Skew | Write Skew | 성능 |
|------------------|-----------|---------------------|--------------|-----------|------------|------|
| Read Uncommitted | 발생      | 발생                | 발생         | 발생      | 발생       | 가장 빠름 |
| Read Committed  | 방지      | 발생                | 발생         | 발생      | 발생       | 빠름 |
| Repeatable Read | 방지      | 방지                | 발생         | 발생      | 발생       | 중간 |
| Snapshot Isolation | 방지   | 방지                | 방지 가능    | 방지      | 발생 가능  | 중간 ~ 높음 |
| Serializable    | 방지      | 방지                | 방지         | 방지      | 방지       | 가장 느림 |

→ **높은 격리 수준을 사용할수록 성능 저하가 발생하지만, 데이터 정합성이 강하게 보장됨.**

