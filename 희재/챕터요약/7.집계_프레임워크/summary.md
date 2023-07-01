### 7.1 파이프라인, 단계 및 조정 가능 항목

- 집계 프레임워크는 몽고DB 내 분속 도구 모음으로, 하나 이상의 컬렉션에 있는 도큐먼트에 대한 분석을 수행하게 해준다.
- 집계 프레임워크는 파이프라인 개념을 기반으로 한다. 우리는 집계 파이프라인을 통해 몽고DB 컬렉션에서 입력을 받고, 컬렉션에서 나온 도큐먼트를 하나 이상의 단계를 거쳐 전달한다.
- 집계 파이프라인은 bash 와 같은 셸 파이프라인과 매우 유사한 개념이며, 단계마다 특정 작업을 수행한다.
- 각 단계는 `knobs` 또는 `tunables` 셋을 제공한다. 이 항목들을 조정해 각 단계를 매개변수로 지정함으로써 원하는 작업을 수행할 수 있다.

> 함수형 프로그래밍의 함수 합성처럼, 데이터 누산 과정에서 파이프라인을 통해 집계 기능을 만드는 듯 하다.

### 7.2 단계 시작하기: 익숙한 작업들

집계 파이프라인 개발을 시작하기 위해 이미 우리에게 익숙한 작업과 관련된 몇몇 파이프라인의 구축을 살펴본다.

> 일치(match), 선출(projection), 정렬(sort), 건너뛰기(skip), 제한(limit) 단계를 살펴보자

다음의 예시 데이터를 살펴보자

```json
{
  "_id": "52cdef7c4bab8bd675297d8e",
  "name": "Facebook",
  "category_code": "social",
  "founded_year": 2004,
  "description": "Social network",
  "relationships": [
    {
      "is_past": false,
      "title": "Founder and CEO, Board Of Directors",
      "person": {
        "first_name": "Mark",
        "Last_name": "Zuckerberg",
        "permalink": "mark-zuckerberg"
      }
    },
    {
      "is_past": true,
      "title": "CFO",
      "person": {
        "first_name": "David",
        "last_name": "Ebersman",
        "permalink": "david-ebersman"
      }
    }
  ],
  "funding_rounds": [
    {
      "id": 4,
      "round_code": "b",
      "raised_amount": 27500000,
      "raised_currency_code": "USD",
      "funded_year": 2006,
      "investments": [
        {
          "company": null,
          "financial_org": {
            "name": "Greylock Partners",
            "permalink": "greylock"
          },
          "person": null
        },
        {
          "company": null,
          "financial_org": {
            "name": "Meritech Capital Partners",
            "permalink": "meritech-capital-partners"
          },
          "person": null
        },
        {
          "company": null,
          "financial_org": {
            "name": "Founders Fund",
            "permalink": "founders-fund"
          },
          "person": null
        },
        {
          "company": null,
          "financial_org": {
            "name": "SV Angel",
            "permalink": "sv-angel"
          },
          "person": null
        }
      ]
    },
    {
      "id": 2197,
      "round_code": "c",
      "raised_amount": 15000000,
      "raised_currency_code": "USD",
      "funded_year": 2008,
      "investments": [
        {
          "company": null,
          "financial_org": {
            "name": "European Founders Fund",
            "permalink": "european-founders-fund"
          },
          "person": null
        }
      ]
    }
  ],
  "ipo": {
    "valuation_amount": NumberLong("104000000000"),
    "valuation_currency_code": "USD",
    "pub_year": 2012,
    "pub_month": 5,
    "pub_day": 18,
    "stock_symbol": "NASDAQ:FB"
  }
}
```

```javascript
db.companies.aggregate([
  {
    $match: {
      founded_year: 2004,
    },
  },
]);
```

이는 find 를 사용하는 다음 작업과 동일하다.

```javascript
db.companies.find({
  founded_year: 2004,
});
```

다음의 예시 를 또 살펴보자

```javascript
db.companies.aggregate([
  {
    $match: {
      founded_year: 2004,
    },
    {
      $project: {
        _id: 0,
        name: 1,
        founded_year: 1,
      }
    }
  }
])
```

실행하면 `name`, `founded_year` 만 추출된 결과값을 찾아볼 수 있다.  
`aggregate` 는 집계 쿼리(aggregation query) 를 실행할 때 호출하는 메서드다.
각 도큐먼트는 특정 단계 연산자(stage operator) 를 규정해야 한다.

`limit` 단계를 포함하도록 파이프라인을 조금 확장해보자.

