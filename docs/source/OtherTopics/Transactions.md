# 트랜잭션

Sequelize는 트랜잭션 사용의 두 가지 방법을 제공합니다.

1. **Managed**, Promise 체인의 결과에 따라 자동으로 트랜잭션을 커밋 또는 롤백하고 (CLS가 활성화 된 경우) 콜백 내의 모든 호출에 트랜잭션을 전달합니다.

2. **Unmanaged**, 커밋, 롤백 및 트랜잭션을 사용자에게 전달

주요 차이점은 **Managed** 트랜잭션은 Promise가 반환 될 것으로 예상되는 콜백을 사용하고 **Unmanaged** 트랜잭션은 Promise을 반환한다는 것입니다.

## Managed transaction (auto-callback)

Managed 트랜잭션은 커밋 또는 롤백을 자동으로 처리합니다. Managed 트랜잭션을 시작할 때 `sequelize.transaction`에 콜백을 전달하여 트랜잭션을 사용합니다.

트랜잭션으로 전달 된 콜백이 어떻게 Promise 체인을 반환하는지 확인하고 `t.commit()` 또는 `t.rollback()`을 명시 적으로 호출하지 않습니다. 리턴 된 체인의 모든 Promise가 성공적으로 해결되면 트랜잭션이 commit 됩니다. 만약, 하나 또는 다수의 Promise가 reject 되면, 트랜잭션은 rollback 됩니다.

```js
return sequelize.transaction(t => {

  // chain all your queries here. make sure you return them.
  return User.create({
    firstName: 'Abraham',
    lastName: 'Lincoln'
  }, {transaction: t}).then(user => {
    return user.setShooter({
      firstName: 'John',
      lastName: 'Boothe'
    }, {transaction: t});
  });

}).then(result => {
  // Transaction has been committed
  // result is whatever the result of the promise chain returned to the transaction callback
}).catch(err => {
  // Transaction has been rolled back
  // err is whatever rejected the promise chain returned to the transaction callback
});
```

### 롤백 오류 발생

Managed 트랜잭션을 사용할 때는 트랜잭션을 수동으로 커밋하거나 롤백해서는 안됩니다. 모든 쿼리가 성공적이지만 여전히 트랜잭션을 롤백하려는 경우 (예 : 유효성 검사 실패로 인해) 체인을 중단하고 거부하는 오류를 발생시켜야합니다.

```js
return sequelize.transaction(t => {
  return User.create({
    firstName: 'Abraham',
    lastName: 'Lincoln'
  }, {transaction: t}).then(user => {
    // Woops, the query was successful but we still want to roll back!
    throw new Error();
  });
});
```

### 모든 쿼리들에게 트랜잭션을 자동으로 전달 

