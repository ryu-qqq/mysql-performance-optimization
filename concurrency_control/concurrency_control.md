# 트랜잭션 스케줄과 Conflict

## 1. 상황 설명
- **a 계좌**: 200만원 보유
- **b 계좌**: 200만원 보유

### 트랜잭션 발생
1. **트랜잭션 1 (T1)**
    - `a`의 잔액을 읽음
    - `a`의 잔액에서 20만원을 차감
    - `b`의 잔액을 읽음
    - `b`의 잔액에 20만원을 추가

2. **트랜잭션 2 (T2)**
    - `b`의 잔액을 읽음
    - `b`의 잔액에 30만원을 추가

트랜잭션 실행 스케줄 예시:
```
t1r(a) -> t1w(a) -> t1r(b) -> t1w(b) -> t2r(b) -> t2w(b)
```
이처럼 **트랜잭션 연산이 실행되는 순서**를 **스케줄(Schedule)**이라 한다.

---

## 2. Serial Schedule vs Non-serial Schedule

### **Serial Schedule (직렬 스케줄)**
- 한 트랜잭션이 완료된 후 다음 트랜잭션이 실행됨.
- **데이터 정합성**이 완벽히 보장됨.
- 하지만 **성능 저하** 문제 발생 (동시에 실행되는 작업이 없음).

### **Non-serial Schedule (비직렬 스케줄)**
- 여러 트랜잭션이 동시에 실행될 수 있음.
- **성능은 향상되지만, 데이터 정합성이 보장되지 않을 수도 있음.**
- 특정 연산의 순서가 바뀌면 결과가 달라질 수 있음.

---

## 3. Conflict와 Lost Update 문제

### **Conflict (충돌) 발생 조건**
트랜잭션 간에 **Conflict(충돌)**이 발생하는 경우는 다음과 같다:
- **동일한 데이터(X)를 두 개 이상의 트랜잭션(T1, T2)이 접근**할 때
- **최소한 하나의 트랜잭션이 해당 데이터를 WRITE(쓰기)할 경우**

### **Conflict 예시**
#### **Case 1: 정상적인 실행**
```
t1r(a)  200만원
t1w(a)  170만원
t1r(b)  200만원
t1w(b)  220만원
```
결과적으로 `a`의 잔액은 170만원, `b`의 잔액은 220만원이 되어 데이터 정합성이 유지됨.

#### **Case 2: Lost Update 문제**
```
t2r(b)  200만원
t1r(b)  200만원
t2w(b)  230만원
t1w(b)  220만원
```
여기서 `t2w(b)` 이후 `t1w(b)`가 덮어쓰면서 `T2`가 추가한 30만원이 사라지는 **Lost Update(갱신 손실) 문제** 발생.

---

## 4. Conflict Equivalent와 Serializability

### **Conflict Equivalent (충돌 등가)**
두 개의 스케줄이 같은 트랜잭션 집합을 가지고 있으며, **모든 Conflict Operation의 순서가 동일할 때** 두 스케줄은 **Conflict Equivalent(충돌 등가)** 하다고 한다.

### **Serializability (직렬 가능성)**
비직렬 스케줄에서도 **결과가 직렬 스케줄과 동일한 경우** 이를 **Serializable(직렬 가능)**하다고 한다.
- **Conflict Serializable**: Conflict Equivalent 관계를 이용해 직렬 가능성을 보장하는 방식
- **View Serializable**: 트랜잭션의 결과만 고려하여 직렬 가능성을 보장하는 방식

---

## 5. Recoveryability (복구 가능성)

### **Recoveryable Schedule (복구 가능 스케줄)**
- 트랜잭션이 실행되다가 **어떤 트랜잭션이 롤백되었을 때, 해당 트랜잭션에 의존하는 다른 트랜잭션도 함께 롤백될 수 있도록 보장**하는 성질을 갖는다.
- 만약 **복구 가능하지 않은 스케줄(Non-Recoveryable Schedule)**이 존재한다면, 시스템 충돌 시 데이터 불일치가 발생할 수 있다.

### **Recoveryability 예시**
#### **Case 1: 복구 가능한 스케줄**
```
t1r(a)  200만원
t1w(a)  170만원
t1c    (T1 커밋)
t2r(a)  170만원
t2w(a)  200만원
t2c    (T2 커밋)
```
- `T1`이 먼저 커밋되므로 `T2`는 `T1`의 결과를 보고 안전하게 실행됨.
- 시스템 충돌 시 `T1`이 롤백되면 `T2`도 롤백될 수 있음 → **Recoveryable Schedule**

#### **Case 2: 복구 불가능한 스케줄 (Non-Recoveryable)**
```
t1r(a)  200만원
t2r(a)  200만원
t2w(a)  230만원
t2c    (T2 커밋)
t1w(a)  170만원
t1r(b)  200만원
t1w(b)  220만원
t1a    (T1 중단 및 롤백)
```
- `T2`가 `T1`의 데이터를 보고 변경했는데, `T1`이 나중에 롤백됨.
- **`T2`는 `T1`의 롤백을 알지 못하고 커밋됨 → 데이터 불일치 발생**
- **해결책:** `T1`이 커밋되기 전에 `T2`가 커밋되지 않도록 강제하는 메커니즘 필요.

