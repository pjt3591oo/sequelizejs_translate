
# 설치

Sequelize는 NPM을 통하여 사용할 수 있다.

```
$ npm install --save sequelize

# 다음 중 하나를 실행(사용하는 디비에 맞게 설치)
$ npm install --save pg pg-hstore
$ npm install --save mysql // mysql, mariadb 사용 할 경우
$ npm install --save sqlite3
$ npm install --save tedious // MSSQL사용 할 경우
```


# 연결 설정

Sequelize는 초기화시 연결풀을 설정 하므로 데이터베이스당 하나의 인스턴스만 생성하는 것이 이상적입니다.


```.js
var sequelize = new Sequelize('database', 'username', 'password', {
  host: 'localhost',
  dialect: 'mysql'|'mariadb'|'sqlite'|'postgres'|'mssql',

  pool: {
    max: 5,
    min: 0,
    idle: 10000
  },

  // 오직 SQLite만
  storage: 'path/to/database.sqlite'
});

// 또는 간단하게 연결하기 위한 URI를 사용할 수 있다.
var sequelize = new Sequelize('postgres://user:pass@example.com:5432/dbname');
```

Sequelize생성자는 [API 레퍼런스](http://sequelize.readthedocs.io/en/latest/api/sequelize/)를 통해 사용할 수 있는 다양한 옵션들을 확인 할 수 있다.


# 첫번째 모델

`sequelize.define('name', {attributes}, {options})`와 같이 모델 정의

```.js
var User = sequelize.define('user', {
  firstName: {
    type: Sequelize.STRING,
    field: 'first_name' // Will result in an attribute that is firstName when user facing but first_name in the database
  },
  lastName: {
    type: Sequelize.STRING
  }
}, {
  freezeTableName: true // 모델의 테이블 이름(user)은 해당 모델 이름과 같다

});

User.sync({force: true}).then(function () {
  // 테이블 생성
  return User.create({
    firstName: 'John',
    lastName: 'Hancock'
  });
});
```

더 다양한 옵션은 [API 레퍼런스](http://sequelize.readthedocs.io/en/latest/api/model/)에서 찾을 수 있다.


## 어플리케이션 다양한 모델 옵션

Sequelize 생성자는 정의된 모든 모델들의 기본 옵션인 `define`옵션을 가지고 있다.

```.js
var sequelize = new Sequelize('connectionUri', {
  define: {
    timestamps: false // 기본값인 true
  }
});

var User = sequelize.define('user', {}); // timestamps is false by default
var Post = sequelize.define('post', {}, {
  timestamps: true // timestamps는 이제 true이다.
});
```


## Promises

Sequelize는 비동기의 흐름을 제어하기 위해 Promises를 사용한다. 만약 여러분들이 promise 사용 하는 방법이 익숙하지 않는다면 [here](https://github.com/wbinnssmith/awesome-promises) 또는 [here](http://bluebirdjs.com/docs/why-promises.html)를 들어가서 복습을 하면 좋은 시간을 보낼 수 있습니다.

기본적으로 하나의 promise 어느 지점에서의 값을 보여준다. "나는 여러분들에게 어느 지점에서 결과나 오류를 줄 것이라고 약속합니다.
이것이 의미하는 것은

```.js
// 이렇게 쓰면 안된다!
user = User.findOne()

console.log(user.get('firstName'));
```

적용되지 않을 것이다. `user`는 promise 객체이기 때문에 DB에서 가져온 데이터 행이 아닙니다. promise를 사용하는 올바른 방법은 다음과 같습니다.

```.js
User.findOne().then(function (user) {
    console.log(user.get('firstName'));
});
```

여러분들은 promise를 다루는 방법과 그것이 어떻게 작동하는 지에 대해서 다음 [bluebird API 레퍼런스](http://bluebirdjs.com/docs/api-reference.html)를 참조할 것이다.
특히, 여러분들은 아마도 `.all`을 많이 사용할 것이다.
