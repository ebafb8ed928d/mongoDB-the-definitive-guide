몽고DB 는 여러 작업, 컬렉션, 데이터베이스, 도큐먼트 및 샤드에서 ACID 호환 트랜잭션을 지원한다.

### 8.1 트랜잭션 소개 
트랜잭션은 읽기나 쓰기 작업이 가능한 데이터베이스 작업을 하나 이상 포함하는 데이터베이스의 논리적 처리 단위(unit of processing)다. 트랜잭션의 중요한 특징은 작업이 성공하든 실패하든 부분적으로는 완료되지 않는다는 점이다. 


#### 8.1.1 ACID 의 정의 
ACID 는 Atomicity(원자성)성 Consistency(일관성), Isolation(고립성), Durability(영속성) 의 약어다.

원자성은 트랜잭션 내 모든 작업이 적용되거나 아무 작업도 적용되지 않도록 한다. 즉, commit 되거나 롤백된다.

일관성은 트랜잭션이 성공하면 데이터베이스가 하나의 일관성 있는 상태에서 다음 일관성 있는 상태로 이동하도록 한다.

고립성은 여러 트랜잭션이 데이터베이스에서 동시에 실행되도록 허용하는 속성이다. 트랜잭션이 다른 트랜잭션의 부분 결과를 보지 않도록 보장한다. 즉 여러 병렬 트랜잭션이 각 트랜잭션을 순차적으로 실행할 때와 동일한 결과를 얻게 된다.

영속성은 트랜잭션이 커밋될 때 시스템 오류가 발생하더라도 모든 데이터가 유지되도록 한다.

몽고DB 는 복제 셋과 샤드 전체에 ACID 호환 트랜잭션이 있는 분산 데이터베이스다. 


### 8.2 트랜잭션 사용법 
몽고DB는 트랜잭션을 사용하기 위한 두 가지 API 를 제공한다. 첫 번째는 core API 라는 관계형 데이터베이스와 유사한 구문(`start_transaction`, `commit_transaction`) 이고, 두 번째는 트랜잭션 사용에 권장되는 접근 방식인 callback API 이다. 

코어 API 는 오류에 재시도 로직을 제공하지 않으며, 개발자가 작업에 대한 로직, 트랜잭션 커밋 함수, 필요한 재시도 및 오류 로직을 모두 작성해야 한다.

콜백 API 는 트랜잭션 시작, 콜백 함수로 제공된 함수 실행, 트랜잭션 커밋을 포함해 코어 API 에 비해 많은 기능을 래핑하는 단일 함수를 제공한다. 이 함수는 커밋 오류를 처리하는 재시도 로직도 포함한다.

| 코어 API  | 콜백 API  |
|---|---|
| 트랜잭션을 시작하고 커밋하려면 명시적인 호출 필요  | 트랜잭션을 시작하고 지정된 잭업을 실행한 후 커밋(혹은 롤백)  |

다음의 예제를 살펴보자

```javascript
orders.insert_one({
  sku: "abc123",
  qty: 100,
}, session=session);

inventory.update_one({
  sku: "abc123",
  qty: {
    $gte: 100
  }, {
    $inc: {
      qty: -100
    }
  }
}, session = session)
```

전자상거래 사이트에서 주문이 이루어지고 해당 품목이 판매되면 재고에서 제거된다.

### 8.3 어플리케이션을 위한 트랜잭션 제한 조정 
트랜잭션을 사용할 때 알아야 할 몇 가지 매개변수를 설명한다

#### 8.3.1 타이핑과 Oplog 크기 제한
몽고DB 트랜잭션에는 두 가지 주요 제한 범주가 있다.
* 트랜잭션의 시간 제한 
  * `트랜잭션이 실행될 수 있는 시간`, `트랜잭션이 락을 획득하려고 대기하는 시간`, `모든 트랜잭션이 실행될 최대 길이를 제어하는 것과 관련 있다.`
  * 트랜잭션의 실행 시간은 기본적으로 1분 이하다. 
    * `transactionLifetimeLimitSeconds` 변수로 제어할 수 있다.
    * 트랜잭션에 시간 제한을 명시적으로 설정하려면 commitTransaction 에 `maxTimeMS` 를 직접 지정할 수 있다.
  * 트랜잭션 락을 획득하는 데 대기하는 최대 시간은 기본적으로 5ms 이다.
    * 이 값은 maxTransactionLockRequestTimeoutMillis 에 의해 조정할 수 있다.  
* 트랜잭션이 실행될 최대 길이 (Oplog)
  * 몽고DB 는 트랜잭션의 쓰기 작업에 필요한 만큼 oplog 항목을 생성한다. 그러나 각 oplog 항목은 BSON 도큐먼트 크기 제한인 16MB 이하여야 한다.