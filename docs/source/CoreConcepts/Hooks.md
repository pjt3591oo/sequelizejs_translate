# 훅스(후크)

(콜백 또는 이벤트의 주기로 알려진)후크은 sequelize가 실행되기 전후에 호출되는 함수이다. 예를들어 만약에 여러분들이 모델에 저장되기 이전에 값을 설정하고 싶다면 여러분들은 `beforeUpdate` 훅을 사용 할 수 있다.

**참고**: 인스턴스와 함께 후크를 사용할 수 없습니다. 후쿠는 모델과 함께 사용됩니다.

#후크의 전체 목록을 보기위해 [후크 API](https://github.com/sequelize/sequelize/blob/master/lib/hooks.js#L7)를 참조하세요

## Order of Operations(운영명령)

```js
(1)
  beforeBulkCreate(instances, options)
  beforeBulkDestroy(options)
  beforeBulkUpdate(options)
(2)
  beforeValidate(instance, options)
(-)
  validate
(3)
  afterValidate(instance, options)
  - or -
  validationFailed(instance, options, error)
(4)
  beforeCreate(instance, options)
  beforeDestroy(instance, options)
  beforeUpdate(instance, options)
  beforeSave(instance, options)
  beforeUpsert(values, options)
(-)
  create
  destroy
  update
(5)
  afterCreate(instance, options)
  afterDestroy(instance, options)
  afterUpdate(instance, options)
  afterSave(instance, options)
  afterUpsert(created, options)
(6)
  afterBulkCreate(instances, options)
  afterBulkDestroy(options)
  afterBulkUpdate(options)
```

## Declaring Hooks(후크 선언)

후크에 사용되는 인자들은 참조에 의해 전달된다. 이것은 여러분이 값을 바꿔지는것, 삽입/업데이트 상태가 반영되는것을 의미한다. 후크는 비동기 액션을 가지고 있다. 이 경우 해당 함수는 promise를 반환해야만 한다.

후크를 추가하는 방법은 현재 세 가지가 있습니다.

```js
// Method 1 via the .init() method
class User extends Model {}
User.init({
  username: DataTypes.STRING,
  mood: {
    type: DataTypes.ENUM,
    values: ['happy', 'sad', 'neutral']
  }
}, {
  hooks: {
    beforeValidate: (user, options) => {
      user.mood = 'happy';
    },
    afterValidate: (user, options) => {
      user.username = 'Toni';
    }
  },
  sequelize
});

// Method 2 via the .addHook() method
User.addHook('beforeValidate', (user, options) => {
  user.mood = 'happy';
});

User.addHook('afterValidate', 'someCustomName', (user, options) => {
  return Promise.reject(new Error("I'm afraid I can't let you do that!"));
});

// Method 3 via the direct method
User.beforeCreate((user, options) => {
  return hashPassword(user.password).then(hashedPw => {
    user.password = hashedPw;
  });
});

User.afterValidate('myHookAfter', (user, options) => {
  user.username = 'Toni';
});
```


## Removing Hooks(후크 삭제)

오직 이름 파라미터가 있는 후크만 지울 수 있습니다.

```js
class Book extends Model {}
Book.init({
  title: DataTypes.STRING
}, { sequelize });

Book.addHook('afterCreate', 'notifyUsers', (book, options) => {
  // ...
});

Book.removeHook('afterCreate', 'notifyUsers');
```

같은 이름과 함께 후크를 가질 수 있습니다. `removeHook()` 호출하는 것은 그것들을 모두 제거할 수 있습니다.

## Global/ universal hooks (전역, 만능 후크)

전역 후크는 모든 모델을 위해 실행되는 후크입니다. 개발자들은 모든 모델에 대해 원하는 행동을 정의 할 수 있고, 특히 플러그인에 유용하다. 그들은 약간은 다른 의미가 있는 두가지의 방법을 통해 정의를 할 수 있습니다.

### Default Hooks (Sequelize.options.define)

```js
const sequelize = new Sequelize(..., {
    define: {
        hooks: {
            beforeCreate: () => {
              // Do stuff
            }
        }
    }
});
```

이것은 실행됬을 떄 모델이 `beforeCreate` 후크를 가지고 있지 않은 모든 모델에 후크를 추가합니다.

```js
class User extends Model {}
User.init({}, { sequelize });
class Project extends Model {}
Project.init({}, {
    hooks: {
        beforeCreate: () => {
            // Do other stuff
        }
    },
    sequelize
});

User.create() // Runs the global hook
Project.create() // Runs its own hook (because the global hook is overwritten)
```

### Permanent Hooks (Sequelize.addHook)

```js
sequelize.addHook('beforeCreate', () => {
    // Do stuff
});
```

이 후크는 모델이 `beforeCreate` 지정 여부에 관계없이 항상 생성되기 이전에 실행됩니다. 로컬 후크는 항상 글로벌 후크보다 먼저 실행합니다.

