### 3.1 도큐먼트 삽입
#### 3.1.1 insertMany
Document 를 Bulk Insert 할 때 사용한다. 
* insertMany 사용시 유의사항
  * 메세지 크기 제한 
    * 현재 몽고DB 는 48mb 보다 큰 메세지를 허용하지 않으므로, 일괄 삽입할 수 있는 데이터의 크기에 제한이 있다. 48mb 보다 큰 데이터를 삽입하려 하면, 드라이버는 데이터를 48mb 크기로 나누어 추가한다. 
  * 정렬된 삽입
    * insertMany 의 두 번째 인자로 option 을 줄 수 있다. `ordered: true` 를 설정하면, insert 에 실패한 데이터의 다음 행부터는 insert 되지 않는다. 만약 `ordered: false` 가 설정되면, 몽고DB 는 모든 도큐먼트 삽입을 시도한다.

#### 3.1.2 삽입 유효성 검사
Document 가 삽입될 때, 다음과 같은 유효성 검사를 수행한다.

* `_id` 필드가 존재하지 않으면 새로 추가한다.
* 도큐먼트의 크기가 16mb 미반인지 검사한다.

#### 3.1.3 삽입
몽고DB 3.0 이전에는 데이터 삽입 함수가 `insert` 였는데, 3.0 이후로는 `insertOne`, `insertMany` 를 사용한다.

<hr />
## 3.2 삭제
```javascript
db.movies.deleteOne({
  _id: 4
})
```
### 3.2.1 drop
movies 컬렉션의 도큐먼트를 모두 삭제하려면 `db.movies.deleteMany({})` 를 사용할 수 있지만, `db.movies.drop()` 명령어를 사용하면, 더 빠르게 작업을 수행할 수 있다.

<hr />
## 3.3 도큐먼트 갱신
도큐먼트 갱신은 updateOne, updateMany, replaceOne 과 같은 갱신 매서드를 사용해 변경한다. 갱신은 원자적으로 처리된다. 갱신 요청 두 개가 동시에 발생하면 서버에 먼저 도착한 요청이 적용된 후 다음 요청이 적용된다. 따라서 여러 개의 갱신 요청이 빠르게 발생해도, 도큐먼트는 변질 없이 안전하게 처리된다. 

### 3.3.1 도큐먼트 치환(replaceOne)
```javascript
db.people.replaceOne({_id: ...}, joe);
```
와 같은 문법으로 Document 를 대체할 수 있다. 단, 조건절은 유일한 도큐먼트를 치환해야 하므로, `_id` 값을 조건절로 잡는 것이 좋다.

### 3.3.2 갱신 연산자
* `$inc`
  ```javascript
  // int, long, double, decimal 타입 값에만 사용 가능하다
  db.analytics.updateOne({
    url: "www.example.com"
    }, {
      "$inc": {
        pageviews: 1
      }
    })
  ```
* `$set`
  필드 값을 설정한다. 필드가 존재하지 않으면, 새 필드가 생성된다.
  ```javascript
    db.users.updateOne({
      _id: ObjectId("...")
    }, {
      "$set": {
        "favorite book": "..."
      }
    })
  ```
  ```javascript
    db.users.updateOne({
      "author.name": "joe"
    }, {
      "$set": {
        "author.name": "kim"
      }
    })
  ```
* `$unset`
  ```javascript
    db.users.updateOne({
      _id: ObjectId("...")
    }, {
      "$unset": {
        // 제거하려는 필드를 1로 표시하면 삭제된다
        "favorite book": 1
      }
    })
  ```
* `$inc`
이미 존재하는 키의 값을 변경하거나, 새 키를 생성하는데 사용한다. $inc 는 int, long, double, decimal 타입 값에만 사용할 수 있다.
  ```javascript
    db.games.updateOne({
      game: "pinball",
      user: "joe",
    }, {
      $inc: {
        score: 50
      }
    })
  ```
### 배열 연산자
* `$push`
배열을 다룰 때 갱신 연산자를 사용할 수 있다.
  ```javascript
    db.blog.posts.updateOne({
      title: "A blog post",
    }, {
      $push: {
        comments: {
          name: "joe",
          email: "joe@example.com",
          content: "nice post.",
        }
      }
    })
  ```
