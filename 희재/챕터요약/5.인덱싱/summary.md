### 5.1 인덱싱 소개
인덱싱을 사용하지 않는 쿼리를 컬렉션 스캔(collection scan) 이라 하며, 서버가 쿼리 결과를 찾으려면 "전체 내용을 살펴봐야 함" 을 의미한다. 다음의 쿼리를 살펴보자 

```javascript
db.users.find({
  username: "user101"
}).explain("executionStats")
```

explain 함수를 이용해 쿼리가 실행될 때 몽고DB 가 무엇을 하는지 확인할 수 있다. explain 은 명령을 감싸는 커서 보조자 메서드(cursor helper method) 와 사용하면 좋다. explain 커서 메서드는 다양한 CRUD 작업의 실행 정보를 제공한다. 이 메서드는 여러 가지 장황한 모드에서 실행할 수 있다. `executionStats` 모드는 인덱스를 이용한 쿼리의 효과를 이해하는 데 도움이 된다. 

위의 쿼리에 대한 결과는 다음과 같이 나타난다

```javascript
{
  queryPlanner: {
    plannerVersion: 1,
    namespace: "test.users",
    indexFilterSet: false,
    parseQuery: {
      username: {
        $eq: "user101"
      }
    },
    winningPlan: {
      stage: "COLLSCAN",
      filter: {
        username: {
          $eq: "user101
        }
      },
      direction: "forward"
    },
    rejectedPlans: [],
  },
  executionStats: {
    totalDocsExamined: 1000000,
    nReturned: 1,
    executionTimeMillis: 419,
  }
}
```

이 도큐먼트에서 `executionStats` 필드의 값인 중첩된 도큐먼트를 살펴보자. 

* `executionStats.totalDocExamined` : 쿼리를 실행하면서 살펴본 도큐먼트 개수
* `executionStats.executionTimeMillis` : 쿼리를 수행하는 데 소요된 시간
* `executionStats.nReturned` : 반환받은 결과의 개수를 보여준다.

몽고DB가 쿼리에 효율적으로 응답하게 하려면 어플리케이션의 모든 쿼리 패턴에 인덱스를 사용한다. 쿼리 패턴이란 단순히 어플리케이션이 데이터베이스에 요구하는 다양한 유형의 질문을 의미한다.


#### 5.1.1 인덱스 생성 
인덱스를 만드려면 createIndex 클렉션 메서드를 사용한다.

```javascript
db.users.createIndex({
  username: 1
})
```

이제 처음의 쿼리를 다시 실행시켜 보자 

```javascript
db.users.find({
  username: "user101"
}).explain("executionStats")

{
  queryPlanner: {
    winningPlan: {
      stage: "FETCH",
      inputStage: {
        stage: "IXSCAN",
        indexName: "username_1"
      },
      ...
    }
  },
  executionStats: {
    nReturned: 1,
    executionTimeMillis: 1,
    totalDocsExamined: 1,
  }
}
```

인덱스는 쿼리 시간에 놀라운 차이를 만들지만, 쓰기 작업은 더 오래 걸리게 한다. 데이터가 변경될 때마다 도큐먼트 뿐만아니라 인덱스를 갱신해야 하기 때문이다.


#### 5.1.2 복합 인덱스 소개 
상당수의 쿼리 패턴은 두 개 이상의 키를 기반으로 인덱스를 작성해야 한다. 앞의 예에서, `username` 인덱스는 다음의 정렬에 큰 도움이 되지 않는다.

```javascript
db.users.find().sort({
  age: 1,
  username: 1,
})
```

먼저 age 로 정렬한 후에 username 으로 정렬하는데, username 키만을 위한 정렬은 도움이 되지 않는다. 정렬을 최적화하려면 age 와 username 에 인덱스를 만든다.

```javascript
db.users.createIndex({
  age: 1,
  username: 1,
})
```

이를 `복합 인덱스` 라 부르며, 쿼리에서 정렬 방향이 여러 개이거나 검색 조건에 여러 개의 키가 있을 때 유용한다. 복합 인덱스는 2개 이상의 필드로 구성된 인덱스다.

정렬 없는(순차 정렬) 쿼리를 실행하면 다음처럼 보이는 user 컬렉션이 있다고 가정하자.

```javascript
db.users.find({}, {_id: 0, i: 0, created: 0});

/*
{ username: "user0", age: 69 }
{ username: "user1", age: 69 }
{ username: "user2", age: 69 }
{ username: "user3", age: 69 }
{ username: "user4", age: 69 }
{ username: "user5", age: 69 }
{ username: "user6", age: 69 }
{ username: "user7", age: 69 }
{ username: "user8", age: 69 }
{ username: "user9", age: 69 }
{ username: "user10", age: 69 }
*/
```

컬렉션에 `{ age: 1, username:1 }` 로 인덱스를 만들면, 인덱스는 다음과 같은 형태로 표현된다.

