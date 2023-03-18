#### RDBMS 와 비교하기 쉬운 MongoDB 개념들

| RDBMS | DocumentDB |
| --- | --- |
| Database | Database |
| Table | Collections |
| Instance | Documents |

<hr />

## 2.1 도큐먼트
* 몽고DB 의 핵심은 정렬된 키와 연결된 값의 집합으로 이루어진 도큐먼트이다.
  > 도큐먼트는 자바의 HashMap, 파이썬의 Dictionary, Javascript 의 object 와 같은 자료구조라 생각하고 넘어가자
* 도큐먼트의 키는 다음과 같은 성질을 만족한다
  * 키는 \0(0x00, null 문자) 를 포함하지 않는다. \0 은 키의 끝을 나타내는 데에 사용되기 때문이다.
  * `.`, `$` 는 보통 예약어로 취급해야 하며, 부적절하게 사용하면 드라이버에서 경고를 발생시킨다.
  * 위의 두 조건을 제외한 모든 UTF-8 문자는 키로 사용될 수 있다.
  * 몽고DB 의 키는 데이터형과 대소문자를 구별한다. 아래의 4 개의 도큐먼트는 모두 다르다.
  ```javascript
  {"count": 5}
  {"Count": 5}
  {"count": "5"}
  {"Count": "5"}
  ```
  * 몽고DB 에서는 키가 중복될 수 없다.

<hr />

## 2.2 컬렉션
컬렉션은 도큐먼트의 모음이다.

### 2.2.1 동적 스키마
기본적으로 컬렉션은 동적 스키마를 지닌다. 어플리케이션 스키마는 기본적으로 필요하지는 않지만, 정의하면 좋다. (왜인지는 직감적으로 알 수 있기 때문에 생략)
몽고DB 의 도큐먼트 유효성 검사기능과 객체-도큐먼트 매핑 라이브러리는 많은 프로그래밍 언어에서 사용 가능하다.

언어별 대표적인 ODM
| 언어 | Library | Link |
| --- | --- | --- |
| Javascript (Node.js) | mongoose | https://github.com/Automattic/mongoose | 
| Java | spring-data-mongodb | https://github.com/spring-projects/spring-data-mongodb | 
| Java | morphia | https://morphia.dev/landing/index.html | 
| Python | PyMongo | https://github.com/mongodb/mongo-python-driver | 
| Ruby | Mongoid | https://github.com/mongodb/mongoid | 

### 2.2.2 네이밍
컬렉션명은 어떤 UTF-8 문자열이든 쓸 수 있지만 몇 가지 제약조건이 있다.
* 빈 문자열("")은 유효한 컬렉션명이 아니다
* \0(null) 은 컬렉션명의 끝을 나타내는 문자이므로, 컬렉션명에 사용할 수 없다.
* `system.` 으로 시작하는 컬렉션명은 예약어이므로 사용할 수 없다.
* 사용자가 많은 컬렉션은 이름에 `$` 를 포함할 수 없다. (시스템에서 생성한 컬렉션에는 `$`가 들어갈 수 있다.)


#### 서브컬렉션
MongoDB 에서 서브컬렉션을 namespace 에 `.` 문자를 사용하여 컬렉셔능ㄹ 체계화한다. 서브컬렉션은 몽고DB 의 데이터를 체계화하는 훌륭한 방법이다.

> db.blog 는 blog 컬렉션을, db.blog.posts 는 blog.posts 컬렉션을 보여준다

<hr />

### 2.3 데이터베이스
한 몽고DB 인스턴스는 여러 데이터베이스를 호스팅할 수 있으며, 각 데이터베이스를 완전히 독립적으로 취급할 수 있다. 데이터베이스의 이름에는 어떤 UTF-8 문자열이든 쓸 수 있지만 몇 가지 제약조건이 있다.

* 빈 문자열("")은 유효한 데이터베이스 이름이 아니다.
* 데이터베이스 이름은 다음의 문자를 포함할 수 없다. `/.\'*<>:|?$`
* 데이터베이스 이름은 대소문자를 구별한다.
* 데이터베이스 이름은 최대 64바이트이다.

직접 접근할 수는 있지만 특별한 의미론(semantics)을 갖는 예약된 데이터베이스 이름도 있다

`admin`
  admin 데이터베이스는 authentication, authorization 역할을 한다. 또한 일부 작업을 하려면 admin 데이터베이스에 대한 접근이 필요하다.

`local`
  local 데이터베이스는 단일 서버에 대한 데이터를 저장한다. replica set 에서 local 은 복제 프로세스에 사용된 데이터를 저장한다. local 데이터베이스 자체는 복제되지 않는다.
  * local.startup_log
  ```javascript
    /**
     * 다음은 local 데이터베이스에 저장되는 collection 중 하나인 startup_log Collection 이다
     * 
     * @remarks
     * On startup, each mongod instance inserts a document into startup_log with diagnostic information about the mongod instance itself and host information.
     * startup_log is a capped collection. 
     * This information is primarily useful for diagnostic purposes.
     * @see https://www.mongodb.com/docs/manual/reference/local-database/
     */
    {
      "_id" : "<string>",
      "hostname" : "<string>",
      "startTime" : ISODate("<date>"),
      "startTimeLocal" : "<string>",
      "cmdLine" : {
            "dbpath" : "<path>",
            "<option>" : <value>
      },
      "pid" : <number>,
      "buildinfo" : {
            "version" : "<string>",
            "gitVersion" : "<string>",
            "sysInfo" : "<string>",
            "loaderFlags" : "<string>",
            "compilerFlags" : "<string>",
            "allocator" : "<string>",
            "versionArray" : [ <num>, <num>, <...> ],
            "javascriptEngine" : "<string>",
            "bits" : <number>,
            "debug" : <boolean>,
            "maxBsonObjectSize" : <number>
      }
    }
  ```

