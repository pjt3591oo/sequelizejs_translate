# 마이그레이션

소스코드 변경사항 관리를 위해 git/svn을 사용하는 것처럼, 데이터베이스의 변경사항을 추적하기 위해 마이그레이션을 사용할 수 있습니다. 마이그레이션을 통해 기존 데이터베이스를 다른 상태로 또는 그 반대로 전송할 수 있습니다. 이러한 상태 전환은 마이그레이션 파일에 저장됩니다. 마이그레이션 파일은 새 상태로 이동하는 방법과 이전 상태로 돌아 가기 위해 변경 사항을 되 돌리는 방법을 설명합니다.

[Sequelize CLI](https://github.com/sequelize/cli)가 필요합니다. CLI는 마이그레이션과 프로젝트 부트스트랩 지원을 제공합니다.

## The CLI

### CLI 설치

CLI 설치부터 시작하겠습니다. [여기서](https://github.com/sequelize/cli) 지침을 찾을 수 있습니다. 가장 선호되는 방법은 다음과 같이 로컬로 설치하는 것입니다

```bash
$ npm install --save sequelize-cli
```

### 부트스트랩핑

빈 프로젝트를 생성하기 위해 `init` 명령어를 사용할 수 있습니다.

```bash
$ npx sequelize-cli init
```

이것은 다음과 같은 폴더를 생성합니다.

* `config`, CLI에 데이터베이스 연결 방법을 알려주는 설정 파일 포함
* `models`, 프로젝트를 위한 모든 모델 파일들 포함
* `migrations`, 모든 마이그레이션 파일들 포함
* `seeders`, 모든 시드 파일들 포함

#### 설정

계속 진행하기 전에 CLI에 데이터베이스 연결 방법을 알려야합니다. 이를 위해 설정 구성 파일 `config/config.json`을 엽니다. 설정파일은 다음과 같습니다.

```json
{
  "development": {
    "username": "root",
    "password": null,
    "database": "database_development",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "test": {
    "username": "root",
    "password": null,
    "database": "database_test",
    "host": "127.0.0.1",
    "dialect": "mysql"
  },
  "production": {
    "username": "root",
    "password": null,
    "database": "database_production",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
}
```

이제이 파일을 편집하고 올바른 데이터베이스 정보 및 방언(dialect)을 설정하십시오. 객체의 키 (예 : "development")는 `model.index.js`에서 `process.env.NODE_ENV`와 일치시키기 위해 사용됩니다 (정의되지 않은 경우 "development"가 기본값).

**참고** : 데이터베이스가 아직 없으면 **db:create** 명령을 호출하면됩니다. 적절한 액세스 권한이 있으면 해당 데이터베이스가 작성됩니다.

### 첫 번쨰 모델 생성(그리고 마이그레이션)

CLI 설정파일을 올바르게 구성했으면 첫 번째 마이그레이션을 작성할 준비가되었습니다. 이것은 매우 간단한 명령어로 실행하 수 있습니다.

`model:generate` 명령어를 사용할 것 입니다. 두 가지 옵션을 반드시 포함해야 합니다.

* `name`, 모델의 이름
* `attributes`, 모델의 속성 리스트

`User` 모델을 생성해봅시다.

```bash
$ npx sequelize-cli model:generate --name User --attributes firstName:string,lastName:string,email:string
```

이것은 다음을 따릅니다.

* `models` 디렉터리(폴더)에 `user` 파일을 생성합니다.
* `migrations` 디렉터리(폴더)에 `XXXXXXXXXXXXXX-create-user.js`와 같은 이름을 가진 파일을 생성합니다.

**참고**: Sequelize는 모델 파일 만 사용하며 테이블 표현입니다. 반면, 마이그레이션 파일은 CLI에서 사용되는 해당 모델 또는 보다 구체적으로 해당 테이블의 변경입니다. 데이터베이스의 일부 변경에 대한 커밋 또는 로그처럼 마이그레이션을 처리하십시오.

### 마이그레이션 동작

이 단계까지는 데이터베이스에 아무것도 삽입하지 않았습니다. 첫 번째 모델 사용자에 필요한 모델 및 마이그레이션 파일을 만들었습니다. 이제 데이터베이스에서 해당 테이블을 실제로 만들려면 `db:migrate` 명령을 실행해야합니다.

```bash
$ npx sequelize-cli db:migrate
```

이것은 다음 스탭을 실행합니다.

* 데이터베이스에서 `SequelizeMeta`로 불리는 테이블을 안전하게 합니다. 이 테이블은 현재 데이터베이스에서 실행 된 마이그레이션을 기록하는 데 사용됩니다.
* 아직 실행되지 않은 마이그레이션 파일을 찾기 시작하십시오. `SequelizeMeta` 테이블을 확인하면 가능합니다. 이 경우 마지막 단계에서 생성한 `XXXXXXXXXXXXXX-create-user.js` 마이그레이션을 실행합니다.
* 마이그레이션 파일에 지정된대로 모든 열이있는 `Users`라는 테이블을 작성합니다.

### 마이그레이션 취소

이제 테이블이 생성되어 데이터베이스에 저장되었습니다. 마이그레이션을 사용하면 명령을 실행하여 이전 상태로 되돌릴 수 있습니다.

가장 최근의 마이그레이션을 되돌리기 위해 `dv:migrate:undo`를 사용할 수 있습니다.

```bash
$ npx sequelize-cli db:migrate:undo
```

`db:migrate:undo:all` 명령으로 모든 마이그레이션을 실행 취소하여 초기 상태로 되돌릴 수 있습니다. `--to` 옵션에 이름을 전달하여 특정 마이그레이션으로 되돌릴 수도 있습니다.

```bash
$ npx sequelize-cli db:migrate:undo:all --to XXXXXXXXXXXXXX-create-posts.js
```

### 첫 번째 피드 생성

일부 데이터를 몇 개의 테이블에 삽입하려고한다고 가정합니다. 이전 예제를 따라 가면 `User` 테이블에 대한 데모 사용자를 만들 수 있습니다.

모든 데이터 마이그레이션을 관리하기 위해 파종기(씨앗뿌리기)를 사용할 수 있습니다. 시드 파일은 데이터베이스 테이블을 샘플 데이터 또는 테스트 데이터로 채우는 데 사용할 수있는 일부 데이터 변경입니다. 

`User` 테이블에 데모 사용자를 추가 할 seed 파일을 만들어 봅시다.

```bash
$ npx sequelize-cli seed:generate --name demo-user
```

이 명령어는 `seeder` 디렉터리(폴더)에 씨드 파일을 생성합니다. 파일 이름은 `XXXXXXXXXXXXXX-demo-user.js`와 같습니다. 마이그레이션 파일과 동일한 `up/down` 시맨틱을 따릅니다.

이제이 파일을 편집하여 데모 사용자를 사용자 테이블에 삽입해야합니다.

```js
'use strict';

module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.bulkInsert('Users', [{
        firstName: 'John',
        lastName: 'Doe',
        email: 'demo@demo.com',
        createdAt: new Date(),
        updatedAt: new Date()
      }], {});
  },

  down: (queryInterface, Sequelize) => {
    return queryInterface.bulkDelete('Users', null, {});
  }
};
```

## 씨드 실행

마지막 스탭에서 씨드를 생성했습니다. 여전히 데이터베이스에 커밋되지 않았습니다. 커밋을 위해 간단한 명령을 실행해야합니다. 

```bash
$ npx sequelize-cli db:seed:all
```

이것은 씨드파일을 실행하고 `User` 테이블에 데모 유저 데이터를 추가합니다.

**참고**: 시더 실행은 `SequelizeMeta` 테이블을 사용하는 마이그레이션과 달리 어디에도 저장되지 않습니다. 이것을 무시하려면 저장 섹션을 읽으십시오

## 씨드 취소

시더는 스토리지를 사용하는 경우 취소 할 수 있습니다. 사용할 수있는 두 가지 명령이 있습니다.

가장 최신의 시드를 취소하고 싶다면

```bash
$ npx sequelize-cli db:seed:undo
```

특정 요소의 시드를 취소하고 싶다면

```bash
$ npx sequelize-cli db:seed:undo --seed name-of-seed-as-in-data
```

모든 시드를 취소하고 싶다면

```bash
$ npx sequelize-cli db:seed:undo:all
```

## Advance Topics

### 마이그레이션 스켈레톤

다음 스켈레톤은 일반적인 마이그레이션 파일을 보여줍니다.

```js
module.exports = {
  up: (queryInterface, Sequelize) => {
    // logic for transforming into the new state
  },

  down: (queryInterface, Sequelize) => {
    // logic for reverting the changes
  }
}
```

`migration:generate`를 사용하여이 파일을 생성 할 수 있습니다. 마이그레이션 디렉터리(폴더)에 `xxx-migration-skeleton.js` 파일을 생성합니다.

```bash
$ npx sequelize-cli migration:generate --name migration-skeleton
```

전달 된 `queryInterface` 오브젝트를 사용하여 데이터베이스를 수정할 수 있습니다. `Sequelize`는 `STRING` 또는 `INTEGER`와 같이 사용가능한 데이터 타입을 저장합니다. `up` 또는 `down` 함수는 `Promise`를 반환합니다. 예제를 살펴 보십쇼.

```js
module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.createTable('Person', {
        name: Sequelize.STRING,
        isBetaMember: {
          type: Sequelize.BOOLEAN,
          defaultValue: false,
          allowNull: false
        }
      });
  },
  down: (queryInterface, Sequelize) => {
    return queryInterface.dropTable('Person');
  }
}
```

다음은 트랜잭션을 사용하여 데이터베이스에서 두 가지 변경을 수행하여 실패시 모든 명령이 성공적으로 실행 또는 롤백되도록하는 마이그레이션의 예입니다.

```js
module.exports = {
    up: (queryInterface, Sequelize) => {
        return queryInterface.sequelize.transaction((t) => {
            return Promise.all([
                queryInterface.addColumn('Person', 'petName', {
                    type: Sequelize.STRING
                }, { transaction: t }),
                queryInterface.addColumn('Person', 'favoriteColor', {
                    type: Sequelize.STRING,
                }, { transaction: t })
            ])
        })
    },

    down: (queryInterface, Sequelize) => {
        return queryInterface.sequelize.transaction((t) => {
            return Promise.all([
                queryInterface.removeColumn('Person', 'petName', { transaction: t }),
                queryInterface.removeColumn('Person', 'favoriteColor', { transaction: t })
            ])
        })
    }
};
```

외래키를 가지고 있는 마이그레이션의 다음예입니다. 참조를 사용하여 외래키를 지정할 수 있습니다.

```js
module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.createTable('Person', {
      name: Sequelize.STRING,
      isBetaMember: {
        type: Sequelize.BOOLEAN,
        defaultValue: false,
        allowNull: false
      },
      userId: {
        type: Sequelize.INTEGER,
        references: {
          model: {
            tableName: 'users',
            schema: 'schema'
          }
          key: 'id'
        },
        allowNull: false
      },
    });
  },

  down: (queryInterface, Sequelize) => {
    return queryInterface.dropTable('Person');
  }
}
```

다음은 async / await를 사용하여 새 열에 고유 인덱스를 생성하는 마이그레이션의 예입니다.

```js
module.exports = {
  async up(queryInterface, Sequelize) {
    const transaction = await queryInterface.sequelize.transaction();
    try {
      await queryInterface.addColumn(
        'Person',
        'petName',
        {
          type: Sequelize.STRING,
        },
        { transaction }
      );
      await queryInterface.addIndex(
        'Person',
        'petName',
        {
          fields: 'petName',
          unique: true,
        },
        { transaction }
      );
      await transaction.commit();
    } catch (err) {
      await transaction.rollback();
      throw err;
    }
  },

  async down(queryInterface, Sequelize) {
    const transaction = await queryInterface.sequelize.transaction();
    try {
      await queryInterface.removeColumn('Person', 'petName', { transaction });
      await transaction.commit();
    } catch (err) {
      await transaction.rollback();
      throw err;
    }
  },
};
```

### .sequelizerc 파일

이것은 특수 설정 파일입니다. 일반적으로 CLI에 인수로 전달할 다음 옵션을 지정할 수 있습니다.

* `env`: 명령을 실행할 환경
* `config`: 설정파일 경로
* `options-path`: 추가 옵션이있는 JSON 파일의 경로
* `migrations-path`: 마이그레이션 디렉터리(폴더) 경로
* `seeders-path`: seeder 디렉터리(폴더) 경로
* `models-path`: models 디렉터리(폴더) 경로
* `url`: 사용할 데이터베이스 연결 문자열입니다. --config 파일 사용의 대안
* `debug`: 사용 가능한 경우 다양한 디버그 정보 표시

사용할 수있는 일부 시나리오.

* `migrations`, `models`, `seeders` 또는 `config` 디렉터리(폴더) 기본경로 재정의를 원할 때
*  `config.json`의 이름을 `database.json`과 같은 다른 이름으로 바꾸고 싶을 때


그리고 훨씬 더. 이 파일을 사용자 정의 구성에 사용하는 방법을 살펴 보겠습니다.

우선, 프로젝트의 루트 디렉토리에 빈 파일을 만들어 봅시다.

```js
const path = require('path');

module.exports = {
  'config': path.resolve('config', 'database.json'),
  'models-path': path.resolve('db', 'models'),
  'seeders-path': path.resolve('db', 'seeders'),
  'migrations-path': path.resolve('db', 'migrations')
}
```

이 구성을 사용하면 CLI에

* 설정 구성을 위한 `config/database.json` 사용
* `db/models`를 모델 폴더로 사용
* `db/seeders`를 모델 폴더로 사용
* `db/migrations`를 모델 폴더로 사용

### 동적 설정

설정 파일은 기본적으로 `config.json`이라는 JSON 파일입니다. 그러나 때로는 JSON 파일에서는 불가능한 일부 코드를 실행하거나 환경 변수에 액세스하려고합니다.

Sequelize CLI는 `JSON`과 `JS` 파일에서 읽을 수 있습니다. 이것은 `.sequelizerc` 파일과 함께 설정하 수 있습니다. 

첫 번째로 프로젝트 루트 경로에 `.sequelizerc` 파일을 생성합니다. 이 파일은 JS 파일의 구성 경로를 대체해야합니다. 

```js
const path = require('path');

module.exports = {
  'config': path.resolve('config', 'config.js')
}
```

Sequelize CLI는 설정 옵션을 가져오기 위해 `config/config.js`를 읽을 수 있습니다. 이 파일은 JS 파일이므로 모든 코드를 실행하고 최종 동적 구성 파일을 내보낼 수 있습니다.

예시 `config/config.js`파일

```js
const fs = require('fs');

module.exports = {
  development: {
    username: 'database_dev',
    password: 'database_dev',
    database: 'database_dev',
    host: '127.0.0.1',
    dialect: 'mysql'
  },
  test: {
    username: 'database_test',
    password: null,
    database: 'database_test',
    host: '127.0.0.1',
    dialect: 'mysql'
  },
  production: {
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    host: process.env.DB_HOSTNAME,
    dialect: 'mysql',
    dialectOptions: {
      ssl: {
        ca: fs.readFileSync(__dirname + '/mysql-ca-master.crt')
      }
    }
  }
};
```

### 바벨 사용

이제 `.sequelizerc` 파일 사용법을 알았습니다. 이제 ` ` 설정에서 babel을 사용하기 위해이 파일을 사용하는 방법을 살펴 보겠습니다. 이를 통해 ES6/ES7 구문으로 마이그레이션 및 시드를 작성할 수 있습니다.

첫 번째로 `babel-register`를 설치합니다.

```bash
$ npm i --save-dev babel-register
```

`.sequelizerc` 파일을 실행해라. 여기에는 sequelize-cli를 위해 변경하려는 구성이 포함될 수 있지만 코드베이스에 babel을 등록하기를 원합니다. 

```bash
$ touch .sequelizerc # Create rc file
```

이 파일에 `babel-register` 추가하라

```js
require("babel-register");

const path = require('path');

module.exports = {
  'config': path.resolve('config', 'config.json'),
  'models-path': path.resolve('models'),
  'seeders-path': path.resolve('seeders'),
  'migrations-path': path.resolve('migrations')
}
```

이제 CLI는 마이그레이션/시더 등에서 ES6/ES7 코드를 실행할 수 있습니다. 이는 `.babelrc` 구성에 따라 다릅니다. [babeljs.io](https://babeljs.io/)에서 자세한 내용을 읽으십시오.

### 환경 변수 사용

 CLI를 사용하여 `config/config.js`에 있는 환경 변수를 직접적으로 접근할 수 있습니다. `.sequelizerc`를 사용하여 구성에 `config/config.js`를 사용하도록 CLI에 지시 할 수 있습니다. 이 것은 마지막 세션에서 설명합니다.

환경 변수 속성을 노출할 수 있습니다.

```js
module.exports = {
  development: {
    username: 'database_dev',
    password: 'database_dev',
    database: 'database_dev',
    host: '127.0.0.1',
    dialect: 'mysql'
  },
  test: {
    username: process.env.CI_DB_USERNAME,
    password: process.env.CI_DB_PASSWORD,
    database: process.env.CI_DB_NAME,
    host: '127.0.0.1',
    dialect: 'mysql'
  },
  production: {
    username: process.env.PROD_DB_USERNAME,
    password: process.env.PROD_DB_PASSWORD,
    database: process.env.PROD_DB_NAME,
    host: process.env.PROD_DB_HOSTNAME,
    dialect: 'mysql'
  }
};
```

### dialect 옵션 선택

dialectOption을 지정하고 싶을 때, 일반적인 설정이라면 `config/config.json`에 추가하면됩니다. 때로는 dialectOptions를 얻기 위해 일부 코드를 실행하려는 경우 이러한 경우 동적 구성 파일을 사용해야합니다.

```js
{
    "production": {
        "dialect":"mysql",
        "dialectOptions": {
            "bigNumberStrings": true
        }
    }
}
```

### 프로덕션 사용

프로덕션 환경에서 CLI 및 마이그레이션 설정 사용에 대한 팁.

1) 구성 설정에 환경 변수를 사용합니다. 이것은 동적구성으로 더 좋은 수행결과를 얻습니다. 샘플 프로덕션 안전 구성은 다음과 같습니다.

```js
const fs = require('fs');

module.exports = {
  development: {
    username: 'database_dev',
    password: 'database_dev',
    database: 'database_dev',
    host: '127.0.0.1',
    dialect: 'mysql'
  },
  test: {
    username: 'database_test',
    password: null,
    database: 'database_test',
    host: '127.0.0.1',
    dialect: 'mysql'
  },
  production: {
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    host: process.env.DB_HOSTNAME,
    dialect: 'mysql',
    dialectOptions: {
      ssl: {
        ca: fs.readFileSync(__dirname + '/mysql-ca-master.crt')
      }
    }
  }
};
```

우리의 목표는 다양한 데이터베이스 비밀에 환경 변수를 사용하고 실수로 소스 제어에 체크인하지 않는 것입니다.

### 저장소

사용할 수있는 세 가지 유형의 저장소가 있습니다: `sequelize`, `json` 그리고 `none`.

* `sequelize`: sequelize 데이터베이스의 테이블에 마이그레이션 및 시드를 저장합니다.
* `json`: json 파일에 마이그레이션 및 시드 저장
* `none`: 마이그레이션/시드를 저장하지 않습니다

#### 마이그레이션 저장소

기본적으로 CLI는 실행 된 각 마이그레이션에 대한 항목을 포함하는 `SequelizeMeta`라는 테이블을 데이터베이스에 작성합니다. 이 동작을 변경하기 위해 구성 파일에 추가 할 수있는 세 가지 옵션이 있습니다. `migrationStorage`를 사용하여 마이그레이션에 사용할 스토리지 유형을 선택할 수 있습니다. `json`을 선택하면 `migrationStoragePath`를 사용하여 파일의 경로를 지정할 수 있습니다. 그렇지 않으면 CLI가 파일 `sequelize-meta.json`에 씁니다. `sequelize`를 사용하여 데이터베이스에 정보를 유지하지만 다른 테이블을 사용하려는 경우 `migrationStorageTableName`을 사용하여 테이블 이름을 변경할 수 있습니다. 또한 `migrationStorageTableSchema` 속성을 제공하여 `SequelizeMeta` 테이블에 대해 다른 스키마를 정의 할 수 있습니다.

```json
{
  "development": {
    "username": "root",
    "password": null,
    "database": "database_development",
    "host": "127.0.0.1",
    "dialect": "mysql",

    // Use a different storage type. Default: sequelize
    "migrationStorage": "json",

    // Use a different file name. Default: sequelize-meta.json
    "migrationStoragePath": "sequelizeMeta.json",

    // Use a different table name. Default: SequelizeMeta
    "migrationStorageTableName": "sequelize_meta",

    // Use a different schema for the SequelizeMeta table
    "migrationStorageTableSchema": "custom_schema"
  }
}
```

**참고**: **nonce** 저장소는 추천하지 않습니다. 이를 사용하기로 결정한 경우 마이그레이션이 수행하거나 수행하지 않은 작업에 대한 기록이 없다는 의미에 유의하십시오.

#### 시드 저장소

기본적으로 CLI는 실행 된 시드를 저장하지 않습니다. 이 동작을 변경하기로 선택한 경우 구성 파일에서 `seederStorage`를 사용하여 스토리지 타입을 변경할 수 있습니다. `json`을 선택하면 `seederStoragePath`를 사용하여 파일 경로를 지정하거나 CLI가 `sequelize-data.json` 파일에 기록합니다. sequelize를 사용하여 데이터베이스에 정보를 유지하려는 경우 `seederStorageTableName`을 사용하여 테이블 이름을 지정하거나 기본적으로 `SequelizeData`가 됩니다.

```json
{
  "development": {
    "username": "root",
    "password": null,
    "database": "database_development",
    "host": "127.0.0.1",
    "dialect": "mysql",
    // Use a different storage. Default: none
    "seederStorage": "json",
    // Use a different file name. Default: sequelize-data.json
    "seederStoragePath": "sequelizeData.json",
    // Use a different table name. Default: SequelizeData
    "seederStorageTableName": "sequelize_data"
  }
}
```

### 연결정보 문자열

데이터베이스를 정의하는 구성 파일이있는 `--config` 옵션의 대안으로 `--url` 옵션을 사용하여 연결 문자열을 전달할 수 있습니다. 예를 들면 다음과 같습니다.

```bash
$ npx sequelize-cli db:migrate --url 'mysql://root:password@mysql_host.com/database_name'
```

### dialect 별 옵션 전달

```json
{
    "production": {
        "dialect":"postgres",
        "dialectOptions": {
            // dialect options like SSL etc here
        }
    }
}
```

### 프로그래밍 방식의 사용

sequelize는 프로그래밍 방식으로 실행 후 처리, 마이그레이션 작업을 로깅하기 위해 [sister library](https://github.com/sequelize/umzug)을 가지고 있습니다.

## 쿼리 인터페이스

데이터베이스 스키마 변경하기 전에 설명된 `queryInterface` 객체를 사용하세요. 공개된 메소드의 전체 목록을 보려면 [CheckInterface API](https://sequelize.org/v5/class/lib/query-interface.js~QueryInterface.html) 검사를 지원합니다.