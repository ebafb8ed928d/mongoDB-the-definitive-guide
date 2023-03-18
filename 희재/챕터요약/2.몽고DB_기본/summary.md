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