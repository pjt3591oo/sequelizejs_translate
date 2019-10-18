# 범위

범위를 지정하면 나중에 쉽게 사용할 수있는 일반적으로 사용되는 쿼리를 정의 할 수 있습니다. 범위에는 일반 파인더와 동일한 속성 (`where`, `include`, `limit` 등)이 모두 포함될 수 있습니다.

## 정의

범위는 모델 정의에 정의하고 파인더 오브젝트일 수 있으며, 파인더 객체를 반환하는 함수일 수 있습니다. 기본 범위를 제외하고 오브젝트만 가능.

```js
class Project extends Model {}
Project.init({
  // Attributes
}, {
  defaultScope: {
    where: {
      active: true
    }
  },
  scopes: {
    deleted: {
      where: {
        deleted: true
      }
    },
    activeUsers: {
      include: [
        { model: User, where: { active: true }}
      ]
    },
    random () {
      return {
        where: {
          someNumber: Math.random()
        }
      }
    },
    accessLevel (value) {
      return {
        where: {
          accessLevel: {
            [Op.gte]: value
          }
        }
      }
    }
    sequelize,
    modelName: 'project'
  }
});
```

`addScope` 호출에 의해 모델을 정의한 후 스코프를 추가할 수 있습니다. 이는 포함이있는 범위에 특히 유용합니다. 포함의 모델은 다른 모델을 정의 할 때 정의되지 않을 수 있습니다.

기본 범위는 항상 적용됩니다. 앞의 코드에서 `Project.findAll()`은 다음과 같은 쿼리를 생성할 수 있습니다.

```sql
SELECT * FROM projects WHERE active = true
```

기본 범위는 `.unscoped()`와 `.scope(null)` 호출 또는 다른 범위를 호출하여 지울 수 있습니다.

```js
Project.scope('deleted').findAll(); // Removes the default scope
```

```sql
SELECT * FROM projects WHERE deleted = true
```

스코프가 정의된 모델에 범위를 추가하는 것이 가능합니다. 이를 통해 include, `attributes` 또는 `where` 정의가 중복되는 것을 피할 수 있습니다. 앞의 예를 사용하면, User 모델에 포함된 `active` 범위를 호출 (include 객체에 조건을 직접 지정하지 않습니다)

```js
activeUsers: {
  include: [
    { model: User.scope('active')}
  ]
}
```

## 사용법


범위는 모델 정의에서 `.scope`를 호출하여 적용되며, 하나 이상의 범위 이름을 전달합니다. `.scope`는 `.findAll`, `.update`, `.count`, `destory` 등 일반적인 메소드를 포함한 모델 객체를 반환합니다. 이 모델 인스턴스를 저장하고 나중에 다시 사용할 수 있습니다.

```js
const DeletedProjects = Project.scope('deleted');

DeletedProjects.findAll();
// some time passes

// let's look for deleted projects again!
DeletedProjects.findAll();
```

범위는 `.find`, `.findAll`, `.count`, `.update`, `.increment` 및 `.destroy`에 적용됩니다.

함수인 범위는 두가지 방법으로 호출할 수 있습니다. 범위가 인수를 취하지 않으면 정상적으로 호출 될 수 있습니다. 범위가 인수를 사용하는 경우 객체를 전달하십시오.

```js
Project.scope('random', { method: ['accessLevel', 19]}).findAll();
```

```sql
SELECT * FROM projects WHERE someNumber = 42 AND accessLevel >= 19
```

## 병합

범위 배열을 `.scope`에 전달하거나 범위를 연속 인수로 전달하여 여러 범위를 동시에 적용 할 수 있습니다.

```js
// These two are equivalent
Project.scope('deleted', 'activeUsers').findAll();
Project.scope(['deleted', 'activeUsers']).findAll();
```

```sql
SELECT * FROM projects
INNER JOIN users ON projects.userId = users.id
WHERE projects.deleted = true
AND users.active = true
```


기본 범위와 함께 다른 범위를 적용하려면 `defaultScope` 키를 `.scope`에 전달하십시오.

```js
Project.scope('defaultScope', 'deleted').findAll();
```

```sql
SELECT * FROM projects WHERE active = true AND deleted = true
```