```js
User.init({}, { sequelize });
class Project extends Model {}
Project.init({}, {
    hooks: {
        beforeCreate: () => {
            // Do other stuff
        }
    },
    sequelize
});

User.create() // Runs the global hook
Project.create() // Runs its own hook, followed by the global hook
```

영구 후크는 `Sequelize.options` 에서도 정의 될 수 있습니다.

```js
new Sequelize(..., {
    hooks: {
        beforeCreate: () => {
            // do stuff
        }
    }
});
```

### Connection Hooks

Sequelize는 데이터베이스 연결을 얻거나 해제하기 직전과 직후에 실행되는 네 가지 후크를 제공합니다.

```
beforeConnect(config)
afterConnect(connection, config)
beforeDisconnect(connection)
afterDisconnect(connection)
```

이 후크는 데이터베이스 정보를 비동기 적으로 확보해야하거나 저수준 데이터베이스 연결이 작성된 후 직접 액세스해야하는 경우 유용 할 수 있습니다.

예를 들어, 토큰 저장소에서 데이터베이스 비밀번호를 비동기 적으로 얻을 수 있으며 새로운 자격 증명으로 Sequelize의 구성 객체를 변경할 수 있습니다.

```js
sequelize.beforeConnect((config) => {
    return getAuthToken()
        .then((token) => {
             config.password = token;
         });
    });
```

연결 풀은 모든 모델에서 공유되므로 이러한 후크는 영구 전역 후크로만 선언 될 수 잇습니다.

## Instance hooks

단일 객체를 수정 할 때마다 다음과 같은 후크가 방출된다.

```
beforeValidate
afterValidate or validationFailed
beforeCreate / beforeUpdate / beforeSave  / beforeDestroy
afterCreate / afterUpdate / afterSave / afterDestroy
```

```js
// ...define ...
User.beforeCreate(user => {
  if (user.accessLevel > 10 && user.username !== "Boss") {
    throw new Error("You can't grant this user an access level above 10!")
  }
})
```

해당 예제는 에러를 반환한다.

```js
User.create({username: 'Not a Boss', accessLevel: 20}).catch(err => {
  console.log(err); // You can't grant this user an access level above 10!
});
```

다음의 예제는 성공적으로 반환을 한다.

```js
User.create({username: 'Boss', accessLevel: 20}).then(user => {
  console.log(user); // user object with username as Boss and accessLevel of 20
});
```


### Model Hooks

떄떄로 여러분은 해당 모델이 가지고 있는 `bulkCreate, update, destoy`함수를 활용되어 두개 이상이 수정이 될 것이다. 다음중 하나를 사용할 경우 다음과 같이 표시가 됩니다.

```
beforeBulkCreate(instances, options)
beforeBulkUpdate(options)
beforeBulkDestroy(options)
afterBulkCreate(instances, options)
afterBulkUpdate(options)
afterBulkDestroy(options)
```

만약 여러분들은 각각의 레코드의 후크가 실행되기를 원한다면, 여러개의 후크와 함꼐 `individualHooks :true`를 전달할 수 있습니다.

경고: 개별 후크를 사용하는 경우 후크가 호출되기 전에 업데이트되거나 제거 된 모든 인스턴스가 메모리에로드됩니다. Sequelize가 개별 후크로 처리 할 수있는 인스턴스 수는 사용 가능한 메모리에 의해 제한됩니다.

```js
Model.destroy({ where: {accessLevel: 0}, individualHooks: true});
// Will select all records that are about to be deleted and emit before- + after- Destroy on each instance

Model.update({username: 'Toni'}, { where: {accessLevel: 0}, individualHooks: true});
// Will select all records that are about to be updated and emit before- + after- Update on each instance
```

hook 메소드의 `option` 인자는 해당 메소드 또는 복제 및 확장 버전에 제공된 두 번째 인수입니다.

```js
Model.beforeBulkCreate((records, {fields}) => {
  // records = the first argument sent to .bulkCreate
  // fields = one of the second argument fields sent to .bulkCreate
})

Model.bulkCreate([
    {username: 'Toni'}, // part of records argument
    {username: 'Tobi'} // part of records argument
  ], {fields: ['username']} // options parameter
)

Model.beforeBulkUpdate(({attributes, where}) => {
  // where - in one of the fields of the clone of second argument sent to .update
  // attributes - is one of the fields that the clone of second argument of .update would be extended with
})

Model.update({gender: 'Male'} /*attributes argument*/, { where: {username: 'Tom'}} /*where argument*/)

Model.beforeBulkDestroy(({where, individualHooks}) => {
  // individualHooks - default of overridden value of extended clone of second argument sent to Model.destroy
  // where - in one of the fields of the clone of second argument sent to Model.destroy
})

Model.destroy({ where: {username: 'Tom'}} /*where argument*/)
```