<hr />

### 2.4 몽고DB 시작
`mongod` 실행 파일을 실행하면, 몽고DB 서버를 시작할 수 있다.
별다른 argument 가 없다면, 기본 데이터 디렉토리로 `/data/db` 를 사용한다.

<hr />

### 2.5 몽고DB Shell
#### 2.5.1 Shell 실행
몽고DB 는 CLI 에서 인스턴스와 상호작용하는 Javascript Shell을 실행할 수 있다.  `mongo` 를 통해 Shell을 실행할 수 있다. [표준 자바스크립트](https://ko.wikipedia.org/wiki/ECMA%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8) 라이브러리의 모든 기능을 활용할 수 있다.


#### 2.5.2 몽고DB 클라이언트 
Shell 은 독자적으로 사용할 수 있는 몽고DB 클라이언트다. Shell 은 시작할 때 몽고DB 서버의 `test` 데이터베이스에 연결하고, 데이터베이스 연결을 전역 변수 `db` 에 할당한다.

또한 Shell SQL 사용자에게 친숙한 추가 기능을 포함한다. 예를 들어, `use` 명령어를 사용하면 데이터베이스를 선택할 수 있다. (이는 올바른 Javascript 문법은 아니다.)

#### 2.5.3 Shell 기본 작업(CRUD)

##### 생성
* insertOne
Collection 에 Document 를 추가한다.
```javascript
movie = {
  "title": "foo",
  "director": "icecreamparlor"
}
db.movies.insertOne(movie);
```
```
{
  "acknowledged": true,
  "insertedId: ObjectId("...")
}
```
```javascript
db.movies.find().pretty()
```
```
{
  _id: ObjectId("..."),
  title: "foo",
  director: "icecreamparlor
}
```

##### 읽기 
* find
* findOne
컬렉션에서 단일 도큐먼트를 읽을 때 사용한다.


##### 갱신 
* updateOne
```javascript
db.movies.updateOne({
  title: "foo"
}, {
  $set: {
    reviews: []
  }
});
db.movies.find().pretty()
```
```
{
  _id: ObjectId("..."),
  title: "foo",
  director: "icecreamparlor",
  reviews: []
}
```

##### 삭제 
* deleteOne
* deleteMany
```javascript
db.movies.deleteOne({
  title: "foo"
})
```

<hr />

### 2.6 데이터형
몽고DB 의 Document 는 [자바스크립트의 object](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Object) 와 닮았고, 자바스크립트의 object 가 JSON 과 닮았으므로, [JSON](http://www.json.org) 와도 닮았다. JSON 에서는 null, boolean, number, string, array, object 를형지원한다. 
JSON 에 없지만 중요한 데이터형도 있다. Date, float, regular expression 등이 이에 해당한다. 

#### 2.6.1 기본 데이터형
| Data Type | Description | 
| --- | --- | 
| null | null 값과 존재하지 않는 필드를 표현한다 | 
| boolean | 참과 거짓 값에 사용한다 | 
| number | 64비트 부동소수점 수를 기본으로 사용한다. | 
| string | UTF-8 문자열을 표현할 수 있다. | 
| Date | 1970년 1월 1일부터의 시간을 milliseconds 단위로 나타내는 64비트 정수로 저장한다. time zone 정보는 제외한다 | 
| regexp | 자바스크립트의 정규 표현식 문법을 사용할 수 있다. | 
| array | 값의 set, list 를 배열로 표현할 수 있다. | 
| nested document | 값의 set, list 를 배열로 표현할 수 있다. | 
| ObjectId | 12바이트 id 값이다 | 
| binary | 임의의 바이트 문자열이며, 셸에서는 조작이 불가능하다. UTF-8 이 아닌 문자열을 저장할 수 있다 | 
| code | 임의의 Javascript 코드를 저장할 수 있다. | 

#### 2.6.2 날짜 
몽고DB 의 날짜는 자바스크립트의 Date 를 사용한다. [ECMAScript 의 Date](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date) 문서를 참고

#### 2.6.3 배열 
배열은 정렬 연산(list, stack, queue) 와 비정렬 연산(set) 에 호환성 있게 사용 가능한 값이다. 배열 내부에 인덱스를 생성하여 쿼리 속도를 향상시킬 수 있다. 

#### 2.6.4 내장 도큐먼트 
몽고DB 는 내장 도큐먼트의 구조를 이해하고, 인덱스를 구성할 수 있다. 

#### _id 와 ObjectId 
몽고DB 의 모든 도큐먼트는 `_id` 키를 가진다. `_id` 는 어떠한 데이터형이어도 상관없지만, `ObjectId` 가 기본이다. ObjectId 는 12바이트 스토리지를 사용하며, 24자리 16진수 문자열 표현이 가능하다. 첫 4바이트는 timestamp, 다음 5바이트는 랜덤값, 마지막 3바이트는 counter 이다. 

<hr />
### 셸 활용 팁 
* `.mongorc.js`
다음과 같이 .mongorc.js 를 홈 디렉토리에 작성해 놓으면, 셸에서 dropDatabase, deleteIndexes 와 같은 함수를 무효화할 수 있다.

```javascript
const no = () => print("해당 함수는 실행할 수 없습니다");

/** DB 삭제 방지 */
db.dropDatabase = DB.prototype.dropDatabase = no;

/** 컬렉션 삭제 방지 */
DBCollection.prototype.drop = no;

/** 인덱스 삭제 방지 */
DBCollection.prototype.dropIndex = no;
DBCollection.prototype.dropIndexes = no;
```