여러 범위를 호출 할 때 후속 범위의 키는 병합 될 `where` 및 `include`을 제외하고 이전 범위 ([Object.assign](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)과 유사)를 덮어 씁니다. 두 가지 범위를 고려하세요.

```js
{
  scope1: {
    where: {
      firstName: 'bob',
      age: {
        [Op.gt]: 20
      }
    },
    limit: 2
  },
  scope2: {
    where: {
      age: {
        [Op.gt]: 30
      }
    },
    limit: 10
  }
}
```

`.scope('scope1', 'scope2')`를 호출하는 것은 다음과 같이 쿼리를 생성합니다.

```sql
WHERE firstName = 'bob' AND age > 30 LIMIT 10
```

`scope2`가`limit`와`age`를 어떻게 덮어 쓰는 반면`firstName`은 유지됩니다. `limit`,`offset`,`order`,`paranoid`,`lock` 및`raw` 필드는 덮어 쓰여지고`where`는 얕게 병합됩니다 (동일한 키가 덮어 쓰기 됨). `include`의 병합 전략은 나중에 논의 될 것이다.

여러 적용된 범위의 속성 키는 `attributes.exclude`가 항상 유지되는 방식으로 병합됩니다. 이를 통해 여러 범위를 병합하고 최종 범위에서 민감한 필드를 유출하지 않습니다.

범위가 지정된 모델의 `findAll` (및 유사한 파인더)에 찾기 오브젝트를 직접 전달할 때 동일한 병합 논리가 적용됩니다.

```js
Project.scope('deleted').findAll({
  where: {
    firstName: 'john'
  }
})
```

```sql
WHERE deleted = true AND firstName = 'john'
```

여기서 '삭제 된'범위는 파인더와 병합됩니다. `where : {firstName : 'john', deleted : false}`를 파인더에게 전달하면`deleted` 범위가 덮어 쓰기됩니다.

### Merging includes(병합 포함)

포함되는 모델에 따라 포함이 재귀 적으로 병합됩니다. 이것은 v5에 추가 된 매우 강력한 병합이며 예제로 더 잘 이해됩니다.

Foo, Bar, Baz 및 Qux의 네 가지 모델을 고려하십시오.

```js
class Foo extends Model {}
class Bar extends Model {}
class Baz extends Model {}
class Qux extends Model {}
Foo.init({ name: Sequelize.STRING }, { sequelize });
Bar.init({ name: Sequelize.STRING }, { sequelize });
Baz.init({ name: Sequelize.STRING }, { sequelize });
Qux.init({ name: Sequelize.STRING }, { sequelize });
Foo.hasMany(Bar, { foreignKey: 'fooId' });
Bar.hasMany(Baz, { foreignKey: 'barId' });
Baz.hasMany(Qux, { foreignKey: 'bazId' });
```

이제 Foo에 정의 된 다음 네 가지 범위를 고려하십시오.

```js
{
  includeEverything: {
    include: {
      model: this.Bar,
      include: [{
        model: this.Baz,
        include: this.Qux
      }]
    }
  },
  limitedBars: {
    include: [{
      model: this.Bar,
      limit: 2
    }]
  },
  limitedBazs: {
    include: [{
      model: this.Bar,
      include: [{
        model: this.Baz,
        limit: 2
      }]
    }]
  },
  excludeBazName: {
    include: [{
      model: this.Bar,
      include: [{
        model: this.Baz,
        attributes: {
          exclude: ['name']
        }
      }]
    }]
  }
}
```

이 네 가지 범위는 `Foo.scope('includeEverything', 'limitedBars', 'limitedBazs', 'excludeBazName').findAll()`을 호출하여 쉽게 병합 할 수 있습니다. 이는 다음을 호출하는 것과 완전히 같습니다.

```js
Foo.findAll({
  include: {
    model: this.Bar,
    limit: 2,
    include: [{
      model: this.Baz,
      limit: 2,
      attributes: {
        exclude: ['name']
      },
      include: this.Qux
    }]
  }
});
```

네 가지 스코프가 어떻게 하나로 통합되었는지 관찰하십시오. 범위 포함은 포함되는 모델을 기반으로 병합됩니다. 한 범위에 모델 A가 포함되고 다른 범위에 모델 B가 포함 된 경우 병합 된 결과에는 모델 A와 B가 모두 포함됩니다. 반면에, 두 범위 모두 동일한 모델 A를 포함하지만 다른 옵션 (예 : 중첩 포함 또는 다른 속성)을 포함하는 경우 위에 표시된대로 재귀 적으로 병합됩니다.


