## 쿼리
### 4.1 find 소개
쿼리는 컬렉션에서 도큐먼트의 서브셋을 반환한다. 어떠한 서브셋을 탐색할지는 첫 번재 파라미터에 의해 결정되는데, 나열한 키들은 AND 조건으로 검색된다.

```javascript
db.users.find({
    "username": "joe",
    "age": 27,
})
```


#### 4.1.1 반환받을 키 지정 
find 를 사용할 때, 원하는 key 만 반환받고 싶을 때에는 두 번째 인자를 통해 처리할 있다.

```javascript
db.users.find({}, {
    username: 1,
    email: 1,
})
```
이 때에, `_id` key 는 별도로 0 으로 설정하지 않는 이상, 항상 반환된다.

#### 4.1.2 제약사항 
데이터베이스에서 쿼리 도큐먼트 값은 반드시 상수여야 한다. 다른 키의 값을 동적으로 계산하는 등의 쿼리는 불가능하다. (`$where` 쿼리를 사용하면 복잡한 쿼리를 작성할 수 있다.)


### 4.2 쿼리 조건 
#### 4.2.1 쿼리 조건절 
`<. <=, >, >=` 에 해당하는 비교 연산자는 각각 `$lt, $lte, $gt, $gte` 이다.

```javascript
db.users.find({
    age: {
        $gte: 18,
        $lte: 30,
    }
})
```
> age 가 18 이상, 30 이하인 도큐먼트를 찾는 쿼리

Javascript Date 를 통해서도 비교 연산자 쿼리를 작성할 수 있다.
```javascript
start = new Date("01/01/2007")
db.users.find({
    registered: {
        $lt: start
    }
})
```

키 값이 특정 값과 일치하지 않는 도큐먼트를 찾는 데에는 `$ne` 를 사용할 수 있다.

```javascript
db.users.find({
    username: {
        $ne: "joe"
    }
})
```


#### 4.2.2 OR 쿼리
몽고DB의 OR 쿼리는 `$or`, `$in` 두 가지가 있다. `$or` 은 더 일반적이며, 여러 키를 주어진 값과 비교하는 쿼리에 사용한다.
하나의 키에 일치시킬 값이 여러 개 있다면 `$in` 에 조건 배열을 사용한다. 이 때, 조건 배열에 값이 하나만 있다면, 일반적인 find 쿼리와 동일한 동작으로 처리된다. NOT IN 조건으로는 `$nin` 을 사용할 수 있다.

```javascript
db.raffle.find({
    ticket_no: {
        $in: [1, 2, 3]
    }
})
```

`$or` 은 `$in` 보다 더 일반적인 경우에 사용할 수 있따. 예를 들어, 여러 컬럼의 조건들을 나열할 때 사용할 수 있다.

```javascript
db.raffle.find({
    $or: [
        {
            ticket_no: 725
        },
        {
            winner: true
        },
    ]
})
```
> ticket_no 가 725 이거나, winner 가 true 인 서브셋을 추출하는 쿼리

* AND 쿼리에서는 최소한의 인수로 최적의 결과(범위를 좁힌 결과)를 추려내야 한다. OR 쿼리는 반대인데, 첫 번째 인수가 일치하는 도큐먼트가 많을수록 효율적이다.
* `$or` 와 `$in` 연산자를 선택할 수 있다면 `$in` 을 사용하자. 쿼리 옵티마이저는 `$in` 을 더 효율적으로 다룬다.


#### 4.2.3 $not
`$not` 은 메타 조건절이며 어떤 조건에도 적용할 수 있다. 
```javascript
db.users.find({
    id_num: {
        $not: {
            $mod: [5, 1]
        }
    }
})
```
> id_num 를 5로 나는 결과가 1인 유저만 추출

`$not` 은 정규표현식과 같이 사용될 때 특히 유용하다.

### 4.3 형 특정 쿼리 
일부 데이터형은 쿼리 시 형에 특정하게(type-specific) 작동한다.

#### 4.3.1 null 
null 은 스스로와 일치하는 것을 찾는다. 하지만 null 은 `존재하지 않음` 과도 일치한다. 따라서 키가 null 인 값을 쿼리하면 해당 키를 갖지 않는 도큐먼트도 반환한다.

```javascript
db.c.find({
    z: null
})
```

만약 key 값이 정확히 `null` 인 값을 쿼리하고 싶으면 다음과 같이 작성해야 한다
```javascript
db.c.find({
    z: {
        $eq: null,
        $exists: true,
    }
})
```

추가로, `[null]` 이 데이터에 들어있는 경우에도 null 연산자에 포함된다. 이 때에도 `$exists` 절을 사용하여 쿼리를 해야 한다.
```javascript
db.family.find({'favorite food': {$eq: null}})
{
  _id: ObjectId("643a2989215348317c537df9"),
  name: 'Mipong',
  age: 3
}
{
  _id: ObjectId("643a2989215348317c537dfa"),
  name: 'Danpong',
  age: 30
}
{
  _id: ObjectId("643a2989215348317c537dfb"),
  name: 'Heepong',
  age: '30',
  'favorite food': [
    null
  ]
}
```
```javascript
db.family.find({'favorite food': {$eq: null, $exists: false}})
{
  _id: ObjectId("643a2989215348317c537df9"),
  name: 'Mipong',
  age: 3
}
{
  _id: ObjectId("643a2989215348317c537dfa"),
  name: 'Danpong',
  age: 30
}
```

