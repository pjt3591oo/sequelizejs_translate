# Dialects

Sequelize는 특정 데이터베이스들과 의존성을 가지지 않습니다. 이것은 당신의 프로젝트에 사용하는 연결 라이브러리 사용을 위해 설치해야 됨을 의미합니다.

## MySQL

MySQL과 함께 sequelize가 잘 동작하기 위해 `mysql2@^1.5.2` 이상의 버전을 설치해야 합니다. 완료하면 다음과 같이 사용할 수 있습니다.

```js
const sequelize = new Sequelize('database', 'username', 'password', {
  dialect: 'mysql'
})
```

**참고**: `dialectOptions` 파라미터 설정에 의해 dialects 라이브러리에 직접적으로 옵션을 전달할 수 있습니다.

## MariaDB

MariaDB를 위한 라이브러리는 `mariadb` 입니다.

```js
const sequelize = new Sequelize('database', 'username', 'password', {
  dialect: 'mariadb',
  dialectOptions: {connectTimeout: 1000} // mariadb 커넥터 옵션
})
```

또는 연결 문자열 사용

```js
const sequelize = new Sequelize('mariadb://user:password@example.com:9821/database')
```

## SQLite

적합한 SQLite 사용을 위해 `sqlite3@^4.0.0`이 필요합니다. Sequelize 설정은 다음과 같습니다.

```js
const sequelize = new Sequelize('database', 'username', 'password', {
  // sqlite! now!
  dialect: 'sqlite',

  // the storage engine for sqlite
  // - default ':memory:'
  storage: 'path/to/database.sqlite'
})
```

또는 경로형태의 연결 문자열 사용할 수 있습니다.

```js
const sequelize = new Sequelize('sqlite:/home/abs/path/dbname.db')
const sequelize = new Sequelize('sqlite:relativePath/dbname.db')
```

## PostgreSQL

postgreSQL을 위해, `pg@^7.0.0`, `pg-hstore` 두개의 라이브러리가 필요합니다. 당신은 데이터베이스 종류만 정의하면 됩니다.

```js
const sequelize = new Sequelize('database', 'username', 'password', {
  // gimme postgres, please!
  dialect: 'postgres'
})
```

유닉스 도메인 소켓을 통해 연결하려면, 호스트 옵션에서 소켓 디렉터리의 경로를 지정합니다.

소켓 경로는 `/`로 시작합니다.

```js
const sequelize = new Sequelize('database', 'username', 'password', {
  // gimme postgres, please!
  dialect: 'postgres',
  host: '/path/to/socket_directory'
})
```

## MSSQL

MSSQL을 위한 라이브러리는 `tedious@^6.0.0`입니다. 당신은 데이터베이스 종류만 정의하면 됩니다. 


> 참고 : `tedious@^6.0.0`을 사용하려면 `dialectOptions-object` 내부의 옵션설정인 `options-object` 내에 MSSQL 관련 옵션을 중첩해야합니다.

```js
const sequelize = new Sequelize('database', 'username', 'password', {
  dialect: 'mssql',
  dialectOptions: {
    options: {
      useUTC: false,
      dateFirst: 1,
    }
  }
})
```