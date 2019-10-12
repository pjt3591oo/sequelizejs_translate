
(콜백 또는 이벤트의 주기로 알려진)후크은 sequelize가 실행되기 전후에 호출되는 함수이다. 예를들어 만약에 여러분들이 모델에 저장되기 이전에 값을 설정하고 싶다면 여러분들은 `beforeUpdate` 훅을 사용 할 수 있다.

후크의 전체 목록을 보기위해 [후크 API](http://docs.sequelizejs.com/en/v3/api/hooks)를 참조하세요

# Order of Operations(운영명령)

```js
(1)
  beforeBulkCreate(instances, options, fn)
  beforeBulkDestroy(options, fn)
  beforeBulkUpdate(options, fn)
(2)
  beforeValidate(instance, options, fn)
(-)
  validate
(3)
  afterValidate(instance, options, fn)
  - or -
  validationFailed(instance, options, error, fn)
(4)
  beforeCreate(instance, options, fn)
  beforeDestroy(instance, options, fn)
  beforeUpdate(instance, options, fn)
(-)
  create
  destroy
  update
(5)
  afterCreate(instance, options, fn)
  afterDestroy(instance, options, fn)
  afterUpdate(instance, options, fn)
(6)
  afterBulkCreate(instances, options, fn)
  afterBulkDestroy(options, fn)
  afterBulkUpdate(options, fn)
```

# Declaring Hooks(후크 선언)

후크에 사용되는 인자들은 참조에 의해 전달된다. 이것은 여러분이 값을 바꿔지는것, 삽입/업데이트 상태가 반영되는것을 의미한다. 후크는 비동기 액션을 가지고 있다. 이 경우 해당 함수는 promise를 반환해야만 한다.

후크를 추가하는 방법은 현재 세 가지가 있습니다.

```js
// 첫번째 방법 .define() 함수를 통한 방법
var User = sequelize.define('user', {
  username: DataTypes.STRING,
  mood: {
    type: DataTypes.ENUM,
    values: ['happy', 'sad', 'neutral']
  }
}, {
  hooks: {
    beforeValidate: function(user, options) {
      user.mood = 'happy'
    },
    afterValidate: function(user, options) {
      user.username = 'Toni'
    }
  }
})

// 두번째 방법 .hook() 함수를 통한 방법
User.hook('beforeValidate', function(user, options) {
  user.mood = 'happy'
})

User.hook('afterValidate', function(user, options) {
  return sequelize.Promise.reject("I'm afraid I can't let you do that!")
})

// 세번째 방법 direct 함수를 통한 방법
User.beforeCreate(function(user, options) {
  return hashPassword(user.password).then(function (hashedPw) {
    user.password = hashedPw;
  });
})

User.afterValidate('myHookAfter', function(user, options, fn) {
  user.username = 'Toni'
})
```


# Removing Hooks(후크 삭제)

오직 이름 파라미터가 있는 후크만 지울 수 있습니다.

```js
var Book = sequelize.define('book', {
  title: DataTypes.STRING
})

Book.addHook('afterCreate', 'notifyUsers', function(book, options) {
  // ...
})

Book.removeHook('afterCreate', 'notifyUsers')
```


# Global/ universal hooks (전역, 만능 후크)

전역 후크는 모든 모델을 위해 실행되는 후크입니다. 개발자들은 모든 모델에 대해 원하는 행동을 정의 할 수 있고, 특히 플러그인에 유용하다. 그들은 약간은 다른 의미가 있는 두가지의 방법을 통해 정의를 할 수 있습니다.


## Sequelize.options.define (default hook)

```js
var sequelize = new Sequelize(..., {
    define: {
        hooks: {
            beforeCreate: function () {
                // Do stuff
            }
        }
    }
});
```

이것은 실행됬을 떄 모델이 `beforeCreate` 후크를 가지고 있지 않은 모든 모델에 후크를 추가합니다.

```js
var User = sequelize.define('user');
var Project = sequelize.define('project', {}, {
    hooks: {
        beforeCreate: function () {
            // Do other stuff
        }
    }
});

User.create() // 전역 후크 실행
Project.create() // 전역 후크가 덮어쓰고 있기때문에 자기 자신의 후크 실행
```

## Sequelize.addHook (permanent hook)

```js
sequelize.addHook('beforeCreate', function () {
    // Do stuff
});
```

이 후크는 모델이 `beforeCreate` 지정 여부에 관계없이 항상 생성되기 이전에 실행됩니다..

```js
var User = sequelize.define('user');
var Project = sequelize.define('project', {}, {
    hooks: {
        beforeCreate: function () {
            // Do other stuff
        }
    }
});

User.create() // Runs the global hook
Project.create() // Runs its own hook, followed by the global hook
```

지역 후크는 한상 전역 후크 이전에 실행됩니다.


## Instance hooks

단일 객체를 수정 할 때마다 다음과 같은 후크가 방출된다.

```
beforeValidate
afterValidate or validationFailed
beforeCreate / beforeUpdate  / beforeDestroy
afterCreate / afterUpdate / afterDestroy
```

```js
// ...define ...
User.beforeCreate(function(user) {
  if (user.accessLevel > 10 && user.username !== "Boss") {
    throw new Error("You can't grant this user an access level above 10!")
  }
})
```

해당 예제는 에러를 반환한다.

```js
User.create({username: 'Not a Boss', accessLevel: 20}).catch(function(err) {
  console.log(err) // 여러분은 이 유저에게 accessLevel을 10을 줄 수 없다.
})
```

다음의 예제는 성공적으로 반환을 한다.

```js
User.create({username: 'Boss', accessLevel: 20}).then(function(user) {
  console.log(user) // user 객체는 username이 Bost이고 accessLevel이 20
})
```


## Model Hooks

떄떄로 여러분은 해당 모델이 가지고 있는 `bulkCreate, update, destoy`함수를 활용되어 두개 이상이 수정이 될 것이다. 다음중 하나를 사용할 경우 다음과 같이 표시가 됩니다.

```
beforeBulkCreate / beforeBulkUpdate / beforeBulkDestroy
afterBulkCreate / afterBulkUpdate / afterBulkDestroy
```

만약 여러분들은 각각의 레코드의 후크가 실행되기를 원한다면, 여러개의 후크와 함꼐 `individualHooks :true`를 전달할 수 있습니다.

```js
Model.destroy({ where: {accessLevel: 0}, individualHooks: true})
// 선택 된 레코드들 은 제거가 되고, 제거된 각각의 인스턴스들에 대해 beforeDestory, afterDestory가 발생할 것입니다.

Model.update({username: 'Toni'}, { where: {accessLevel: 0}, individualHooks: true})
// 선택 된 레코드들 은 수정이 되고, 수정된 각각의 인스턴스들에 대해 beforeDestory, afterDestory가 발생할 것입니다.
```

일부 모델 후크에는 유형에 따라 각 후크로 보내지는 두개 또는 세개의 매개 변수가 있습니다.

```js
Model.beforeBulkCreate(function(records, fields) {
  // records = .bulkCreate의 첫번째 인자로 전달됩니다
  // fields = .bulkCreate의 두번째 인자로 전달됩니다
})

Model.bulkCreate([
  {username: 'Toni'}, // 레코드 인자의 일부
  {username: 'Tobi'} // 레코드 인자의 일부
], ['username'] /* 필드인자의 일부 */)

Model.beforeBulkUpdate(function(attributes, where) {
  // attributes = Model.update의 첫번째 인자로 전달
  // where = Model.update의 두번째 인자로 전달
})

Model.update({gender: 'Male'} /*attributes 인자*/, { where: {username: 'Tom'}} /*where 인자*/)

Model.beforeBulkDestroy(function(whereClause) {
  // whereClause = Model.destroy의 첫번째 인자로 전달
})

Model.destroy({ where: {username: 'Tom'}} /*whereClause argument*/)
```

만약 여러분들이  `updatesOnDuplicate` 옵션과 함께 `Model.bulkCreate(...)`을 사용한다면, updateOnDuplicate 배열에서 포함되지 않은 필드에 대한 후크의 변경은 데이터 베이스에 적용되지 않습니다. 그러나 여러분이 원한다면 후크 내부에서 updatesOnDuplicate옵션을 바꾸는 것이 가능합니다.

```js
// updatesOnDuplicate과 함께 존재하는 유저의 수정
Users.bulkCreate([{ id: 1, isMemeber: true},
                 { id: 2, isMember: false}],
                 { updatesOnDuplicate: ['isMember']})

User.beforeBulkCreate(function (users, options) {
  users.forEach(function (user) {
    if (user.isMember) {
      user.memberSince = new Date()
    }
  })

  // updatesOnDuplicate로 존재하지 않는 memberSince를 memberSince로 덮어씌어라
  // 데이터 베이스에 저장된
  options.updatesOnDuplicate.push('memberSince')
})
```


# Associations

대부분의 후크는 몇 가지 점을 제외하면 연관된 인스턴스에 대해 동일한 작용을 할 것입니다.

1. add/set 함수를 사용하면 beforeUpdate / afterUpdate 후크가 실행됩니다.
2. beforeDestroy/afterDestroy후크를 호출하는 방법은 `onDelete: 'cascade'` 그리고 `hooks: true`옵션과 함께 관계를 가집니다.

```js
var Projects = sequelize.define('projects', {
  title: DataTypes.STRING
})

var Tasks = sequelize.define('tasks', {
  title: DataTypes.STRING
})

Projects.hasMany(Tasks, { o
```

이 코드는 Tasks 테이블에서 beforeDestroy/afterDestroy이 실행 될 것 입니다. sequelize는 기본적으로 쿼리문을 최대한 최적화하려고 시도를 합니다. delete에서 cascade를 호출 할 때, sequelize는 간단하게 다음과 같이 실행을 할 것입니다.

```
DELETE FROM `table` WHERE associatedIdentifier = associatedIdentifier.primaryKey
```

그러나 `hooks: true`를 명시적으로 추가하는 것은 최적화가 여러분들의 관심사가 아닌, 관계를 가진 각각의 객체들의 `select` 성능 그리고 올바른 매개 변수와 함께 호출된 하나의 후크에 의해 생성된 각각의 인스턴스가 삭제되는 성능이 관심사라고 말합니다.

만약에 여러분들은 `m:m`의 관계를 가진다면, 여러분들은 `remove` 호출 하는 모델을 통해 후크를 제거하는 것에 관심을 가질 것 입니다. 내부적으로, sequelize는 `model.destroy`을 사용하여 각각의 인스턴스를 통해 `before/afterDestory` 후크 대신에 `bulkDestroy` 호출합니다.

이것은 `remove`를 호출하기 위해 `{individualHooks: true}`을 전달하여 간단하게 해결 할 수 있습니다. 각각의 인스턴스들을 통해 각각의 인스턴스에서 remove를 호출하는 후크의 결과를 가집니다.


# A Note About Transactions

sequelize의 많은 연산을 통해 메소드의 옵션 매개 변수에 트랜잭션을 지정할 수 있습니다. 기존의 호출에서 트랜젝션이 지정되어있다면, 후크 함수에 옵션 매개변수에 보여집니다. 예를 들어 하나의 예를 고려해보자.

```js
// 우리는 비동기식 후크의 promise 스타일을 사용한다.
// 콜백.
User.hook('afterCreate', function(user, options) {
  // 'transaction'은 options.transaction에서 사용 가능하다.

  // 이 연산자는 같은 트랜젝션의 일부.
  // 원래 User.create를 호출
  return User.update({
    mood: 'sad'
  }, {
    where: {
      id: user.id
    },
    transaction: options.transaction
  });
});


sequelize.transaction(function(t) {
  User.create({
    username: 'someguy',
    mood: 'happy',
    transaction: t
  });
});
```

만약에 여러분들이 `User.update`를 호출했을 때 트랜젝션 옵션을 설정하지 않았다면, 커밋이 될때까지 데이터 베이스에 데이터가 존재하지 않을 것입니다.


## Internal Transactions

트랜젝션은 `Model.findOrCreate`와 같은 특정 작업을 위해 연산자들 사용하게 만들것을 인지하는 것이 매우 중요합니다.. 만약 여러분들의 후크가 데이터베이스의 객체에 의존하여 읽기/쓰기 연산을 실행 한다면, 또는 앞의 예제와 같이 오브젝트의 저장된 값을 수정할때, 여러분들은 항상 `transaction : options.transaction}` 요소를 사용해야 합니다.

만약 트랜젝션 연산중에 후크가 실행 된다면, 여러분들의 읽기/쓰기가 트랜젝션에 의존적인지 확인을 할 것이다. 여러분들은 `{transaction : null}`을 사용하여 간단하게 트랜젝션이 아닌 후크를 가질 수 있고, 이를통해 기본적인 행동을 기대 할 수 있을것입니다.
