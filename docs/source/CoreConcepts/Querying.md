# 쿼리(조회)

## 속성

특정 속성을 선택하기 위해 `attributes`을 사용할 수 있습니다. 이 옵션은 배열을 전달합니다.

```js
Model.findAll({
  attributes: ['foo', 'bar']
});
```

```sql
SELECT foo, bar ...
```

attributes는 내부에 배열을 사용하여 이름을 바꿀 수 있습니다.

```js
Model.findAll({
  attributes: ['foo', ['bar', 'baz']]
});
```

```sql
SELECT foo, bar AS baz ...
```

여러분들은 집계를 하기위해 `sequelize.fn`를 사용할 수 있습니다.

```js
Model.findAll({
  attributes: [[sequelize.fn('COUNT', sequelize.col('hats')), 'no_hats']]
});
```

```sql
SELECT COUNT(hats) AS no_hats ...
```

집계함수를 사용 할 때, 여러분들은 해당 모델에서 접근할 수 있는 별명을 전달 해야합니다. 위의 예를 보면 여러분은 hats의 갯수를 `instance.get('no_hats')`를 사용하여 가져올 수 있습니다.

집계값을 추가 할 때 모델의 모든 속성들을 명시하는 것은 불편 할 수 있습니다.

```js
// This is a tiresome way of getting the number of hats...
Model.findAll({
  attributes: ['id', 'foo', 'bar', 'baz', 'quz', [sequelize.fn('COUNT', sequelize.col('hats')), 'no_hats']]
});

// This is shorter, and less error prone because it still works if you add / remove attributes
Model.findAll({
  attributes: { include: [[sequelize.fn('COUNT', sequelize.col('hats')), 'no_hats']] }
});
```

```sql
SELECT id, foo, bar, baz, quz, COUNT(hats) AS no_hats ...
```

몇 가지 속성을 제거 할 수 있습니다.

```js
Model.findAll({
  attributes: { exclude: ['baz'] }
});
```

```sql
SELECT id, foo, bar, quz ...
```


## WHERE

여러분들은 findAll/find 또는 bulk updates/destorys 쿼리를 보낼때 쿼리를 필터하기 위해 `where` 객체를 보낼 수 있다.

`where`는 일반적으로 속성:값 쌍을 이루는 객체를 가진다, where value는 매칭이 되는 기본적인 값 또는 다양한 연산자 객체를 위한 키가된다.

그것은 내부적으로 `or` 그리고 `and`로 설정된 복잡한 AND/OR 조합이 가능하다

### (기본) Basics

```js
const Op = Sequelize.Op;

Post.findAll({
  where: {
    authorId: 2
  }
});
// SELECT * FROM post WHERE authorId = 2

Post.findAll({
  where: {
    authorId: 12,
    status: 'active'
  }
});
// SELECT * FROM post WHERE authorId = 12 AND status = 'active';

Post.findAll({
  where: {
    [Op.or]: [{authorId: 12}, {authorId: 13}]
  }
});
// SELECT * FROM post WHERE authorId = 12 OR authorId = 13;

Post.findAll({
  where: {
    authorId: {
      [Op.or]: [12, 13]
    }
  }
});
// SELECT * FROM post WHERE authorId = 12 OR authorId = 13;

Post.destroy({
  where: {
    status: 'inactive'
  }
});
// DELETE FROM post WHERE status = 'inactive';

Post.update({
  updatedAt: null,
}, {
  where: {
    deletedAt: {
      [Op.ne]: null
    }
  }
});
// UPDATE post SET updatedAt = null WHERE deletedAt NOT NULL;

Post.findAll({
  where: sequelize.where(sequelize.fn('char_length', sequelize.col('status')), 6)
});
// SELECT * FROM post WHERE char_length(status) = 6;
```


### (연산자) Operators