---

## 6. 트랜잭션 이상현상 (Anomalies)

### **Dirty Read (더티 리드)**
- **T1**이 데이터 A를 변경하고 커밋하지 않은 상태에서 **T2**가 이 변경된 데이터를 읽음.
- 이후 **T1**이 롤백되면 **T2**는 존재하지 않는 데이터를 기반으로 작업하게 됨.

예시:
```
T1: UPDATE a SET balance = 180 WHERE id = 1;  -- (커밋 안 함)
T2: SELECT balance FROM a WHERE id = 1;  -- (변경된 180을 읽음)
T1: ROLLBACK;  -- (T1 변경 사항이 취소됨)
```
- **T2는 존재하지 않는 데이터(180만원)를 기반으로 연산**하게 됨.

### **Non-Repeatable Read (비반복적 읽기)**
- **T1**이 데이터를 읽은 후 **T2**가 해당 데이터를 변경하고 커밋하면, **T1이 같은 데이터를 다시 읽을 때 값이 달라지는 현상**.

예시:
```
T1: SELECT balance FROM a WHERE id = 1;  -- (200만원 읽음)
T2: UPDATE a SET balance = 180 WHERE id = 1; COMMIT;
T1: SELECT balance FROM a WHERE id = 1;  -- (180만원으로 변경됨)
```
- **T1이 같은 데이터를 다시 조회했을 때 값이 다르게 나옴**.

### **Phantom Read (팬텀 리드)**
- **T1이 특정 조건을 만족하는 여러 행을 조회했을 때, 다른 트랜잭션(T2)이 데이터를 추가하면, T1이 동일한 조회를 수행할 때 추가된 데이터가 나타나는 현상**.

예시:
```
T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- (결과: 10건)
T2: INSERT INTO orders (status) VALUES ('PENDING'); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- (결과: 11건)
```
- **T1이 처음 조회한 결과와 두 번째 조회한 결과가 다름**.

### **Dirty Write (더티 라이트)**
- **T1이 데이터를 변경하고 커밋하지 않은 상태에서 T2도 같은 데이터를 변경하는 경우**.
- 이후 **T1이 롤백하면 T2가 변경한 값도 의미 없는 값이 됨**.

예시:
```
T1: UPDATE a SET balance = 180 WHERE id = 1;  -- (커밋 안 함)
T2: UPDATE a SET balance = 160 WHERE id = 1; COMMIT;
T1: ROLLBACK;  -- (T1 변경 사항 취소됨)
```
- **T2는 T1의 변경 사항을 모른 채 작업을 수행하므로 데이터 일관성이 깨질 수 있음**.

### **Read Skew (읽기 왜곡)**
- **T1이 여러 개의 관련 데이터를 읽은 후, T2가 일부 데이터를 변경 및 커밋한 경우, T1이 다시 조회할 때 불일치가 발생하는 현상**.

예시:
```
T1: SELECT balance FROM a WHERE id = 1;  -- (200만원 읽음)
T1: SELECT balance FROM b WHERE id = 2;  -- (200만원 읽음)
T2: UPDATE a SET balance = 150 WHERE id = 1; COMMIT;
T2: UPDATE b SET balance = 250 WHERE id = 2; COMMIT;
T1: SELECT balance FROM a WHERE id = 1;  -- (150만원으로 변경됨)
T1: SELECT balance FROM b WHERE id = 2;  -- (여전히 200만원으로 읽음)
```
- **T1이 연관된 데이터를 읽는 동안, T2가 일부만 변경하여 불일치 발생**.

### **Write Skew (쓰기 왜곡)**
- **두 개 이상의 트랜잭션이 서로 독립적으로 데이터를 읽고 조건에 따라 업데이트하지만, 최종적으로 일관성이 깨지는 현상**.

예시:
```
T1: SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- (2명)
T2: SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- (2명)
T1: UPDATE doctors SET on_call = false WHERE id = 1; COMMIT;
T2: UPDATE doctors SET on_call = false WHERE id = 2; COMMIT;
```
- **결과적으로 모든 의사가 on-call 상태가 아니게 되며, 병원 운영에 문제 발생**.

---




## 7. Isolation의 필요성
비직렬 스케줄을 사용할 때 발생할 수 있는 문제(Conflict, Lost Update, Recoveryability 등)를 해결하기 위해 **트랜잭션의 고립성(Isolation)**이 필요하다.

Isolation은 트랜잭션이 실행되는 동안 **다른 트랜잭션의 영향을 최소화하여 데이터 정합성을 유지**하도록 보장하는 개념이다.

(다음 내용에서는 Isolation 레벨에 대해 다룬다.)