#### 4.3.2 정규표현식 
`$regex` 는 쿼리에서 일치 문자열(pattern match string)을 위한 정규식 기능을 제공한다. 

```javascript
db.users.find({
    name: {
        $regex: /joe/i
    }
})
```

#### 4.3.3 배열에 쿼리하기 
배열 요소 쿼리는 스칼라 쿼리와 같은 방식으로 동작하도록 설계됐다.

```javascript
db.food.insertOne({
    fruit: ["apple", "banana", "peach"]
})
db.food.find({
    fruit: "banana"
})
```

#### $all 연산자
```javascript
db.food.insertOne({_id: 1, fruit: ["apple", "banana", "peach"]})
db.food.insertOne({_id: 1, fruit: ["apple", "kiwi", "orange"]})
db.food.insertOne({_id: 1, fruit: ["cherry", "banana", "apple"]})
```
2개 이상의 배열 요소가 일치하는 배열을 찾으려면 `$all` 을 사용한다.
```javascript
db.food.find({
    fruit: {
        $all: ["apple", "banana"]
    }
})
```
> fruit 필드에 "apple", "banana" 가 항상 들어있는 도큐먼트만 반환한다.

배열 내 특정 요소를 쿼리하려면 key.index 구문을 이용해 순서를 지정한다.
```javascript
db.food.find({
    "fruit.2": "peach"
})
```

#### $size 연산자
`$size` 는 특정 크기의 배열을 쿼리하는 유용한 조건절이다. 
```javascript
db.food.find({
    fruit: {
        $size: 3
    }
})
```
```javascript
db.food.update(criteria, {
    $push: {
        fruit: "strawberry"
    }
})
```

$size 는 다른 $조건절과 결합해 사용할 수 없지만, 도큐먼트에 "size" 키를 추가하면 이런 쿼리를 처리할 수 있다.
```javascript
db.food.update(criteria, {
    $push: {
        fruit: "strawberry"
    },
    $inc: {
        size: 1
    }
})
```

#### $slice 연산자
find의 두 번째 매개변수로 반환받을 특정 키를 지정할 수 있다. `$slice` 연산자를 사용해서 배열 요소의 부분집합을 반환받을 수 있다.

```javascript
db.blog.posts.findOne(criteria, {
    comments: {
        $slice: 10
    }
})
```
> 먼저 달린 댓글 10개를 반환받기

```javascript
db.blog.posts.findOne(criteria, {
    comments: {
        $slice: -10
    }
})
```
> 최근에 댓글 10개를 반환받기

offset 과 limit 을 활용해서 원하는 범위 안에 있는 결과를 반환할 수 있다.
```javascript
db.blog.posts.findOne(criteria, {
    comments: {
        $slice: [23, 10]
    }
})
```
> 처음 23개를 건너뛰고, 24번째 요소부터 33번째 요소까지 반환한다.


#### 일치하는 배열 요소의 반환 
$연산자를 사용하면 일치하는 요소를 반환받을 수 있다.
```javascript
db.blog.posts.find({
    "comments.name": "bob"
}, {
    "comments.$": 1
})
```

#### 배열 및 범위 쿼리의 상호작용
도큐먼트 내 스칼라(비배열 요소)는 쿼리 기준의 각 절과 일치해야 한다. 하지만 배열은 스칼라와 같이 탐색된다. 다음의 예시를 살펴보자

```
{x: 5}
{x: 15}
{x: 25}
{x: [5, 25]}
```

```javascript
db.test.find({
    x: {
        $gt: 10,
        $lt: 20,
    }
})
/**
{x: 15}
{x: [5, 25]}
*/
```

배열의 경우, 요소 중 하나라도 일치하는 값이 있으면 find 절에 true 로 반환된다. 원하는 결과를 얻으려면 다음의 방법을 사용해야 한다.

* $elemMatch
```javascript
db.test.find({
    x: {
        $elemMatch: {
            $gt: 10,
            $lt: 20
        }
    }
}) 
// 결과 없음
```

이 때, 배열이 아닌 요소는 검색되지 않는다. `$elemMatch` 는 배열 요소에 대한 범위 쿼리에 유용하다.

쿼리하는 필드에 인덱스가 있다면 min 함수와 max 함수를 이용해 `$lt`, `$gt` 값 사이로 인덱스 범위를 제한해 쿼리할 수 있다.

#### 4.3.4 내장 도큐먼트에 쿼리하기
다음과 같은 도큐먼트가 있다고 가정하자 
```
{
    name: {
        first: "Joe",
        last: "Schmoe"
    },
    age: 45
}
```