```js
const Op = Sequelize.Op

[Op.and]: [{a: 5}, {b: 6}] // (a = 5) AND (b = 6)
[Op.or]: [{a: 5}, {a: 6}]  // (a = 5 OR a = 6)
[Op.gt]: 6,                // > 6
[Op.gte]: 6,               // >= 6
[Op.lt]: 10,               // < 10
[Op.lte]: 10,              // <= 10
[Op.ne]: 20,               // != 20
[Op.eq]: 3,                // = 3
[Op.is]: null              // IS NULL
[Op.not]: true,            // IS NOT TRUE
[Op.between]: [6, 10],     // BETWEEN 6 AND 10
[Op.notBetween]: [11, 15], // NOT BETWEEN 11 AND 15
[Op.in]: [1, 2],           // IN [1, 2]
[Op.notIn]: [1, 2],        // NOT IN [1, 2]
[Op.like]: '%hat',         // LIKE '%hat'
[Op.notLike]: '%hat'       // NOT LIKE '%hat'
[Op.iLike]: '%hat'         // ILIKE '%hat' (case insensitive) (PG only)
[Op.notILike]: '%hat'      // NOT ILIKE '%hat'  (PG only)
[Op.startsWith]: 'hat'     // LIKE 'hat%'
[Op.endsWith]: 'hat'       // LIKE '%hat'
[Op.substring]: 'hat'      // LIKE '%hat%'
[Op.regexp]: '^[h|a|t]'    // REGEXP/~ '^[h|a|t]' (MySQL/PG only)
[Op.notRegexp]: '^[h|a|t]' // NOT REGEXP/!~ '^[h|a|t]' (MySQL/PG only)
[Op.iRegexp]: '^[h|a|t]'    // ~* '^[h|a|t]' (PG only)
[Op.notIRegexp]: '^[h|a|t]' // !~* '^[h|a|t]' (PG only)
[Op.like]: { [Op.any]: ['cat', 'hat']}
                           // LIKE ANY ARRAY['cat', 'hat'] - also works for iLike and notLike
[Op.overlap]: [1, 2]       // && [1, 2] (PG array overlap operator)
[Op.contains]: [1, 2]      // @> [1, 2] (PG array contains operator)
[Op.contained]: [1, 2]     // <@ [1, 2] (PG array contained by operator)
[Op.any]: [2,3]            // ANY ARRAY[2, 3]::INTEGER (PG only)

[Op.col]: 'user.organization_id' // = "user"."organization_id", with dialect specific column identifiers, PG in this example
[Op.gt]: { [Op.all]: literal('SELECT 1') }
                          // > ALL (SELECT 1)
```

#### 범위 연산자

지원되는 모든 연산자를 사용하여 범위 유형을 쿼리 할 수 ​​있습니다.