```javascript
db.companies.aggregate([
  { $match: { founded_year: 2004 } },
  { $limit: 5 },
  {
    $project: {
      _id: 0,
      name: 1,
    },
  },
]);
```

`$sort` 문을 추가해 보자

```javascript
db.companies.aggregate([
  { $match: { founded_year: 2004 } },
  { $sort: { name: 1 } },
  { $limit: 5 },
  {
    $project: {
      _id: 0,
      name: 1,
    },
  },
]);
```

### 7.3 표현식

집계 파이프라인을 구축할 ㄸ ㅐ사용할 수 있는 다양한 유형의 표현식을 이해하는 것이 중요하다. 집계 프레임워크는 다양한 표현식 클래스를 지원한다.

- 불리언 표현식
  - 불리언 표현식을 사용하면 AND, OR, NOT 표현식을 쓸 수 있다.
- 집합 표현식
  - 집합 표현식(set expression) 을 사용하면 배열을 집합으로 사용할 수 있다. 예를 들어 2개 이상의 집합의 교집합이나 합집합을 얻을 수 있다. 또한 두 집합의 차를 이용해 여러 집합 연산을 수행할 수 있다.
- 비교 표현식
  - 비교 표현식을 통해 다양한 유형의 범위 필터를 표현할 수 있다.
- 산술 표현식
  - 산술 표현식을 사용하면 수학적 연산을 수행할 수 있다. ceil, floor, 자연 로그, 로그를 계산할 수 있을 뿐 아니라 곱하기, 나누기, 더하기, 빼기와 같은 간단한 산술 연산을 수행할 수 있다.
- 문자열 표현식
  - 문자열 표현식을 사용하면 연결, 하위 문자열 검색, 대소문자 및 텍스트 검색과 관련된 작업을 수행할 수 있다.
- 배열 표현식
  - 배열 표현식은 요소를 필터링하거나, 분할하거나, 특정 배열에서 값의 범위를 가져오는 등 배열을 조작하는 데 유용하다
- 가변적 표현식
  - variable expression 을 사용해 리터럴, 날짜 값 구문 분석을 위한 식, 조건식을 사용한다.
- 누산기
  - 누산기는 합계, 기술 통계 및 기타 여러 유형의 값을 계산하는 기능을 제공한다.

### 7.4. $project

중첩 필드(nested field) 를 승격(promoting) 하는 방법을 알아보자. 다음 파이프라인에서는 일치를 수행한다.

```javascript
db.companies.aggregate([
  {
    $match: {
      "funding_rounds.investments.financial_org.permalink": "greylock",
    },
  },
  {
    $project: {
      _id: 0,
      name: 1,
      ipo: "$ipo.pub_year",
      valuation: "$ipo.valuation_amount",
      funders: "$funding_rounds.investments.financial_org.permalink",
    },
  },
]);
```

### 7.5 $unwind

$unwind 문을 사용하면, 집계 파이프라인의 선출 단계 전에 전개 단계를 포함하고, $unwind 문에 속한 배열을 펼친다. 갱신된 집계 쿼리는 다음과 같다.

```javascript
db.companies.aggregate([
  {
    $match: {
      "funding_rounds.investments.financial_org.permalink": "greylock",
    },
  },
  { $unwind: "$funding_rounds" },
  {
    $project: {
      _id: 0,
      name: 1,
      amount: "$funding_rounds.raised_amount",
      year: "$funding_rounds.funded_year",
    },
  },
]);
```
> 주의할만한 점 : `$unwind` 명령어 앞에는 각 컬럼명 앞에 `"$"` 를 붙여야 제대로 동작한다

위의 쿼리를 실행하면 funding_rounds 배열이 펼쳐지고, 각 배열 요소는 독립된 도큐먼트로 반환된다. `funding_rounds` 필드를 제외한 모든 필드는 키와 값이 동일하다. 

```json
{ "name": "Digg", "amount": 8500000, "year": 2006 },
{ "name": "Digg", "amount": 2800000, "year": 2005 },
{ "name": "Digg", "amount": 2870000, "year": 2008 },
{ "name": "Digg", "amount": 5000000, "year": 2011 },
{ "name": "Facebook", "amount": 50000, "year": 2004 },
{ "name": "Facebook", "amount": 2750000, "year": 2006 },
```

### 7.7 누산기 
* 모든 값 합산(`$sum`)
* 평균 합산(`$avg`)
* 첫 번째 값(`$first`)
* 마지막 값(`$last`)