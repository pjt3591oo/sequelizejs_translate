# 인스턴스



## 비 영구적인 인스턴스 작성

정의된 클래스의 인스턴스를 만들려면 다음과 같이 할 수 있습니다. 인식할 수 있습니다. 만약 과거에 **`Ruby`** 언어를 했다면 알 수 있습니다. `build` 메서드를 사용하는 것은 저장되지 않은 객체를 반환하며, 이 객체는 명시적으로 저장해야 합니다.

```js
const project = Project.build({
  title: 'my awesome project',
  description: 'woot woot. this will make me a rich man'
})

const task = Task.build({
  title: 'specify the project idea',
  description: 'bla',
  deadline: new Date()
})
```

생성된 인스턴스들은 정의될 때 자동으로 기본값을 가져옵니다.

```js
// first define the model
class Task extends Model {}
Task.init({
  title: Sequelize.STRING,
  rating: { type: Sequelize.TINYINT, defaultValue: 3 }
}, { sequelize, modelName: 'task' });

// now instantiate an object
const task = Task.build({title: 'very important task'})

task.title  // ==> 'very important task'
task.rating // ==> 3
```

데이터베이스에 저장하려면 `save` 메서드를 사용하고 필요한 경우 이벤트를 이용할 수 있습니다.

```js
project.save().then(() => {
  // my nice callback stuff
})

task.save().catch(error => {
  // mhhh, wth!
})

// you can also build, save and access the object with chaining:
Task
  .build({ title: 'foo', description: 'bar', deadline: new Date() })
  .save()
  .then(anotherTask => {
    // you can now access the currently saved task with the variable anotherTask... nice!
  })
  .catch(error => {
    // Ooops, do some error-handling
  })
```



## 영구적인 인스턴스 생성

`.build()`로 생성된 인스턴스는 데이터베이스에 저장하기 위해 `.save()` 호출을 명시해야 하지만, `.create()`는해당 요구 사항을 모두 생략하고 일단 호출되면 인스턴스의 데이터를 자동으로 저장합니다.

```js
Task.create({ title: 'foo', description: 'bar', deadline: new Date() }).then(task => {
  // you can now access the newly created task via the variable task
})
```

`create()` 메소드를 통해 설정할 수 있는 속성을 정의할 수 있습니다. 이것은 사용자가 채울 수 있는 양식 기반으로 데이터베이스 항목을 작성하는 경우에 매우 유용합니다. 예를들어 `User` 모델을 사용하여 사용자 이름과 주소만 설정하고 관리자 플래그는 설정하지 못하도록 제한할 수 있습니다.

```js
User.create({ username: 'barfooz', isAdmin: true }, { fields: [ 'username' ] }).then(user => {
  // let's assume the default of isAdmin is false:
  console.log(user.get({
    plain: true
  })) // => { username: 'barfooz', isAdmin: false }
})
```



## 인스턴스의 업데이트 / 저장 / 지속

이제 일부 값을 업데이트하고 데이터 베이스에 변경사항을 저장하겠습니다. 두 가지 방법이 있습니다.

```js
// way 1
task.title = 'a very different title now'
task.save().then(() => {})

// way 2
task.update({
  title: 'a very different title now'
}).then(() => {})
```

`save`가 호출될 때 컬럼 이름을 배열로 전달함으로 써 저장할 속성을 정의할 수 있습니다. 이것은 이전에 정의 된 객체 기반으로  속성을 설정할 때 유용합니다. 예를들면 웹 앱 형식을 통해 객체의 값을 얻는 경우 유용합니다. 또한. 이것은 `update`를 위해 다음과 같이 내부적으로도 사용합니다. 

```js
task.title = 'foooo'
task.description = 'baaaaaar'
task.save({fields: ['title']}).then(() => {
 // title will now be 'foooo' but description is the very same as before
})

// The equivalent call using update looks like this:
task.update({ title: 'foooo', description: 'baaaaaar'}, {fields: ['title']}).then(() => {
 // title will now be 'foooo' but description is the very same as before
})
```

`save`를 호출할 때 바뀔 속성이 없을 경우, 이 메서드는 실행하지 않습니다.



## 파괴 / 삭제 인스턴스

객체를 생성하고 참조하면, 데이터베이스로부터 그것을 제거할 수 있습니다. `destroy`가 관련된 메서드입니다.

```js
Task.create({ title: 'a task' }).then(task => {
  // now you see me...
  return task.destroy();
}).then(() => {
 // now i'm gone :)
})
```

`paranoid` 옵션이 true라면, 이 객체는 삭제되지 않습니다. 대신에 `deletedAt` 컬럼이 현재의 타임스탬프로 설정됩니다. 강제로 삭제하기 위해 `destroy` 호출시 `force:true`를 전달하면 됩니다.