명심해야한다. 제공된 범위값은 [바인딩에 포함/제외](https://sequelize.org/master/manual/data-types.html#range-types)에 대해서도 정의할 수 있습니다.

```js
// All the above equality and inequality operators plus the following:

[Op.contains]: 2           // @> '2'::integer (PG range contains element operator)
[Op.contains]: [1, 2]      // @> [1, 2) (PG range contains range operator)
[Op.contained]: [1, 2]     // <@ [1, 2) (PG range is contained by operator)
[Op.overlap]: [1, 2]       // && [1, 2) (PG range overlap (have points in common) operator)
[Op.adjacent]: [1, 2]      // -|- [1, 2) (PG range is adjacent to operator)
[Op.strictLeft]: [1, 2]    // << [1, 2) (PG range strictly left of operator)
[Op.strictRight]: [1, 2]   // >> [1, 2) (PG range strictly right of operator)
[Op.noExtendRight]: [1, 2] // &< [1, 2) (PG range does not extend to the right of operator)
[Op.noExtendLeft]: [1, 2]  // &> [1, 2) (PG range does not extend to the left of operator)
```

#### (조합) Combinations

```js
const Op = Sequelize.Op;

{
  rank: {
    [Op.or]: {
      [Op.lt]: 1000,
      [Op.eq]: null
    }
  }
}
// rank < 1000 OR rank IS NULL

{
  createdAt: {
    [Op.lt]: new Date(),
    [Op.gt]: new Date(new Date() - 24 * 60 * 60 * 1000)
  }
}
// createdAt < [timestamp] AND createdAt > [timestamp]

{
  [Op.or]: [
    {
      title: {
        [Op.like]: 'Boat%'
      }
    },
    {
      description: {
        [Op.like]: '%boat%'
      }
    }
  ]
}
// title LIKE 'Boat%' OR description LIKE '%boat%'
```

#### 연산자 별칭

Sequelize를 사용하면 특정 문자열을 연산자의 별칭으로 설정할 수 있습니다. v5에서는 사용 중단 경고가 표시됩니다.

```js
const Op = Sequelize.Op;
const operatorsAliases = {
  $gt: Op.gt
}
const connection = new Sequelize(db, user, pass, { operatorsAliases })

[Op.gt]: 6 // > 6
$gt: 6 // same as using Op.gt (> 6)
```

#### 연산자 보안

기본적으로 Sequelize는 Symbol 연산자를 사용합니다. 별명없이 Sequelize를 사용하면 보안이 향상됩니다. 문자열 별칭이 없으면 연산자를 주입 할 가능성이 극히 적지 만 항상 사용자 입력의 유효성을 검사해야 합니다.

일부 프레임워크는 자동으로 사용자 입력을 js 객체로 구문 분석하고 입력 처리하지 못하면 문자열 연산자가 포함 된 객체를 Sequelize에 삽입 할 수 있습니다.

더 나은 보안을 위해 `Op.and` / `Op.or`와 같은 `Sequelize.Op`의 기호 연산자를 코드에 사용하고 `$and` / `$or`와 같은 문자열 기반 연산자에 전혀 의존하지 않는 것이 좋습니다. `OperatorAliases` 옵션을 설정하여 응용 프로그램에 필요한 별칭을 제한 할 수 있습니다. 특히 사용자 입력을 Sequelize 메서드에 직접 전달할 때 사용자 입력을 이스케이프합니다.

```js
const Op = Sequelize.Op;

//use sequelize without any operators aliases
const connection = new Sequelize(db, user, pass, { operatorsAliases: false });

//use sequelize with only alias for $and => Op.and
const connection2 = new Sequelize(db, user, pass, { operatorsAliases: { $and: Op.and } });
```

경고없이 모든 기본 별칭을 계속 사용할 경우 기본별칭을 사용하고 제한하지 않을 경우 sequlize에서 경고합니다. 다음 연산자를 전달할 수 있습니다.

```js
const Op = Sequelize.Op;
const operatorsAliases = {
  $eq: Op.eq,
  $ne: Op.ne,
  $gte: Op.gte,
  $gt: Op.gt,
  $lte: Op.lte,
  $lt: Op.lt,
  $not: Op.not,
  $in: Op.in,
  $notIn: Op.notIn,
  $is: Op.is,
  $like: Op.like,
  $notLike: Op.notLike,
  $iLike: Op.iLike,
  $notILike: Op.notILike,
  $regexp: Op.regexp,
  $notRegexp: Op.notRegexp,
  $iRegexp: Op.iRegexp,
  $notIRegexp: Op.notIRegexp,
  $between: Op.between,
  $notBetween: Op.notBetween,
  $overlap: Op.overlap,
  $contains: Op.contains,
  $contained: Op.contained,
  $adjacent: Op.adjacent,
  $strictLeft: Op.strictLeft,
  $strictRight: Op.strictRight,
  $noExtendRight: Op.noExtendRight,
  $noExtendLeft: Op.noExtendLeft,
  $and: Op.and,
  $or: Op.or,
  $any: Op.any,
  $all: Op.all,
  $values: Op.values,
  $col: Op.col
};

const connection = new Sequelize(db, user, pass, { operatorsAliases });
```

### JSON

JSON 데이터 타입은 PostgreSQL, SQLite, MySQL 그리고 MariaDB에서 제공합니다.

#### PostgreSQL

PostgreSQL의 JSON 데이터 형식은 이진 표현이 아닌 일반 텍스트로 값을 저장합니다. 단순히 JSON 표현을 저장하고 검색하려는 경우 JSON을 사용하면 입력 표현에서 빌드하는 데 필요한 디스크 공간과 시간이 줄어 듭니다. 그러나 JSON 값에 대한 조작을 수행하려면 아래 설명 된 JSONB 데이터 유형을 선호해야합니다.

#### MSSQL
 
MSSQL에는 JSON 데이터 형식이 없지만 SQL Server 2016 이후 특정 함수를 통해 문자열로 저장된 JSON을 지원합니다.이 함수를 사용하면 문자열에 저장된 JSON을 쿼리 할 수 ​​있지만 반환 된 값은 모두 필요합니다. 별도로 구문 분석해야합니다.

```js
// ISJSON - to test if a string contains valid JSON
User.findAll({
  where: sequelize.where(sequelize.fn('ISJSON', sequelize.col('userDetails')), 1)
})

// JSON_VALUE - extract a scalar value from a JSON string
User.findAll({
  attributes: [[ sequelize.fn('JSON_VALUE', sequelize.col('userDetails'), '$.address.Line1'), 'address line 1']]
})

// JSON_VALUE - query a scalar value from a JSON string
User.findAll({
  where: sequelize.where(sequelize.fn('JSON_VALUE', sequelize.col('userDetails'), '$.address.Line1'), '14, Foo Street')
})

// JSON_QUERY - extract an object or array
User.findAll({
  attributes: [[ sequelize.fn('JSON_QUERY', sequelize.col('userDetails'), '$.address'), 'full address']]
})
```

### JSONB

JSONB는 3가지 다른방법으로 쿼리를 할 수 있다.

#### Nested object(내부에 객체를 선언)

```js
{
  meta: {
    video: {
      url: {
        [Op.ne]: null
      }
    }
  }
}
```

#### Nested key(내부를 키로 가진다)

```js
{
  "meta.audio.length": {
    [Op.gt]: 20
  }
}
```

#### Containment(원천봉쇄)

```
{
  "meta": {
    [Op.contains]: {
      site: {
        url: 'http://google.com'
      }
    }
  }
}
```

### Relations / Associations(관계)

```js
// Find all projects with a least one task where task.state === project.state
Project.findAll({
    include: [{
        model: Task,
        where: { state: Sequelize.col('project.state') }
    }]
})
```


## Pagination / Limiting(페이지 분할, 제한)

```js
// Fetch 10 instances/rows
Project.findAll({ limit: 10 })

// Skip 8 instances/rows
Project.findAll({ offset: 8 })

// Skip 5 instances and fetch the 5 after that
Project.findAll({ offset: 5, limit: 5 })
```

## Ordering

`order`은 질의를 순서대로 정렬하기 위해 아이템들을 배열로 가지고 있습니다. 여러분들은 안전한 이스케이핑을 원할 때 튜플/배열을 사용할 것이다.

```js
Subtask.findAll({
  order: [
    // Will escape title and validate DESC against a list of valid direction parameters
    ['title', 'DESC'],

    // Will order by max(age)
    sequelize.fn('max', sequelize.col('age')),

    // Will order by max(age) DESC
    [sequelize.fn('max', sequelize.col('age')), 'DESC'],

    // Will order by  otherfunction(`col1`, 12, 'lalala') DESC
    [sequelize.fn('otherfunction', sequelize.col('col1'), 12, 'lalala'), 'DESC'],

    // Will order an associated model's created_at using the model name as the association's name.
    [Task, 'createdAt', 'DESC'],

    // Will order through an associated model's created_at using the model names as the associations' names.
    [Task, Project, 'createdAt', 'DESC'],

    // Will order by an associated model's created_at using the name of the association.
    ['Task', 'createdAt', 'DESC'],

    // Will order by a nested associated model's created_at using the names of the associations.
    ['Task', 'Project', 'createdAt', 'DESC'],

    // Will order by an associated model's created_at using an association object. (preferred method)
    [Subtask.associations.Task, 'createdAt', 'DESC'],

    // Will order by a nested associated model's created_at using association objects. (preferred method)
    [Subtask.associations.Task, Task.associations.Project, 'createdAt', 'DESC'],

    // Will order by an associated model's created_at using a simple association object.
    [{model: Task, as: 'Task'}, 'createdAt', 'DESC'],

    // Will order by a nested associated model's created_at simple association objects.
    [{model: Task, as: 'Task'}, {model: Project, as: 'Project'}, 'createdAt', 'DESC']
  ]

  // Will order by max age descending
  order: sequelize.literal('max(age) DESC')

  // Will order by max age ascending assuming ascending is the default order when direction is omitted
  order: sequelize.fn('max', sequelize.col('age'))

  // Will order by age ascending assuming ascending is the default order when direction is omitted
  order: sequelize.col('age')

  // Will order randomly based on the dialect (instead of fn('RAND') or fn('RANDOM'))
  order: sequelize.random()
})
```

## table hint


mssql을 사용할 때 `tableHint`를 사용하여 테이블 힌트를 선택적으로 전달할 수 있습니다. 힌트는 `Sequelize.TableHints`의 값이어야하며 반드시 필요한 경우에만 사용해야합니다. 쿼리 당 단일 테이블 힌트 만 지원됩니다.

테이블 힌트는 특정 옵션을 지정하여 mssql 쿼리 최적화 프로그램의 기본 동작을 재정의합니다. 해당 절에서 참조 된 테이블이나 뷰에만 영향을줍니다.

```js
const TableHints = Sequelize.TableHints;

Project.findAll({
  // adding the table hint NOLOCK
  tableHint: TableHints.NOLOCK
  // this will generate the SQL 'WITH (NOLOCK)'
})
```

## index Hints

mysql을 사용할 때 `indexHints`를 사용하여 인덱스 힌트를 선택적으로 전달할 수 있습니다. 힌트 유형은 `Sequelize.IndexHints`의 값이어야하며 값은 기존 인덱스를 참조해야합니다.

인덱스 힌트는 [mysql 쿼리 최적화 프로그램의 기본 동작보다 우선](https://dev.mysql.com/doc/refman/5.7/en/index-hints.html)합니다.

```js
Project.findAll({
  indexHints: [
    { type: IndexHints.USE, values: ['index_project_on_name'] }
  ],
  where: {
    id: {
      [Op.gt]: 623
    },
    name: {
      [Op.like]: 'Foo %'
    }
  }
})
```

다음과 같이 쿼리문을 생성할 것 입니다.

```sql
SELECT * FROM Project USE INDEX (index_project_on_name) WHERE name LIKE 'FOO %' AND id > 623;
```

`Sequelize.IndexHints`는 `use`, `FORCE` 그리고 `IGNORE`를 포함합니다.


원래 API 제안에 대해서는 [Issue #9421](https://github.com/sequelize/sequelize/issues/9421)을 참조하십시오.