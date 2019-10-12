# 시작하기

이 튜토리얼에서 기본을 배우기 위해 Sequelize를 간단하게 설정하는 방법을 배웁니다.

## 설치

Sequelize는 NPM 또는 yarn을 이용하여 사용할 수 있습니다.

```bash
$ npm install --save sequelize
```

또한 선택한 데이터베이스를 사용하기 위해 드라이버를 설치해야 합니다.

```bash
## 다음 중 하나를 실행(사용하는 디비에 맞게 설치)
$ npm install --save pg pg-hstore # Postgres
$ npm install --save mysql2 
$ npm install --save mariadb 
$ npm install --save sqlite3
$ npm install --save tedious # Microsoft SQL Server
```


## 연결 설정

데이터베이스를 생성하기 위해, 당신은 Sequelize 인스턴스를 생성해야 합니다. 생성자에게 하나의 연결 URI를 전달하거나 연결 파라미터 변수를 개별적으로 전달하여 Sequelize 인스턴스 생성이 가능합니다.

```js
const Sequelize = require('sequelize');

// Option 1: Passing parameters separately
const sequelize = new Sequelize('database', 'username', 'password', {
  host: 'localhost',
  dialect: /* one of 'mysql' | 'mariadb' | 'postgres' | 'mssql' */
});

// Option 2: Passing a connection URI
const sequelize = new Sequelize('postgres://user:pass@example.com:5432/dbname');
```