```js
task.destroy({ force: true })
```

`paranoid` 모드에서 객체 삭제(soft delete) 후, 같은 PK를 가진 새로운 인스턴스를 기존의 인스턴스를 삭제하기 전까지 생성할 수 없습니다. 

> 참고: soft delete란, 데이터를 완전히 지우는 것이 아닌 deletedAt 컬럼을 이용하여 데이터는 남겨두되 언제 삭제됬는지 명시하는 것, hard delete란 데이터를 완전히 제거하여 데이터베이스에 남아있지 않도록 하는 방법. 
>
> 데이터의 역사성을 위해 soft delete를 사용하는 방법을 권장한다.



## 소프트 삭제 인스턴스 재정장

만약, `paranoid: true`로 soft 삭제 된 인스턴스를 다시 저장하고 싶다면 `restore`  메서드를 사용합니다.

```js
Task.create({ title: 'a task' }).then(task => {
  // now you see me...
  return task.destroy();
}).then((task) => {
  // now i'm gone, but wait...
  return task.restore();
})
```



## 여러작업 (하나의 호출로 여러 로우에 대한 생성, 업데이트 그리고 삭제)

단일 인스턴스 업데이트 외에 여러 인스턴스를 한 번에 생성, 업데이트 및 삭제할 수 있습니다. 그 함수는 다음과 같습니다.

* `Model.bulkCreate`
* `Model.update`
* `Model.destroy`

여러 인스턴스로 작업하는 동안, `DAO` 인스턴스를 반환하지 않습니다. bulkCreate는 모델/DAO를 배열형태로 반환합니다. 그것들은 `create`와 다르게 **`auto increment`** 속성을 가지고 있지 않습니다. `update`와 `destory`는 영향을 받은 행을 반환합니다.

먼저, bulkCreate를 봅시다.

```js
User.bulkCreate([
  { username: 'barfooz', isAdmin: true },
  { username: 'foo', isAdmin: true },
  { username: 'bar', isAdmin: false }
]).then(() => { // Notice: There are no arguments here, as of right now you'll have to...
  return User.findAll();
}).then(users => {
  console.log(users) // ... in order to get the array of user objects
})
```

여러 행을 삽입하고 모든 컬럼 반환 (오직 postges만)

```js
User.bulkCreate([
  { username: 'barfooz', isAdmin: true },
  { username: 'foo', isAdmin: true },
  { username: 'bar', isAdmin: false }
], { returning: true }) // will return all columns for each row inserted
.then((result) => {
  console.log(result);
});
```

여러 행을 삽입하고 특정 컬럼들만 반환 (오직 postgres만)

```js
User.bulkCreate([
  { username: 'barfooz', isAdmin: true },
  { username: 'foo', isAdmin: true },
  { username: 'bar', isAdmin: false }
], { returning: ['username'] }) // will return only the specified columns for each row inserted
.then((result) => {
  console.log(result);
});
```

한번에 다수의 행 업데이터

```js
Task.bulkCreate([
  {subject: 'programming', status: 'executing'},
  {subject: 'reading', status: 'executing'},
  {subject: 'programming', status: 'finished'}
]).then(() => {
  return Task.update(
    { status: 'inactive' }, /* set attributes' value */
    { where: { subject: 'programming' }} /* where criteria */
  );
}).then(([affectedCount, affectedRows]) => {
  // Notice that affectedRows will only be defined in dialects which support returning: true

  // affectedCount will be 2
  return Task.findAll();
}).then(tasks => {
  console.log(tasks) // the 'programming' tasks will both have a status of 'inactive'
})
```

그리고 조건에 따라 그들을 삭제...

```js
Task.bulkCreate([
  {subject: 'programming', status: 'executing'},
  {subject: 'reading', status: 'executing'},
  {subject: 'programming', status: 'finished'}
]).then(() => {
  return Task.destroy({
    where: {
      subject: 'programming'
    },
    truncate: true /* this will ignore where and truncate the table instead */
  });
}).then(affectedRows => {
  // affectedRows will be 2
  return Task.findAll();
}).then(tasks => {
  console.log(tasks) // no programming, just reading :(
})
```

사용자로부터 직접적으로 값의 접근을 허용하는 경우 실제로 삽입하려는 컬럼을 제한하는 것이 좋습니다. `bulkCreate()`는 두 번째 인자로 옵션객체를 받습니다. 이 객체는 `fields` 인자(배열)를 사용하여 명시적으로 만드려는 필드를 알 수 있습니다.

```js
User.bulkCreate([
  { username: 'foo' },
  { username: 'bar', admin: true}
], { fields: ['username'] }).then(() => {
  // nope bar, you can't be admin!
})
```