```javascript
db.people.find({
    name: {
        first: "Joe",
        last: "Schmoe"
    }
})
```
위와 같은 방법으로 쿼리하면 name 필드가 정확히 일치해야 한다 
```javascript
db.people.find({
    "name.first": "Joe",
    "name.last": "Schmoe",
})
```
내장 도큐먼트에 쿼리할 때는 가능하면 특정 키로 쿼리하는 방법이 좋다. 위와 같이 점 표기법으로 쿼리할 수 잇다. 다른 필드가 추가되어도 코드 변경 없이 쿼리할 수 있다.


다음과 같은 데이터가 있다고 가정하나 
```javascript
db.blog.find()
{
    content: "",
    comments: [
        {
            author: "joe",
            score: 3,
            comment: "nice post",
        },
        {
            author: "mary",
            score: 6,
            comment: "terrible post",
        },
    ]
}

db.blog.find({
    comments: {
        $elemMatch: {
            score: {
                $gte: 5
            },
            author: "joe"
        }
    }
})
```


### 4.4 $where 쿼리
키/값 쌍만으로 정확하게 표현할 수 없는 쿼리도 있다. 이 때 `$where` 를 이용하면 거의 모든 쿼리를 표현할 수 있다. 따라서 보안상의 이유로 $where 절 사용을 제한해야 한다.

```javascript
db.foo.insertOne({apple: 1, banana: 6, peach: 3})
db.foo.insertOne({apple: 8, spinach: 4, watermelon: 4})

db.foo.find({
    $where: function() {
        for(var current in this) {
            for(var other in this) {
                if(current != other && this[current] == this[other]) {
                    return true;
                }
            }
        }
        return false;
    }
})
```
where 절은 filter 함수가 처리되는 방식고 비슷하다. `$where` 쿼리는 일반 쿼리보다 느리니 반드시 필요한 경우가 아니면 사용하지 말자. `$where` 절 실행 시 도큐먼트는 BSON 에서 자바스크립트 객체로 변환되기 때문에 오래 걸린다.

### 4.5 커서
데이터베이스는 커서를 사용해 find 의 결과를 반환한다. find 를 호출할 때 셸이 데이터베이스를 즉시 쿼리하지 않으며 결과를 요청하는 쿼리를 보낼때까지 기다린다. 따라서 쿼리하기 전에 옵션을 추가할 수 있다. 또한 cursor 객체상의 거의 모든 메서드가 커서 자체를 반환하므로 옵션을 어떤 순서로든 이어쓸 수 잇다.

```javascript
var cursor = db.food.find().sort({x: 1}).limit(1).skip(10);
var cursor = db.food.find().limit(1).sort({x: 1}).skip(10);
var cursor = db.food.find().limit(1).sort({x: 1}).skip(10);

// 이 때 쿼리가 서버로 전송된다.
cursor.hasNext();
```
next 나 hasNext 메서드 호출 시, 셸은 처음 100개 또는 4MB 크기의 결과 중 작은것을 가져온다.


#### 4.5.1 제한, 건너뛰기, 정렬
```javascript
db.c.find().limit(3)
db.c.find().skip(3)
db.c.find().sort({username: 1, age: -1})
```

* 비교 순서
몽고DB 에는 데이터형을 비교하는 위계 구조가 있다. 데이터형 정렬 순서를 최솟값에서 최댓값 순으로 나타내면 다음과 같다.

1. 최솟값
2. null
3. 숫자
4. 문자열
5. 객체/도큐먼트
6. 배열
7. 이진 데이터
8. ObjectId
9. 불리언
10. 날짜
11. 타임스탬프
12. 정규표현식
13. 최댓값

#### 4.5.2 많은 수의 건너뛰기 피하기 
도큐먼트 수가 많아지면 skip 을 사요하면 성능이 느려진다. 

* skip 을 사용하지 않고 페이지 나누기
Date 를 내림차순으로 정렬해 도큐먼트를 표시한다고 가정하자.
```javascript
var page1 = db.foo.find().sort({date: -1}).limit(100)
while(page1.hasNext()) {
    latest = page1.next()
    display(latest)
} 

var page2 = db.foo.find({date: {$lt: latest.date}})
page2.sort({date: -1}).limit(100)
```


#### 4.5.3 종료되지 않는 커서 
서버 측에서 보면 커서는 메모리와 리소스를 점유한다. 커서가 더는 가져올 결과가 없거나 클라이언트로부터 종료 요청을 받으면 데이터베이스는 점유하고 있던 리소스를 해제한다. 서버 커서를 종료하는 조건이 몇 가지 있다.
* 서버는 조건에 일치하는 결과를 모두 살펴본 후에는 스스로 정리한다.
* 클라이언트측에서 유효 영역을 벗어나면 드라이버는 데이터베이스에 메세지를 보내 커서를 종료해도 된다고 알린다.
* 사용자가 아직 결과를 다 살펴보지 않았더라도, 10분동안 활동이 없으면 커서를 없앤다. (timeout)
  * 많은 드라이버에서는 커서를 종료시키지 못하게 하는 immortal 이라는 함수를 제공한다  