만약 여러분들이  `updatesOnDuplicate` 옵션과 함께 `Model.bulkCreate(...)`을 사용한다면, updateOnDuplicate 배열에서 포함되지 않은 필드에 대한 후크의 변경은 데이터 베이스에 적용되지 않습니다. 그러나 여러분이 원한다면 후크 내부에서 updatesOnDuplicate옵션을 바꾸는 것이 가능합니다.

```js
// Bulk updating existing users with updateOnDuplicate option
Users.bulkCreate([
  { id: 1, isMember: true },
  { id: 2, isMember: false }
], {
  updateOnDuplicate: ['isMember']
});

User.beforeBulkCreate((users, options) => {
  for (const user of users) {
    if (user.isMember) {
      user.memberSince = new Date();
    }
  }

  // Add memberSince to updateOnDuplicate otherwise the memberSince date wont be
  // saved to the database
  options.updateOnDuplicate.push('memberSince');
});
```


## Associations

대부분의 후크는 몇 가지 점을 제외하면 연관된 인스턴스에 대해 동일한 작용을 할 것입니다.

1. add/set 함수를 사용하면 beforeUpdate / afterUpdate 후크가 실행됩니다.
2. beforeDestroy/afterDestroy후크를 호출하는 방법은 `onDelete: 'cascade'` 그리고 `hooks: true`옵션과 함께 관계를 가집니다.

```js
class Projects extends Model {}
Projects.init({
  title: DataTypes.STRING
}, { sequelize });

class Tasks extends Model {}
Tasks.init({
  title: DataTypes.STRING
}, { sequelize });

Projects.hasMany(Tasks, { onDelete: 'cascade', hooks: true });
Tasks.belongsTo(Projects);
```

이 코드는 Tasks 테이블에서 beforeDestroy/afterDestroy이 실행 될 것 입니다. sequelize는 기본적으로 쿼리문을 최대한 최적화하려고 시도를 합니다. delete에서 cascade를 호출 할 때, sequelize는 간단하게 다음과 같이 실행을 할 것입니다.

```
DELETE FROM `table` WHERE associatedIdentifier = associatedIdentifier.primaryKey
```

그러나 `hooks: true`를 명시적으로 추가하는 것은 최적화가 여러분들의 관심사가 아닌, 관계를 가진 각각의 객체들의 `select` 성능 그리고 올바른 매개 변수와 함께 호출된 하나의 후크에 의해 생성된 각각의 인스턴스가 삭제되는 성능이 관심사라고 말합니다.

만약에 여러분들은 `m:m`의 관계를 가진다면, 여러분들은 `remove` 호출 하는 모델을 통해 후크를 제거하는 것에 관심을 가질 것 입니다. 내부적으로, sequelize는 `model.destroy`을 사용하여 각각의 인스턴스를 통해 `before/afterDestory` 후크 대신에 `bulkDestroy` 호출합니다.

이것은 `remove`를 호출하기 위해 `{individualHooks: true}`을 전달하여 간단하게 해결 할 수 있습니다. 각각의 인스턴스들을 통해 각각의 인스턴스에서 remove를 호출하는 후크의 결과를 가집니다.


## A Note About Transactions

sequelize의 많은 연산을 통해 메소드의 옵션 매개 변수에 트랜잭션을 지정할 수 있습니다. 기존의 호출에서 트랜젝션이 지정되어있다면, 후크 함수에 옵션 매개변수에 보여집니다. 예를 들어 하나의 예를 고려해보자.

```js
// Here we use the promise-style of async hooks rather than
// the callback.
User.addHook('afterCreate', (user, options) => {
  // 'transaction' will be available in options.transaction

  // This operation will be part of the same transaction as the
  // original User.create call.
  return User.update({
    mood: 'sad'
  }, {
    where: {
      id: user.id
    },
    transaction: options.transaction
  });
});


sequelize.transaction(transaction => {
  User.create({
    username: 'someguy',
    mood: 'happy',
    transaction
  });
});
```

만약에 여러분들이 `User.update`를 호출했을 때 트랜젝션 옵션을 설정하지 않았다면, 커밋이 될때까지 데이터 베이스에 데이터가 존재하지 않을 것입니다.


### Internal Transactions

트랜젝션은 `Model.findOrCreate`와 같은 특정 작업을 위해 연산자들 사용하게 만들것을 인지하는 것이 매우 중요합니다.. 만약 여러분들의 후크가 데이터베이스의 객체에 의존하여 읽기/쓰기 연산을 실행 한다면, 또는 앞의 예제와 같이 오브젝트의 저장된 값을 수정할때, 여러분들은 항상 `transaction : options.transaction}` 요소를 사용해야 합니다.

만약 트랜젝션 연산중에 후크가 실행 된다면, 여러분들의 읽기/쓰기가 트랜젝션에 의존적인지 확인을 할 것이다. 여러분들은 `{transaction : null}`을 사용하여 간단하게 트랜젝션이 아닌 후크를 가질 수 있고, 이를통해 기본적인 행동을 기대 할 수 있을것입니다.