`bulkCreate`는 레코드를 삽입하는 빠른 방법으로 만들어 졌지만, 한번에 여러 행을 추가할 때 `validate: true`를 이용하여 각 컬럼마다 정의된 유효셩 검사를 할 수 있습니다.

```js
class Tasks extends Model {}
Tasks.init({
  name: {
    type: Sequelize.STRING,
    validate: {
      notNull: { args: true, msg: 'name cannot be null' }
    }
  },
  code: {
    type: Sequelize.STRING,
    validate: {
      len: [3, 10]
    }
  }
}, { sequelize, modelName: 'tasks' })

Tasks.bulkCreate([
  {name: 'foo', code: '123'},
  {code: '1234'},
  {name: 'bar', code: '1'}
], { validate: true }).catch(errors => {
  /* console.log(errors) would look like:
  [
    { record:
    ...
    name: 'SequelizeBulkRecordError',
    message: 'Validation error',
    errors:
      { name: 'SequelizeValidationError',
        message: 'Validation error',
        errors: [Object] } },
    { record:
      ...
      name: 'SequelizeBulkRecordError',
      message: 'Validation error',
      errors:
        { name: 'SequelizeValidationError',
        message: 'Validation error',
        errors: [Object] } }
  ]
  */
})
```



## 인스턴스의 값

인스턴스를 가져오면, 추가항목이 많이 있습니다. 그러한 것들을 숨기고 원하는 정보만 가져오기 위해 `plain: true`를 이용하여 오직 값만 가져올 수 있습니다.

```js
Person.create({
  name: 'Rambow',
  firstname: 'John'
}).then(john => {
  console.log(john.get({
    plain: true
  }))
})

// result:

// { name: 'Rambow',
//   firstname: 'John',
//   id: 1,
//   createdAt: Tue, 01 May 2012 19:12:16 GMT,
//   updatedAt: Tue, 01 May 2012 19:12:16 GMT
// }
```

**힌트**: `JSON.stringify(instance)`를 이용하여 JSON으로 변환할 수 있습니다. 이것은 기본적으로 값과 매우 유사한 값을 반환합니다.



## 인스턴스 리로딩

인스턴스를 동기화 해야하는 경우, `reloac` 메서드를 사용할 수 있습니다. 데이터베이스에서 현재 데이터를 가져와 메소드가 호출 된 모델의 속성을 겹쳐 씁니다.

```js
Person.findOne({ where: { name: 'john' } }).then(person => {
  person.name = 'jane'
  console.log(person.name) // 'jane'

  person.reload().then(() => {
    console.log(person.name) // 'john'
  })
})
```



## 증가

동시성 문제가 발생하지 않고 인스턴스 값을 늘리려면 `increment`를 사용할 수 있습니다.

먼저 필드와 추가 할 값을 정의 할 수 있습니다.

```js
User.findByPk(1).then(user => {
  return user.increment('my-integer-field', {by: 2})
}).then(user => {
  // Postgres will return the updated user by default (unless disabled by setting { returning: false })
  // In other dialects, you'll want to call user.reload() to get the updated instance...
})
```

다음으로 추가할 값을 여러 필드로 정의 할 수 있습니다.

```js
User.findByPk(1).then(user => {
  return user.increment([ 'my-integer-field', 'my-very-other-field' ], {by: 2})
}).then(/* ... */)
```

마지막으로 필드와 증감을 포함하여 객체를 정의 할 수 있습니다.

```js
User.findByPk(1).then(user => {
  return user.increment({
    'my-integer-field':    2,
    'my-very-other-field': 3
  })
}).then(/* ... */)
```



## 감소

동시성 문제가 발생하지 않고 인스턴스 값을 줄이려면 `decrement`를 사용할 수 있습니다.

먼저 필드와 추가 할 값을 정의 할 수 있습니다.

```js
User.findByPk(1).then(user => {
  return user.decrement('my-integer-field', {by: 2})
}).then(user => {
  // Postgres will return the updated user by default (unless disabled by setting { returning: false })
  // In other dialects, you'll want to call user.reload() to get the updated instance...
})
```

다음으로 추가할 값을 여러 필드로 정의 할 수 있습니다.

```js
User.findByPk(1).then(user => {
  return user.decrement([ 'my-integer-field', 'my-very-other-field' ], {by: 2})
}).then(/* ... */)
```

마지막으로 필드와 증감을 포함하여 객체를 정의 할 수 있습니다.

```js
User.findByPk(1).then(user => {
  return user.decrement({
    'my-integer-field':    2,
    'my-very-other-field': 3
  })
}).then(/* ... */)
```