Sequelize생성자는 [API 레퍼런스](http://sequelize.readthedocs.io/en/latest/api/sequelize/)를 통해 사용할 수 있는 다양한 옵션을 확인 할 수 있습니다.


### 참고: SQLite 설정

만약 SQLite를 사용한다면 다음과 같이 사용하십시오.

```js
const sequelize = new Sequelize({
  dialect: 'sqlite',
  storage: 'path/to/database.sqlite'
});
```

### 참고: 커넥션 풀(프로덕션)

만약 당신이 단일 프로세스로 부터 데이터베이스를 연결한다면, 당신은 오직 하나의 Sequelize 인스턴스를 생성해야 합니다. Sequelize는 초기화할 때 커넥션 풀을 설정할 수 있습니다. 커넥션 풀은 생성자의 `옵션` 파라미터(`options.pool` 사용)를 통해 설정되고, 이것은 다음과 같이 예시를 따릅니다

```js
const sequelize = new Sequelize(/* ... */, {
  // ...
  pool: {
    max: 5,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
});
```

[API Reference for the Sequelize constructor](https://sequelize.org/master/class/lib/sequelize.js~Sequelize.html#instance-constructor-constructor)에서 더 많이 배울 수 있습니다. 만약 당신이 다수의 프로세스로부터 데이터베이스 연결을 한다면, 프로세스당 하나의 인스턴스를 생성해야 하며, 각각의 인스턴스는 최대 연결 풀 크기는 총 최대 크기를 준수해야 합니다. 예를들어, 만약 최대 커넥션이 90이고 3개의 프로세스를 가지고 있다면, Sequelize는 각 프로세스의 인스턴스는 최대 커넥션 풀 크기가 30입니다.

### 연결 테스트

당신은 `.authenticate()` 함수를 사용하여 옳바른 연결이 됬는지 테스트 할 수 있습니다.

```js
sequelize
  .authenticate()
  .then(() => {
    console.log('Connection has been established successfully.');
  })
  .catch(err => {
    console.error('Unable to connect to the database:', err);
  });
```

### 연결 종료

Sequelize는 기본적으로 연결이 열린 상태를 유지하며, 쿼리를 위해 같은 커넥션을 사용합니다. 커넥션 종료가 필요하다면, `sequelize.close()`를 호출합니다 (비동기이며 Promise 반환).

## 모델 테이블

한 모델은 `Sequelize.Model` 확장하는 클랙스입니다. 모델은 두 가지 반법으로 정의될 수 있습니다. 첫 번째 방법은 `Sequelize.Model.init(attributes, options)`를 이용합니다.

```js
const Model = Sequelize.Model;
class User extends Model {}
User.init({
  // 속성
  firstName: {
    type: Sequelize.STRING,
    allowNull: false
  },
  lastName: {
    type: Sequelize.STRING
    // allowNull 기본값 true
  }
}, {
  sequelize,
  modelName: 'user'
  // 옵션들
});
```

두 번째 방법은 `sequelize.define`을 이용합니다.

```js
const User = sequelize.define('user', {
  // 속성
  firstName: {
    type: Sequelize.STRING,
    allowNull: false
  },
  lastName: {
    type: Sequelize.STRING
    // allowNull 기본값 true
  }
}, {
  // 옵션들
});
```

내부적으로 `sequelize.define`는 `Model.init`을 호출합니다.

위의 코드는 Sequelize에게 `firstName` 및 `lastName` 필드를 가진 데이터베이스에서 `users` 이름을 가진 테이블을 바라보도록 합니다. 테이블 이름은 자동으로 복수형 형태로 생성하는게 기본값(복수 형태로 생성하기 위해 [inflection](https://www.npmjs.com/package/inflection) 라이브러리가 후드 아래에서 사용)입니다. `freezeTableName: true` 옵션을 사용하여 특정 모델은 이 행위를 중단할 수 있으며, [Sequelize 생성자](https://sequelize.org/master/class/lib/sequelize.js~Sequelize.html#instance-constructor-constructor) 옵션 `define`을 사용하여 모든 모델에 대해 이 행위를 중단할 수 있습니다.

Sequelize는 `id`(primary key), `createdAt` 그리고 `updatedAt` 필드를 기본값으로 가진 모델을 정의합니다. 이 행위는 바꿀 수 있습니다 (사용 가능한 옵션을 더 배우기 위해 API 레퍼런스를 확인하세요).

### 모델 기본옵션 변경

Sequelize 생성자는 정의된 모든 모델의 기본 옵션을 변경하는 `define` 옵션을 사용합니다. 

```js
const sequelize = new Sequelize(connectionURI, {
  define: {
    // `timestamps` 필드는 createdAt, updatedAt 필드 사용을 하지않습니다.
    // 이 필드의 기본값은 true이며, 지금은 사용하지 않습니다.
    timestamps: false
  }
});

// 여기는 timestamps가 false 이며, `createdAt`과 `updatedAt`를 생성하지 않습니다. 
class Foo extends Model {}
Foo.init({ /* ... */ }, { sequelize });

// 여기는 timestamps가 false 이며, `createdAt`과 `updatedAt`를 생성하지 않습니다. 
class Bar extends Model {}
Bar.init({ /* ... */ }, { sequelize, timestamps: true });
```

당신은 [model.init API 레퍼런스](https://sequelize.org/master/class/lib/model.js~Model.html#static-method-init) 또는 [sequelize.define API 레퍼런스](https://sequelize.org/master/class/lib/sequelize.js~Sequelize.html#instance-method-define)에서 모델생성 방법을 더 배울 수 있습니다.

## 데이터베이스와 모델 동기화

모델 정의에 따라 자동으로 테이블이 생성되기 원한다면, 다음과 같이 `sync` 메소드를 사용할 수 있습니다.

```js
// 참고: `force: true`를 사용하는 것은 이미 테이블이 존재한다면 드랍시킵니다.
User.sync({ force: true }).then(() => {
  // 데이터베이스의 `users` 테이블은 모델 정의에 해당합니다. 
  return User.create({
    firstName: 'John',
    lastName: 'Hancock'
  });
});
```

### 한 번에 모든 모델 동기화

모든 모델을 위해 `sync()` 호출 대신 각 모델들마다 sync를 호출하는 함수 `sequelize.sync()`를 호출합니다.

### 프로덕션을 위함 참고사항

프로덕션 환경에서 `sync()` 호출 대신 마이그레이션(복제)을 사용하는 것이 좋습니다. [마이그레이션](https://sequelize.org/master/manual/migrations.html) 가이드에서 더 배울 수 있습니다.

## 쿼리(조회)

아래는 간단한 조회 예제입니다.

```js
// 모든 유저 조회
User.findAll().then(users => {
  console.log("All users:", JSON.stringify(users, null, 4));
});

// 새로운 유저 생성
User.create({ firstName: "Jane", lastName: "Doe" }).then(jane => {
  console.log("Jane's auto-generated ID:", jane.id);
});

// Jane 이름 모두 삭제
User.destroy({
  where: {
    firstName: "Jane"
  }
}).then(() => {
  console.log("Done");
});

// 빈 값lastName을 가진 모든 필드를 Doe로 변경
User.update({ lastName: "Doe" }, {
  where: {
    lastName: null
  }
}).then(() => {
  console.log("Done");
});
```

sequelize는 쿼리를 위해 많은 옵션을 가지고 있습니다. 다음 튜토리얼에서 이것들을 배울 수 있습니다. 만약 당신이 필요하다면, 이 것은 저수준의 SQL 쿼리를 만들 수 있습니다.


## Promises와 async/await

Sequelize `.than`을 사용하여 Promise를 사용합니다. 이것은 노드 버전이 제공한다면 ES2017의 `async/await` 문법을 사용할 수 있습니다.

또한, 모든 Sequelize 프로미스는 [Bluebird](http://bluebirdjs.com/docs/getting-started.html) 프로미스이며, Bluebird의 다양한 API(예를들어, `finally`, `tap`, `tapCatch`, `map`, `mapSeries`, `etc`)를 사용할 수 있습니다. 만약 Bluebird의 특정 옵션을 설정하기를 원한다면, `Sequelize.Promise`와 함께 Sequelize에 의해 내부에서 사용된 Bluebird 생성자에 접근할 수 있습니다.


