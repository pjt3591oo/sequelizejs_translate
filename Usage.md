
# 데이터 검색/ 발견

발견하는 함수들은 데이터 베이스로부터 데이터를 조회하기 위한 함수이다. 그들은 객체를 반한하지 않는 대신에 모델 인스턴스를 반환한다. 발견하는 함수는 모델 인스턴스를 반환하기 때문에, 여러분들은 어느 모델의 인스턴스에서 [instances](http://docs.sequelizejs.com/en/latest/docs/instances/) 문서에 설명된 함수를 호출할 수 있습니다.

이 문서는 발견하는 함수들이 무엇을 할 수 있는지 설명을 할것이다.


## find - database에서 특정 요소 하나를 검색

```.js
// 알고있는 id로부터 검색
Project.findById(123).then(function(project) {
  // project는 Project 테이블 내용이 저정된 인스턴스이다.
  // 만약 id가 123이 아니면 null값을 가질 것 이다.
})

// 속성으로부터 검색
Project.findOne({ where: {title: 'aProject'} }).then(function(project) {
  // project는 table의 title이 'aProject' 또는 null인 Projects의 첫번쨰 테이블 일 것이다.
})


Project.findOne({
  where: {title: 'aProject'},
  attributes: ['id', ['name', 'title']]
}).then(function(project) {
  // project는 table의 title이 'aProject' 또는 null인 Projects의 첫번쨰 테이블 일 것이다.
  // project.title에는 project의 이름이 포함될 것이다.
})
```


## findOrCreate - 특정 요소를 검색한다 만약 그것이 존재하지 않는다면 생성을 한다.

`findOrCreate` 함수는 데이터베이스에 해당 요소가 이미 포함이 되어있는지 확인하는데 사용될 수 있다. 이 경우 각각의 인스턴스의 결과를 가지는 함수가 된다. 만약 해당 요소가 존재하지 않는다면 새로 만들것이다.

우리는 `username`과 하나의 `job`을 가지고 있는`user`모델과 함께 빈 데이터 베이스를 가지고 있다고 가정을 해보자.

```.js
User
  .findOrCreate({where: {username: 'sdepold'}, defaults: {job: 'Technical Lead JavaScript'}})
  .spread(function(user, created) {
    console.log(user.get({
      plain: true
    }))
    console.log(created)

    /*
      {
        username: 'sdepold',
        job: 'Technical Lead JavaScript',
        id: 1,
        createdAt: Fri Mar 22 2013 21: 28: 34 GMT + 0100(CET),
        updatedAt: Fri Mar 22 2013 21: 28: 34 GMT + 0100(CET)
      }
      created: true
    */
  })
```

우리가 이미 인스턴스를 가지고 있을 때, 코드는 새로운 인스턴스를 생성했다.

```.js
User
  .create({ username: 'fnord', job: 'omnomnom' })
  .then(function() {
    User
      .findOrCreate({where: {username: 'fnord'}, defaults: {job: 'something else'}})
      .spread(function(user, created) {
        console.log(user.get({
          plain: true
        }))
        console.log(created)

        /*
          {
            username: 'fnord',
            job: 'omnomnom',
            id: 2,
            createdAt: Fri Mar 22 2013 21: 28: 34 GMT + 0100(CET),
            updatedAt: Fri Mar 22 2013 21: 28: 34 GMT + 0100(CET)
          }
          created: false
        */
      })
  })
```

... 존재하는 엔트리(인스턴스)는 변경되지 않습니다. 두번째 유저의 job을 보면 created가 false임을 확인해라.


## findAndCountAll - 데이터 베이스에서 여러 요소 검색, 데이터와 전체 갯수 반환

이것은` findAll`과 `count`가 조합된 편의를 위한 함수이다. 이것은 여러분들이 `limit`과 `offset`을 사용하여 pagination을 하기위해 검색을 할 때 사용된다. 그러나 검색결과 총 레코드 수를 알아야 한다.

> pagenation이란 데이터가 너무 많아 분할하는것을 말한다.

findAndCountAll을 성공한 핸들러는 한상 2개의 프로퍼티로 구성된 객체를 받을것이다.

* count - 정수형, where조건에 맞는 레코드 수
* rows - 객체의 배열, where조건과 `limit` 그리고 `offset` 범위에 맞는 레코드들

```.js
Project
  .findAndCountAll({
     where: {
        title: {
          $like: 'foo%'
        }
     },
     offset: 10,
     limit: 2
  })
  .then(function(result) {
    console.log(result.count);
    console.log(result.rows);
  });
```


또한 `findAndCountAll`는 다양한 요소들을 포함한다. `required`가 포함되는 것은 포함된 항목만 개수 부분에 추가될 것이다.

여러분들은 profile에 소속된 모든 유저를 발견한다고 가정을 해보자.

```.js
User.findAndCountAll({
  include: [
     { model: Profile, required: true}
  ],
  limit: 3
});
```

`Profile`은 inner join으로 설정된 `required`를 가지고 있다. 그리고 `User`는 오직 `profile`에 존재하는 유저만 카운트 된다. 만약 include에 `required`가 없다면, 유저는 카운트 된다. `where`절을 추가하면 필수항목이 됩니다.

```.sql
User.findAndCountAll({
  include: [
     { model: Profile, where: { active: true }}
  ],
  limit: 3
});
```

위의 쿼리는 오직 profile의 active가 true인 유저들만 카운트 된다. 왜냐하면 `required `는 여러분이 include에 where절을 추가할 떄 암시적으로 설정이 된다.

`findAndCountAll`에 전달되는 옵션 객체들은 `findAll`과 같다(아래에 기술 됨)


## findAll - 데이터 베이스의 여러 요소 검색

```.js
// 여러 항목 조회
Project.findAll().then(function(projects) {
  // projects는 Project인스턴스의 배열이다.
})

// 이것도 가능하다:
Project.all().then(function(projects) {
  // projects는 Project 인스턴스의 배열이다.
})

// 특정 속성 검색
Project.findAll({ where: { name: 'A Project' } }).then(function(projects) {
  // projects는 name이 'Aproject'인 Project 인스턴스의 배열이다
})

// search with string replacements
Project.findAll({ where: ["id > ?", 25] }).then(function(projects) {
  // projects는 id가 25보다 큰 Project 인스턴스의 배열이다
})

// search within a specific range
Project.findAll({ where: { id: [1,2,3] } }).then(function(projects) {
  // projects는 id가 1,2 또는3인 Project 인스턴스의 배열이다
  // 이것은 사실 IN쿼리를 하는 것이다.
})

Project.findAll({
  where: {
    id: {
      $and: {a: 5}           // AND (a = 5)
      $or: [{a: 5}, {a: 6}]  // (a = 5 OR a = 6)
      $gt: 6,                // id > 6
      $gte: 6,               // id >= 6
      $lt: 10,               // id < 10
      $lte: 10,              // id <= 10
      $ne: 20,               // id != 20
      $between: [6, 10],     // BETWEEN 6 AND 10
      $notBetween: [11, 15], // NOT BETWEEN 11 AND 15
      $in: [1, 2],           // IN [1, 2]
      $notIn: [1, 2],        // NOT IN [1, 2]
      $like: '%hat',         // LIKE '%hat'
      $notLike: '%hat'       // NOT LIKE '%hat'
      $iLike: '%hat'         // ILIKE '%hat' (case insensitive)  (PG only)
      $notILike: '%hat'      // NOT ILIKE '%hat'  (PG only)
      $overlap: [1, 2]       // && [1, 2] (PG array overlap operator)
      $contains: [1, 2]      // @> [1, 2] (PG array contains operator)
      $contained: [1, 2]     // <@ [1, 2] (PG array contained by operator)
      $any: [2,3]            // ANY ARRAY[2, 3]::INTEGER (PG only)
    },
    status: {
      $not: false,           // status NOT FALSE
    }
  }
})
```


## (복잡한 필터링, OR, AND 쿼리) Complex filtering / OR / NOT queries

AND, OR, NOT 조건이 포함된 복잡한 where이 사용가능 하다. 여러분들은 `$or`, `$and` 또는 `$not`을 사용할 수 있다.

```.js
Project.findOne({
  where: {
    name: 'a project',
    $or: [
      { id: [1,2,3] },
      { id: { $gt: 10 } }
    ]
  }
})

Project.findOne({
  where: {
    name: 'a project',
    id: {
      $or: [
        [1,2,3],
        { $gt: 10 }
      ]
    }
  }
})
```

코드의 조각들은 다음과 같이 코드를 만들어 낸다.

```.sql
SELECT *
FROM `Projects`
WHERE (
  `Projects`.`name` = 'a project'
   AND (`Projects`.`id` IN (1,2,3) OR `Projects`.`id` > 10)
)
LIMIT 1;
```

`$not` 예시

```.js
Project.findOne({
  where: {
    name: 'a project',
    $not: [
      { id: [1,2,3] },
      { array: { $contains: [3,4,5] } }
    ]
  }
});
```

다음과 같이 만들어 낸다.

```.sql
SELECT *
FROM `Projects`
WHERE (
  `Projects`.`name` = 'a project'
   AND NOT (`Projects`.`id` IN (1,2,3) OR `Projects`.`array` @> ARRAY[1,2,3]::INTEGER[])
)
LIMIT 1;
```

## 데이터베이스를 조작하는 것 limit, offset, order, group

더 다양한 데이터 검색을 하기위해, 여러분들은  limit, offset, order 그리고 group을 사용할 수 있습니다.

```.js
// 쿼리의 결과 제한
Project.findAll({ limit: 10 })

// 10번째 요소까지 건너뛴다.
Project.findAll({ offset: 10 })

// 10번째 요소까지 건너뛰고 2개 출력
Project.findAll({ offset: 10, limit: 2 })
```

grouping과 order 문법은 같다. 아래에는 group와 order를 위한 하나의 예시와 함께 설명이 되어있다. 아래에 보이는 것을 그룹을 위해 사용 할 수 있다.

```.js
Project.findAll({order: 'title DESC'})
// title을 기준으로 정렬한다.

Project.findAll({group: 'name'})
// name을 그룹화 한다.
```

위의 두개의 예제에서, 문자열은 쿼리 내부에 삽입된다. 또한 컬럼 이름은 이스케이핑 되지 않는다. 여러분들이 order/group에 문자열을 제공할 때, 컬럼 이름에 이스케이핑을 하지 않는다. 만약에 여러분들중 컬럼이름에 이스케이핑을 원한다면 여러분들은 해당 인자들의 리스트들을 제공해야만 한다. 심지어 하나의 컬럼에 한해서만이라도 리스트로 제공되어야 한다.

```.js
something.findOne({
  order: [
    'name',
    // 'name'을 반환 할 것이다.
    'username DESC',
    // `username DESC`을 반환 할 것이다.
    ['username', 'DESC'],
    // `username` DESC을 반환 할 것이다.
    sequelize.fn('max', sequelize.col('age')),
    // max(`age`)를 반환 할 것이다
    [sequelize.fn('max', sequelize.col('age')), 'DESC'],
    // max(`age`) DESC를 반환 할 것이다.
    [sequelize.fn('otherfunction', sequelize.col('col1'), 12, 'lalala'), 'DESC'],
    // otherfunction(`col1`, 12, 'lalala') DESC를 반환 할 것이다.
    [sequelize.fn('otherfunction', sequelize.fn('awesomefunction', sequelize.col('col'))), 'DESC']
    // otherfunction(awesomefunction(`col`)) DESC를 반환 할 것이다, 이 중첩은 계속적으로 할 수 있습니다.
    [{ raw: 'otherfunction(awesomefunction(`col`))' }, 'DESC']
    // This won't be quoted, but direction will be added
  ]
})
```

마지막으로 다시한번 더 설명을 하면, order/group의 요소 배열은 다음과 같다.

- String - ''일 것이다.
- Array -
- Object -
- Raw는 ''없이 포함된다.
- 나머지는 무시되고 만약 raw를 설정하지 않았다면, 그 쿼리는 실패한다.
- `Sequelize.fn` 와 `Sequelize.col`는 함수를 반환한다.

## Raw queries

여러분들은 종종 조작에 대한 정보 없이 화면에 출력되는 데이터 셋을 기대할 것이다. 여러분이 조회한 각 행에 대해서 sequelize는 update, delete, 관계에 대한 함수와 함께 인스턴스를 생성한다. 만약 여러분들이 수천개의 행을 가지고 있다면, 이것은 상당한 시간이 소요 될 것 이다. 만약 여러분들이 순수한 데이터만 필요로 하다면, 업데이트에 대해서 아무것도 원하지 않는다면 여러분들은 순수한 데이터 셋을 얻기위해 다음과 같이 할 수 있다.

```.js
// DB에서 방대한 데이터 세트를 기대하나요? ,
// 각 항목에 대해 DAO들을 만드는데 시간을 보내고 싶지않나요?
// 여러분들은 원래의 데이터만 가져오기 위해 쿼리에 옵션을 추가하여 전달 할 수 있습니다.
Project.findAll({ where: { ... }, raw: true })
```

## count - 데이터 베이스로부터 요소의 발생 갯수

데이터 베이스 객체를 카운팅 하기위한 함수

```.js
Project.count().then(function(c) {
  console.log("There are " + c + " projects!")
})

Project.count({ where: ["id > ?", 25] }).then(function(c) {
  console.log("There are " + c + " projects with an id greater than 25.")
})
```

## max - 속성의 값중에서 최대값을 얻기

```.js
/*
  age 속성을 가진 3개의 사람 객체를 가정하자.
  첫번째로 10살,
  두번째로 5살,
  세번째로 40살,
*/
Project.max('age').then(function(max) {
  // 이것은 40을 반환 할 것이다.
})

Project.max('age', { where: { age: { lt: 20 } } }).then(function(max) {
  // 10일 것이다.
})
```

## min - 특정 테이블 내의 특정 속성의 값중 가장 작은 값을 얻기

```.js
/*
    age 속성을 가진 3개의 사람 객체를 가정하자.
    첫번째로 10살,
    두번째로 5살,
    세번째로 40살,
*/
Project.min('age').then(function(min) {
  // 5를 반환할 것이다.
})

Project.min('age', { where: { age: { $gt: 5 } } }).then(function(min) {
  // 10일 것이다.
})
```

## sum - 특정 속성의 값의 합

한 테이블에서 특정 속성들의 합을 계산하기 위해서 여러분들은 `sum` 함수를 사용 할 수 있다.

```.js
/*
    age 속성을 가진 3개의 사람 객체를 가정하자.
    첫번째로 10살,
    두번째로 5살,
    세번째로 40살,
*/
Project.sum('age').then(function(sum) {
  // 55를 반환 할 것이다.
})

Project.sum('age', { where: { age: { $gt: 5 } } }).then(function(sum) {
  // 50일 것이다.
})
```


# Eager loading

데이터베이스에서 데이터를 검색 할 때 동일한 쿼리와의 연관성을 얻으려 할때, 이것을 eager loding이라고 부른다. 그것의 근본적인 아이디어는 `find` 혹은 `findAll`을 호출할 때 `include` 속성을 사용한다. 다음과 같이 설정을 해보자.

```.js
var User = sequelize.define('user', { name: Sequelize.STRING })
  , Task = sequelize.define('task', { name: Sequelize.STRING })
  , Tool = sequelize.define('tool', { name: Sequelize.STRING })

Task.belongsTo(User)
User.hasMany(Task)
User.hasMany(Tool, { as: 'Instruments' })

sequelize.sync().then(function() {
  // 이곳에 코드를 작성하세요.
})
```

됬다. 첫번째로 user와 관련된 작업들을 전부 로드를하자.

```.js
Task.findAll({ include: [ User ] }).then(function(tasks) {
  console.log(JSON.stringify(tasks))

  /*
    [{
      "name": "A Task",
      "id": 1,
      "createdAt": "2013-03-20T20:31:40.000Z",
      "updatedAt": "2013-03-20T20:31:40.000Z",
      "userId": 1,
      "user": {
        "name": "John Doe",
        "id": 1,
        "createdAt": "2013-03-20T20:31:45.000Z",
        "updatedAt": "2013-03-20T20:31:45.000Z"
      }
    }]
  */
})
```

여기서 주목되는 접근자(인스턴스 결과의 `Tasks`의 정보)은 일대일 관계이기 때문에 단수형태이다.

다음으로 다대일의 관계로부터 데이터를 로딩하자

```.js
User.findAll({ include: [ Task ] }).then(function(users) {
  console.log(JSON.stringify(users))

  /*
    [{
      "name": "John Doe",
      "id": 1,
      "createdAt": "2013-03-20T20:31:45.000Z",
      "updatedAt": "2013-03-20T20:31:45.000Z",
      "tasks": [{
        "name": "A Task",
        "id": 1,
        "createdAt": "2013-03-20T20:31:40.000Z",
        "updatedAt": "2013-03-20T20:31:40.000Z",
        "userId": 1
      }]
    }]
  */
})
```

여기서 주목되는 접근자(인스턴스 결과의 `Tasks`의 정보)은 일대다 관계이기 때문에 복수형태이다.

만약 관계에서 별명을 주었다면(`as` 옵션을 사용하는 것), 여러분들은 모델을 포함 할 때 해당 별명을 명시해야만 한다. user의 `Tool`은 `Instruments`와 같이 별칭이 되었다. 여러분들은 별칭을 사용하여 데이터를 불러오기 원한다면 모델을 명시해야만 한다.

```.js
User.findAll({ include: [{ model: Tool, as: 'Instruments' }] }).then(function(users) {
  console.log(JSON.stringify(users))

  /*
    [{
      "name": "John Doe",
      "id": 1,
      "createdAt": "2013-03-20T20:31:45.000Z",
      "updatedAt": "2013-03-20T20:31:45.000Z",
      "Instruments": [{
        "name": "Toothpick",
        "id": 1,
        "createdAt": null,
        "updatedAt": null,
        "userId": 1
      }]
    }]
  */
})
```

우리는 eager loding할 때  `where`을 사용하여 관계되어있는 모델을 필터링을 할 수 있다. 이것은 `tool`모델에서 매칭된 행의 `user`를 반활 할 것이다.

```.js
User.findAll({
    include: [{
        model: Tool,
        as: 'Instruments',
        where: { name: { $like: '%ooth%' } }
    }]
}).then(function(users) {
    console.log(JSON.stringify(users))

    /*
      [{
        "name": "John Doe",
        "id": 1,
        "createdAt": "2013-03-20T20:31:45.000Z",
        "updatedAt": "2013-03-20T20:31:45.000Z",
        "Instruments": [{
          "name": "Toothpick",
          "id": 1,
          "createdAt": null,
          "updatedAt": null,
          "userId": 1
        }]
      }],

      [{
        "name": "John Smith",
        "id": 2,
        "createdAt": "2013-03-20T20:31:45.000Z",
        "updatedAt": "2013-03-20T20:31:45.000Z",
        "Instruments": [{
          "name": "Toothpick",
          "id": 1,
          "createdAt": null,
          "updatedAt": null,
          "userId": 1
        }]
      }],
    */
  })
```

eager lodeing된 모델이 `include.where` 사용하여 필터링 된다면 `include.required`은 암묵적으로 `true`로 설정이 된다. 이것은 하위 항목이 존재하는 상위 모델을 반환하는 내부 조인임을 의미한다.


## includeing everthing

모든 속성을 포함하기 위해, 여러분들은 `all : true`와 함께 객체를 전달 할 수 있다.

```.js
User.findAll({ include: [{ all: true }]});
```

## include soft deleted records

이경우는 여러분들이 레코드 항목에 삭제가 된 eager loding을 원할때 여러분들은 `include.paranoid`를 `true`로 설정 할 수 있다.

```.js
User.findAll({
    include: [{
        model: Tool,
        where: { name: { $like: '%ooth%' } },
        paranoid: true // query and loads the soft deleted records
    }]
});
```

## ordering Eager Loaded Associations

일대다 관계에 해당한다.

```.js
Company.findAll({ include: [ Division ], order: [ [ Division, 'name' ] ] });
Company.findAll({ include: [ Division ], order: [ [ Division, 'name', 'DESC' ] ] });
Company.findAll({
  include: [ { model: Division, as: 'Div' } ],
  order: [ [ { model: Division, as: 'Div' }, 'name' ] ]
});
Company.findAll({
  include: [ { model: Division, as: 'Div' } ],
  order: [ [ { model: Division, as: 'Div' }, 'name', 'DESC' ] ]
});
Company.findAll({
  include: [ { model: Division, include: [ Department ] } ],
  order: [ [ Division, Department, 'name' ] ]
});
```

다대다 조인의 경우, 여러분들은 테이블에서 속성을 통해 정렬을 할 수 있다.

```.js
Company.findAll({
  include: [ { model: Division, include: [ Department ] } ],
  order: [ [ Division, DepartmentDivision, 'name' ] ]
});
```

## Nested eager loading

여러분들은 중첩된 eager loding을 사용하여 연관된 모든 모델을 불러올 수 있다.

```.js
User.findAll({
  include: [
    {model: Tool, as: 'Instruments', include: [
      {model: Teacher, include: [ /* etc */]}
    ]}
  ]
}).then(function(users) {
  console.log(JSON.stringify(users))

  /*
    [{
      "name": "John Doe",
      "id": 1,
      "createdAt": "2013-03-20T20:31:45.000Z",
      "updatedAt": "2013-03-20T20:31:45.000Z",
      "Instruments": [{ // 1:M and N:M association
        "name": "Toothpick",
        "id": 1,
        "createdAt": null,
        "updatedAt": null,
        "userId": 1,
        "Teacher": { // 1:1 association
          "name": "Jimi Hendrix"
        }
      }]
    }]
  */
})
```

이것은 외부조인을 제공한다. 그러나 하위 모델의 `where`절은 내부조인을 사용한다. 그리고 서브모델과 일치하는 인스턴스를 반환한다. 만약 모든 부모의 인스턴스를 반환하고 싶다면 여러분들은 `required: false`를 추가해야한다.

```.js
User.findAll({
  include: [{
    model: Tool,
    as: 'Instruments',
    include: [{
      model: Teacher,
      where: {
        school: "Woodstock Music School"
      },
      required: false
    }]
  }]
}).then(function(users) {
  /* ... */
})
```

위의 쿼리는 모든 유저들과 모든 악기들을 반환을 한다. 그러나 `Woodstock Music School`을 가지고 있는 선생님들과 관계된 인스턴스들만 반환을 한다.

all을 포함하면 중첩로드도 지원이 가능하다.

```.js
User.findAll({ include: [{ all: true, nested: true }]});
```