앞의 예에서, 여전히 트랜잭션을 `{ transaction: t }`을 두 번째 인자로 수동으로 전달합니다. 모든 쿼리에 대해 자동으로 트랜잭션을 전달하기 위해 [CLS](https://github.com/othiym23/node-continuation-local-storage) (continuation local storage) 모듈을 설치하고 고유 한 코드로 네임 스페이스를 인스턴스화해야합니다.

```js
const cls = require('continuation-local-storage');
const namespace = cls.createNamespace('my-very-own-namespace');
```

CLS를 사용하려면 sequelize 생성자의 정적 메서드를 사용하여 sequelize에 사용할 네임 스페이스를 알려야합니다.

```js
const Sequelize = require('sequelize');
Sequelize.useCLS(namespace);

new Sequelize(....);
```


useCLS () 메소드는 sequelize 인스턴스가 아닌 생성자에 있습니다. 모든 인스턴스는 네임스페이스를 공유하고 CLS는 전부 또는 아무것도 아닙니다. 일부 인스턴스에 대해서만 활성화 할 수 없습니다.

CLS는 콜백을위한 스레드 로컬 스토리지처럼 작동합니다. 이것이 실제로 의미하는 것은 다른 콜백 체인은 CLS 네임스페이스를 사용하여 로컬 변수를 접근할 수 있습니다. CLS를 활성화 한 sequelize는 트랜잭션을 생성할 때 네임스페이스에서 `transaction` 속성을 설정할 수 있습니다. 콜백 체인 내에 설정된 변수는 해당 체인 전용이므로 여러 개의 동시 트랜잭션이 동시에 존재할 수 있습니다.

```js
sequelize.transaction((t1) => {
  namespace.get('transaction') === t1; // true
});

sequelize.transaction((t2) => {
  namespace.get('transaction') === t2; // true
});
```

모든 쿼리는 네임 스페이스에서 트랜잭션을 자동으로 찾기 때문에 대부분의 경우 namespace.get ( 'transaction')에 직접 액세스 할 필요가 없습니다.

```js
sequelize.transaction((t1) => {
  // With CLS enabled, the user will be created inside the transaction
  return User.create({ name: 'Alice' });
});
```

`Sequelize.useCLS()`를 사용한 후에는 sequelize에서 반환 된 모든 Promise가 CLS 컨텍스트를 유지하기 위해 패치됩니다. CLS는 복잡한 주제입니다. [cls-bluebird](https://www.npmjs.com/package/cls-bluebird)에 대한 자세한 내용은 CLS와 함께 블루 버드 Promise을 만드는 데 사용되는 패치입니다.

**참고**: CLS는 cls-hooked 패키지를 사용할 때 현재 async/await만 지원합니다.

## 동시 / 부분 트랜잭션

일련의 쿼리 내에서 동시 트랜잭션을 수행하거나 일부 트랜잭션을 트랜잭션에서 제외시킬 수 있습니다. `{transaction :} `옵션을 사용하여 쿼리가 속하는 트랜잭션을 제어합니다.

**경고**: SQLite는 동시에 둘 이상의 트랜잭션을 지원하지 않습니다.

### CLS를 사용하지 않는경우

```js
sequelize.transaction((t1) => {
  return sequelize.transaction((t2) => {
    // With CLS enable, queries here will by default use t2
    // Pass in the `transaction` option to define/alter the transaction they belong to.
    return Promise.all([
        User.create({ name: 'Bob' }, { transaction: null }),
        User.create({ name: 'Mallory' }, { transaction: t1 }),
        User.create({ name: 'John' }) // this would default to t2
    ]);
  });
});
```

## Isolation(격리) 수준


트랜잭션을 시작할 때 사용할 수 있는 격리 수준:

```js
Sequelize.Transaction.ISOLATION_LEVELS.READ_UNCOMMITTED // "READ UNCOMMITTED"
Sequelize.Transaction.ISOLATION_LEVELS.READ_COMMITTED // "READ COMMITTED"
Sequelize.Transaction.ISOLATION_LEVELS.REPEATABLE_READ  // "REPEATABLE READ"
Sequelize.Transaction.ISOLATION_LEVELS.SERIALIZABLE // "SERIALIZABLE"
```

기본적으로 sequelize는 데이터베이스의 격리 수준을 사용합니다. 다른 격리 수준을 원한다면, 첫 번째 인자로 원하는 격리수준을 전달합니다.

```js
return sequelize.transaction({
  isolationLevel: Sequelize.Transaction.ISOLATION_LEVELS.SERIALIZABLE
  }, (t) => {

  // your transactions

  });
```

격리 수준은 `Sequelize` 인스턴스를 초기화 할 때 전역으로 설정하거나 모든 트랜잭션에 대해 로컬로 설정할 수 있습니다.

```js
// globally
new Sequelize('db', 'user', 'pw', {
  isolationLevel: Sequelize.Transaction.ISOLATION_LEVELS.SERIALIZABLE
});

// locally
sequelize.transaction({
  isolationLevel: Sequelize.Transaction.ISOLATION_LEVELS.SERIALIZABLE
});
```

**참고**: MSSQL의 경우 SET ISOLATION LEVEL 쿼리가 기록되지 않습니다.

## Unmanaged 트랜잭션(then-callback)

Unmanaged 트랜잭션은 rollback과 commit을 수동으로 동작합니다. 만약 하지 않는다면, 트랜잭션은 타임아웃 에러가 발생합니다. Unmanaged 트랜잭션을 시작하기 위해 콜백(여전히 옵션 객체를 전달할 수 있습니다.)과 Promise로 반환된 `then` 없이 `sequelize.transaction()`을 호출합니다. `commit()` 및 `rollback()`은 promise를 반환합니다.

```js
return sequelize.transaction().then(t => {
  return User.create({
    firstName: 'Bart',
    lastName: 'Simpson'
  }, {transaction: t}).then(user => {
    return user.addSibling({
      firstName: 'Lisa',
      lastName: 'Simpson'
    }, {transaction: t});
  }).then(() => {
    return t.commit();
  }).catch((err) => {
    return t.rollback();
  });
});
```

## 다른 sequelize 메소드와 사용하기

`transaction` 옵션은 대부분의 다른 옵션과 함께 제공됩니다. 다른 옵션은 보통 메서드의 첫 번째 인자로 전달합니다. `.create`, `.update()` 등과 같은 값을 갖는 메소드의 경우 `transaction`은 두 번째 인수의 옵션으로 전달합니다. 확실하지 않은 경우 서명을 확인하는 데 사용하는 방법에 대한 API 설명서를 참조하십시오

## 커밋 훅 이후

`Transaction` 객체는 commit 될 때 트래킹을 허용합니다.

`afterCommit` 훅은 Managed 또느 UnManaged 트랜잭션 객체를 추가할 수 있습니다.

```js
sequelize.transaction(t => {
  t.afterCommit((transaction) => {
    // Your logic
  });
});

sequelize.transaction().then(t => {
  t.afterCommit((transaction) => {
    // Your logic
  });

  return t.commit();
})
```

afterCommit에 전달 된 함수는 트랜잭션을 작성한 Promise 체인이 해결되기 전에 해결할 Promise을 선택적으로 리턴 할 수 있습니다.

트랜잭션이 롤백되면 `afterCommit` 후크가 발생하지 않습니다.

`afterCommit` 후크는 표준 후크와 달리 트랜잭션의 리턴 값을 수정하지 않습니다.

`afterCommit` 후크를 모델 후크와 함께 사용하여 트랜잭션 외부에서 인스턴스가 저장되고 사용 가능한시기를 알 수 있습니다.

```js
model.afterSave((instance, options) => {
  if (options.transaction) {
    // Save done within a transaction, wait until transaction is committed to
    // notify listeners the instance has been saved
    options.transaction.afterCommit(() => /* Notify */)
    return;
  }
  // Save done outside a transaction, safe for callers to fetch the updated model
  // Notify
})
```

## Locks(락)

트랜잭션 내 쿼리는 잠금으로 수행 할 수 있습니다.

```js
return User.findAll({
  limit: 1,
  lock: true,
  transaction: t1
})
```

트랜잭션 내의 쿼리는 잠긴 행을 건너 뛸 수 있습니다.

```js
return User.findAll({
  limit: 1,
  lock: true,
  skipLocked: true,
  transaction: t2
})
```