`$each` 제한자를 사용하면, 작업 한 번으로 값을 여러 개 추가할 수 있다. 
  ```javascript
    db.blog.posts.updateOne({
      title: "A blog post",
    }, {
      $push: {
        comments: {
          $each: [
            {
              name: "dana",
              email: "dana@example.com",
              content: "nice post.",
            },
            {
              name: "heejae",
              email: "heejae@example.com",
              content: "nice post.",
            },
          ]
        }
      }
    })
  ```
  `$slice` 제한자를 사용하면, 배열을 특정 크키 이상으로 늘어나지 않게 하고 효과적으로 top N 목록을 만들 수 있다
  ```javascript
    db.movies.updateOne({
      genre: "horror"
    }, {
      $push: {
        "top10": {
          $each: [
            "Nightmare on Elm Street",
            "Saw",
          ],
          // 10 으로 설정하면, 앞의 10개만 남게 된다.
          $slice: -10
        }
      }
    })
  ```
  `$sort` 제한자를 사용하면, 배열내 결과를 정렬할 수 있다.
  ```javascript
    db.movies.updateOne({
      genre: "horror"
    }, {
      $push: {
        top10: {
          $each: [
            {
              name: "Nightmare on Elm Street".
              rating: 6.6
            },
            {
              name: "Saw".
              rating: 4.3
            },
          ]
          $slice: -10,
          $sort: {
            ratimg: -1
          }
        }
      }
    })
  ```
  ### 배열을 집합으로 사용하기
  `$ne` 제한자를 사용하면, 특정 element 가 존재하는지 검사할 수 있다.
  ```javascript
    db.papers.updateOne({
      "authors cited": {
        $ne: "Richie"
      },
    }, {
      $push: {
        "authors cited": "Richie"
      }
    })
  ```
  `$addToSet` 제한자를 이용하면, 중복 element 를 제거할 수 있다.
  ```javascript
    db.users.updateOne({
      _id: ObjectId("...")
    }, {
      $addToSet: {
        email: "joe@hotmail.com"
      }
    })
  ```
  ### 요소 제거하기
  `$pop` 을 사용하면, 배열의 양 끝에 있는 요소들을 쉽게 제거할 수 있다.
  ```javascript
    db.blog.posts.update({
      $pop: {
        comment: -1
      }
    })
  ```
  `$pull` 을 사용하면, 주어진 조건에 맞는 배열 요소를 찾아 제거할 수 있다. `$pull` 은 도큐먼트에서 조건과 일치하는 요소를 모두 제거한다. 
  ```javascript
    db.lists.updateOne({}, {
      $pull: {
        todo: "laundry"
      }
    })
  ```
  `arrayFilters` 를 도입하면, 특정 조건에 맞는 배열 요소를 갱신할 수 있다.
  ```javascript
    // 댓글 추천 수가 5개 미만인 댓글은 숨김 처리한다
    db.blog.updateOne({
      post: 1,
    }, {
      $set: {
        "comments.$[elem].hidden": true
      }
    }, {
      arrayFilters: [
        {
          "elem.votes": {
          $lte: -5
        }
      ]
    })
  ```
### 3.3.3 갱신 입력
갱신 조건에 맞는 도큐먼트가 존재하면 갱신하고, 새로운 도큐먼트를 생성한다.
```javascript
db.analytics.updateOne({
  url: "/blog", {
    $inc: {
      pageviews: 1
    }
  }, {
  upsert: true
})
```
갱신 입력이 사용될 때, 기본값으로 지정하고 싶은 필드가 있다면 `$setOnInsert` 제한자를 사용할 수 있다.
```javascript
db.users.updateOne({}, {
  $setOnInsert: {
    createAt: new Date()
  }
}, {
  upsert: true,
})
```
### 3.3.4 다중 도큐먼트 갱신
`updateOne` 는 하나의 도큐먼트만 갱신할 수 있다. 만약 필터 조건에 여러 건이 해당되면, 최초의 한 건만 갱신된다. 만약 여러 건의 데이터를 갱신하고 싶다면, `updateMany` 를 사용한다.

### 3.3.5 갱신한 도큐먼트 반환
`findOneAndUpdate`는 Update 문에서 값을 수정한 후, 해당 값을 반환받게 해준다. 해당 작업은 수정 후의 결과를 atomic 하게 처리하는 데에 사용할 수 있다.
```javascript
  db.processes.findOneAndUpdate({
    status: "READY"
  }, {
    $set: {
      status: "RUNNING"
    }
  }, {
    sort: {
      priority: -1,
      returnNewDocument: true
    }
  })
```
유사하게, `findOneAndDelete`, `findOneAndReplace` 를 사용할 수 있다.