```javascript
[0, "user10020"] -> 8623513776
[0, "user1002"] -> 8623513777
[0, "user100388"] -> 8623518809
...
[1, "user100414"] -> 862351123
...
[2, "user10097"] -> 8623513123
```

각 인덱스 항목은 나이와 사용자명을 포함하고 레코드 식별자(record identifier)를 가리킨다. 레코드 식별자는 내부에서 스토리지 엔진에 의해 사용되며 도큐먼트 데이터를 찾는다. "age" 필드는 완전한 오름차순으로 정렬되며, 각 나이에서 "username" 역시 오름차순으로 정렬된다.

몽고DB 가 실행하는 쿼리의 종류에 따라 인덱스를 사용하는 방법이 다르다. 가장 많이 사용하는 세 가지 방법을 알아보자

```javascript
db.users.find({age: 21}).sort({usernmae: -1})
```
> 단일 값을 찾는 동등 쿼리(equality query) 이다. 결괏값으로 여러 도큐먼트가 있을 수 있다. 인덱스의 두 번째 필드로 인해 결과는 이미 적절한 순서로 정렬된다. 즉 몽고DB 는 `{age: 21}` 과 일치하는 마지막 항목부터 순서대로 인덱스를 탐색한다.
몽고DB 는 인덱스를 어느 방향으로도 쉽게 탐색하므로 정렬 방향은 문제가 되지 않는다.

```javascript
db.users.find({
  age: {
    $gte: 21,
    $lte: 30,
  }
})
```
> 범위 쿼리이며 여러 값이 일치하는 도큐먼트를 찾아낸다. 몽고DB 는 인덱스에 있는 첫 번째 키인 `age` 를 다음처럼 사용해 일치하는 도큐먼트를 반환받는다. 몽고DB 가 인덱스를 사용해 쿼리하면 일반적으로 인덱스 순서에 따라 도큐먼트 결과를 반환한다.


```javascript
db.users.find({
  age: {
    $gte: 21,
    $lte: 30
  }
}).sort({
  username: 1
})
```
> 마찬가지로 다중값 쿼리 (multivalue query) 이지만, 이번에는 정렬을 포함한다. 이전처럼 몽고DB 는 검색 조건에 맞는 인덱스를 사용한다. 하지만 인덱스는 사용자명을 정렬된 순서로 반환하지 않으며, 쿼리는 사용자명에 따라 정렬된 결과를 요청한다. 결과를 반환하기 전에 메모리에서 정렬해야 함을 의미한다. 따라서 일반적으로 이전 쿼리보다 비효율적이다. 만약 정렬해야 하는 결과가 32MB 이상이면, 몽고DB 는 데이터가 너무 많아 정렬을 거부한다는 오류를 내보낸다.

예제에는 같은 키를 역순으로 한 `{username: 1, age: 1}` 인덱스 또한 사용할 수 있다. 이때 몽고DB는 모든 인덱스 항목을 탐색하지만 원하는 순서도 되돌린다. 인덱스의 "age" 부분을 이용해서 일치하는 도큐먼트를 가져온다. 이는 거대한 인메모리 정렬이 필요하지 않다는 장점이 있다. 하지만 일치하는 값을 모두 찾으려면 전체 인덱스를 훑어야 한다. `따라서 복합 인덱스를 구성할 때에는 정렬 키를 첫 번째에 놓으면 좋다.` 이 방법은 동등 쿼리, 다중값 쿼리, 정렬을 고려해 복합 인덱스를 구성할 때의 모범 사례다.


#### 5.1.3 몽고DB 가 인덱스를 선택하는 방법 
쿼리가 들어오면 몽고DB 는 쿼리 모양(query shape)을 확인한다. 모양은 검색할 필드와 정렬 여부 등 추가 정보와 관련이 있다. 쿼리가 들어오고 인덱스 5개 중 3개가 쿼리 후보로 식별됐다고 가정해보자. 몽고DB 는 각 인덱스 후보에 하나씩 총 3개의 쿼리 플랜(query plan) 을 만들고, 각각 다른 인덱스를 사용하는 3개의 병렬 스레드에서 쿼리를 실행한다. 어떤 스레드에서 가장 빨리 결과를 반환받는지 확인하기 위함이다.

`이 과정은 race 와 같다. 가장 먼저 목표 상태에 도달하는 query plan 이 승자가 된다. 앞으로 동일한 모양을 가진 쿼리에 사용할 인덱스로 선택된다는 점이 중요하다.` 서버는 쿼리 플랜의 캐시를 유지하는데, 승리한 플랜은 차후 모야잉 같은 쿼리에 사용하기 위해 캐시에 저장된다. `시간이 지나 컬렉션과 인덱스가 변경되면 쿼리 플랜이 캐시에서 제거되고, 몽고DB 는 다시 가능한 쿼리 플랜을 실험해 해당 컬렉션 및 인덱스 집합에 가장 적합한 플랜을 찾는다.` 쿼리 플랜 캐시는 명시적으로 지울 수 있으며, mongod 프로세스를 다시 시작할 때도 삭제된다.