위에 설명 된 병합은 범위에 적용되는 순서에 관계없이 동일한 방식으로 작동합니다. 특정 옵션이 두 가지 다른 범위로 설정된 경우에만 순서가 달라집니다. 각 범위가 다른 일을하기 때문에 위의 예에서는 그렇지 않습니다.

병합 전략은 `.findAll`, `.findOne`등에 전달된 옵션과 같은 방식으로 동작합니다.

## Associations(관계)

Sequelize에는 관계와 관련하여 서로 다른 두 가지 범위 개념이 있습니다. 차이점은 미묘하지만 중요합니다.

* **Association scopes** 관계에서 getting, setting할 때 기본 속성을 지정할 수있습니다. - 다형성 관계을 구현할 때 유용합니다. 이 범위는 연관된 모델 함수 `get`, `set`, `add` 및 `create`을 사용할 때 두 모델 간의 연관에서만 호출됩니다.

* **Scopes on associated models** 연관을 페치 할 때 기본 및 기타 범위를 적용하고 연관을 작성할 때 범위가 지정된 모델을 전달할 수 있습니다. 이 범위는 모델의 정규 찾기 및 연관을 통한 찾기에 모두 적용됩니다.연관을 페치 할 때 기본 및 기타 범위를 적용하고 연관을 작성할 때 범위가 지정된 모델을 전달할 수 있습니다. 이 범위는 모델의 정규 찾기 및 연관을 통한 찾기에 모두 적용됩니다. 

예를 들어, Post 및 Comment 모델을 고려하십시오. 댓글은 여러 다른 모델 (이미지, 비디오 등)과 연관되어 있으며 댓글과 다른 모델 간의 연관은 다형성입니다. 즉, 댓글은 외래 키 commentable_id 외에도 'commentable'열을 저장합니다.

다형성 연관은 연관 범위로 구현 될 수 있습니다.

```js
this.Post.hasMany(this.Comment, {
  foreignKey: 'commentable_id',
  scope: {
    commentable: 'post'
  }
});
```

`post.getComments()`를 호출할 때, `WHERE commentable = 'post'`를 자동으로 호출합니다. 마찬가지로 게시물에 새 댓글을 추가 할 때 댓글 작성 기능은 자동으로 `post`로 설정됩니다. 연결 범위는 프로그래머가 걱정할 필요없이 백그라운드에서 살아야하기 때문에 비활성화 할 수 없습니다. 보다 완전한 다형성 예제는 [Association scopes](https://sequelize.org/master/manual/associations.html#scopes)를 참조하십시오.

그런 다음 게시물에는 활성 게시물 만 표시하는 기본 범위가 있습니다.

```js
where: { active: true }
```

이 범위는 관련 모델 (포스트)에 있으며 주석 처리 가능한 범위와 같은 연결에는 없습니다. `Post.findAll()`을 호출 할 때 기본 범위가 적용되는 것과 마찬가지로 `User.getPosts()`를 호출 할 때도 적용됩니다. 그러면 해당 사용자의 활성 게시물 만 반환됩니다.

기본 범위를 비 활성화하기 위해, `scope: null`을 getter에 전달합니다 

```js
User.getPosts({scope: null})
```

마찬가지로 다른 범위의 적용을 원한다면, `.scope`에 다음과 같이 배열을 전달합니다.

```js
User.getPosts({ scope: ['scope1', 'scope2']});
```

```js
class Post extends Model {}
Post.init(attributes, {
  defaultScope: {
    where: {
      active: true
    }
  },
  scopes: {
    deleted: {
      where: {
        deleted: true
      }
    }
  },
  sequelize,
});

User.hasMany(Post); // regular getPosts association
User.hasMany(Post.scope('deleted'), { as: 'deletedPosts' });
```

연관된 모델의 범위에 대한 바로 가기 메소드를 작성하려는 경우 범위가 지정된 모델을 연관에 전달할 수 있습니다. 사용자에 대해 삭제 된 모든 게시물을 가져 오는 바로 가기를 고려하십시오. 

```js
User.getPosts(); // WHERE active = true
User.getDeletedPosts(); // WHERE deleted = true
```