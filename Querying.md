
# 속성

여러분들은 `attributes` 옵션을 사용하여 특정 속성만 조회를 할 수 있습니다. 여러분들은 옵션을 사용하기 위해 배열에서 전달 할 것입니다..


```.js
Model.findAll({
  attributes: ['foo', 'bar']
});
```


```.sql
SELECT foo, bar ...
```

attributes는 내부에 배열을 사용하여 이름을 바꿀 수 있습니다.

```.js
Model.findAll({
  attributes: ['foo', ['bar', 'baz']]
});
```

```.sql
SELECT foo, bar AS baz ...
```

여러분들은 집계를 하기위해 `sequelize.fn`를 사용할 수 있습니다.

```.js
Model.findAll({
  attributes: [[sequelize.fn('COUNT', sequelize.col('hats')), 'no_hats']]
});
```

```.sql
SELECT COUNT(hats) AS no_hats ...
```

집계함수를 사용 할 때, 여러분들은 해당 모델에서 접근할 수 있는 별명을 전달 해야합니다. 위의 예를 보면 여러분은 hats의 갯수를 `instance.get('no_hats')`를 사용하여 가져올 수 있습니다.

집계값을 추가 할 때 모델의 모든 속성들을 명시하는 것은 불편 할 수 있습니다.

```.js
// This is a tiresome way of getting the number of hats...
Model.findAll({
  attributes: ['id', 'foo', 'bar', 'baz', 'quz', [sequelize.fn('COUNT', sequelize.col('hats')), 'no_hats']]
});

// This is shorter, and less error prone because it still works if you add / remove attributes
Model.findAll({
  attributes: { include: [[sequelize.fn('COUNT', sequelize.col('hats')), 'no_hats']] }
});
```

```.sql
SELECT id, foo, bar, baz, quz, COUNT(hats) AS no_hats ...
```

몇 가지 속성을 제거 할 수 있습니다.

```.js
Model.findAll({
  attributes: { exclude: ['baz'] }
});
```

```.sql
SELECT id, foo, bar, quz ...
```


# WHERE

여러분들은 findAll/find 또는 bulk updates/destorys 쿼리를 보낼때 쿼리를 필터하기 위해 `where` 객체를 보낼 수 있다.

`where`는 일반적으로 속성:값 쌍을 이루는 객체를 가진다, where value는 매칭이 되는 기본적인 값 또는 다양한 연산자 객체를 위한 키가된다.

그것은 내부적으로 `%or` 그리고 `$and`로 설정된 복잡한 AND/OR 조합이 가능하다

## (기본) Basics

```.sql
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
      $ne: null
    }
  }
});
// UPDATE post SET updatedAt = null WHERE deletedAt NOT NULL;

Post.findAll({
  where: sequelize.where(sequelize.fn('char_length', sequelize.col('status')), 6)
});
// SELECT * FROM post WHERE char_length(status) = 6;
```


## (연산자) Operators

```
$and: {a: 5}           // AND (a = 5)
$or: [{a: 5}, {a: 6}]  // (a = 5 OR a = 6)
$gt: 6,                // > 6
$gte: 6,               // >= 6
$lt: 10,               // < 10
$lte: 10,              // <= 10
$ne: 20,               // != 20
$not: true,            // IS NOT TRUE
$between: [6, 10],     // BETWEEN 6 AND 10
$notBetween: [11, 15], // NOT BETWEEN 11 AND 15
$in: [1, 2],           // IN [1, 2]
$notIn: [1, 2],        // NOT IN [1, 2]
$like: '%hat',         // LIKE '%hat'
$notLike: '%hat'       // NOT LIKE '%hat'
$iLike: '%hat'         // ILIKE '%hat' (case insensitive) (PG only)
$notILike: '%hat'      // NOT ILIKE '%hat'  (PG only)
$like: { $any: ['cat', 'hat']}
                       // LIKE ANY ARRAY['cat', 'hat'] - also works for iLike and notLike
$overlap: [1, 2]       // && [1, 2] (PG array overlap operator)
$contains: [1, 2]      // @> [1, 2] (PG array contains operator)
$contained: [1, 2]     // <@ [1, 2] (PG array contained by operator)
$any: [2,3]            // ANY ARRAY[2, 3]::INTEGER (PG only)

$col: 'user.organization_id' // = "user"."organization_id", with dialect specific column identifiers, PG in this example
```


## (조합) Combinations

```
{
  rank: {
    $or: {
      $lt: 1000,
      $eq: null
    }
  }
}
// rank < 1000 OR rank IS NULL

{
  createdAt: {
    $lt: new Date(),
    $gt: new Date(new Date() - 24 * 60 * 60 * 1000)
  }
}
// createdAt < [timestamp] AND createdAt > [timestamp]

{
  $or: [
    {
      title: {
        $like: 'Boat%'
      }
    },
    {
      description: {
        $like: '%boat%'
      }
    }
  ]
}
// title LIKE 'Boat%' OR description LIKE '%boat%'
```


## JSONB

JSONB는 3가지 다른방법으로 쿼리를 할 수 있다.
(문서에는 3가지 방법이라고 나와있는데 정작 4가지 방법이 기술되어있다.)

## Nested object(내부에 객체를 선언)

```
{
  meta: {
    video: {
      url: {
        $ne: null
      }
    }
  }
}
```

## Nested key(내부를 키로 가진다)

```
{
  "meta.audio.length": {
    $gt: 20
  }
}
```

## Containment(원천봉쇄)

```
{
  "meta": {
    $contains: {
      site: {
        url: 'http://google.com'
      }
    }
  }
}
```

## Relations / Associations(관계)

```.js
// task.state === project.task 조건에 맞는 모든 것을 조회
Project.findAll({
    include: [{
        model: Task,
        where: { state: Sequelize.col('project.state') }
    }]
})
```


# Pagination / Limiting(페이지 분할, 제한)

```.js
// 10개의 인스턴스/행
Project.findAll({ limit: 10 })

// 8개를 건너뛴 인스턴스/행
Project.findAll({ offset: 8 })

// 5개를 건너뛰고 5개의 인스턴스/행
Project.findAll({ offset: 5, limit: 5 })
```

# Ordering

`order`은 질의를 순서대로 정렬하기 위해 아이템들을 배열로 가지고 있습니다. 여러분들은 안전한 이스케이핑을 원할 때 튜플/배열을 사용할 것이다.

```.js
something.findOne({
  order: [
    // Will escape username and validate DESC against a list of valid direction parameters
    ['username', 'DESC'],

    // Will order by max(age)
    sequelize.fn('max', sequelize.col('age')),

    // Will order by max(age) DESC
    [sequelize.fn('max', sequelize.col('age')), 'DESC'],

    // Will order by  otherfunction(`col1`, 12, 'lalala') DESC
    [sequelize.fn('otherfunction', sequelize.col('col1'), 12, 'lalala'), 'DESC'],

    // Will order by name on an associated User
    [User, 'name', 'DESC'],

    // Will order by name on an associated User aliased as Friend
    [{model: User, as: 'Friend'}, 'name', 'DESC'],

    // Will order by name on a nested associated Company of an associated User
    [User, Company, 'name', 'DESC'],
  ]
  // All the following statements will be treated literally so should be treated with care
  order: 'convert(user_name using gbk)'
  order: 'username DESC'
  order: sequelize.literal('convert(user_name using gbk)')
})
```
