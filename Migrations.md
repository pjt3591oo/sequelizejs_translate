sequelize `2.0.0`버전은 gulp 그리고 sequelize-cli와 gulp-sequelize 결합을 기반으로 새로운 CLI를 소개한다. CLI는 마이그레이션과 프로젝트 부트스트랩을 위해 제공을 한다. 마이그이션은 이미 존재하는 데이터 베이스를 교체할 수 있으며 그 반대로도 가능하다. 이러한 상태 변환은 마이그레이션 파일에 저장된다. 새로운 상태로 바꾸는 방법과 이전의 상태로 돌아가는 방법에 대해서 설명합니다.


# The CLI

CLI를 사용하려면 해당 패키지를 설치해야 합니다.

```
$ npm install --save sequelize-cli
```

모든 npm패키지와 마찬가지로, 여러분들은 전역으로 설치하기 위해 글로벌 플래그(`-g`)를 사용할 수 있다. 만약 여러분이 전역 플래그 없이 CLI를 설치했다면, `sequelize [command]` 대신에 `node_modules/.bin/sequelize [command]`을 사용하세요.

해당 CLI는 다음과 같은 명령어들을 제공한다.

```
$ sequelize db:migrate        # 마이그레이션을 실행합니다.
$ sequelize db:migrate:undo   # 마지막으로 실행 한 마이그레이션을 되돌리다.
$ sequelize help              # 도움말 텍스트를 보여줍니다.
$ sequelize init              # 프로젝트를 초기화시킵니다.
$ sequelize migration:create  # 새로운 마이그레이션 파일을 생성합니다.
$ sequelize version           # 버전을 출력합니다..
```

help명령어를 통해 사용가능한 명령어들에 대한 자세한 설명을 볼 수 있다.

```
$ sequelize help:init
$ sequelize help:db:migrate
$ sequelize help:db:migrate:undo
# etc
```

위와 같은 예제는 다음과 같이 출력 합니다.

```
Sequelize [CLI: v0.0.2, ORM: v1.7.5]

COMMANDS
    sequelize db:migrate:undo -- Revert the last migration run.

DESCRIPTION
    Revert the last migration run.

OPTIONS
    --env           The environment to run the command in. Default: development
    --options-path  The path to a JSON file with additional options. Default: none
    --coffee        Enables coffee script support. Default: false
    --config        The path to the config file. Default: config/config.json
```


# Skeleton

다음은 일반적인 마이그레이션 파일을 보여준다. 모든 마이그레이션은 프로젝트 맨 위에있는 마이그레이션이라는 폴더에 있어야 한다. 마이그레이션 바이너리는 마이그레이션 스켈레톤을 생성 할 수 있다. 자세한 내용은 위 내용을 확인 하세요.

```.js
module.exports = {
  up: function(queryInterface, Sequelize) {
    // 새로운 상태로 변환 하기위한 로닉
  },

  down: function(queryInterface, Sequelize) {
    // 변화를 바꾸는 로직
  }
}
```

`queryInterface` 객체는 데이터를 수정하기 위해 사용 할 수 있습니다. `Sequelize`객체는 `STRING` 또는 `INTEGET`와 같은 사용가능 한 데이터 타입을 저장한다. `up` 또는 `down` 함수는 `Promise`를 반환해야 한다. 다음은 몇 가지 코드입니다.

```.js
module.exports = {
  up: function(queryInterface, Sequelize) {
    return queryInterface.dropAllTables();
  }
}
```

queryInterface객체의 사용 가능 한 함수는 다음과 같이 있다.


# function

앞에 설명한 queryInterface를 사용하는 것은, 이미 소개된 대부분에 기능을 사용할 수 있습니다. 실제로 데이터베이스 스키마를 변경하도록 설계된 함수들이 더 있습니다.


## createTable(tablename, attributes, options)

이 함수는 새로운 테이블을 생성한다. 이것은 복잡한 속성 또는 간단한 속성을 가지고 정의할 수 있습니다. 여러분은 테이블의 인코딩과 옵션을 통해 테이블 엔진을 정의 할 수 있습니다.

```.js
queryInterface.createTable(
  'nameOfTheNewTable',
  {
    id: {
      type: Sequelize.INTEGER,
      primaryKey: true,
      autoIncrement: true
    },
    createdAt: {
      type: Sequelize.DATE
    },
    updatedAt: {
      type: Sequelize.DATE
    },
    attr1: Sequelize.STRING,
    attr2: Sequelize.INTEGER,
    attr3: {
      type: Sequelize.BOOLEAN,
      defaultValue: false,
      allowNull: false
    },
    //foreign key usage
    attr4: {
        type: Sequelize.INTEGER,
        references: {
            model: 'another_table_name',
            key: 'id'
        },
        onUpdate: 'cascade',
        onDelete: 'cascade'
    }
  },
  {
    engine: 'MYISAM', // default: 'InnoDB'
    charset: 'latin1' // default: null
  }
)
```


## dropTable(tableName, options)

이 함수는 이미 존재하는 테이블을 제거 할 수 있다.

```.js
queryInterface.dropTable('nameOfTheExistingTable')
```


## dropAllTables(options)

이 함수는 데이터베이스에 존재하는 모든 테이블을 제거 할 수 있다.

```.js
queryInterface.dropAllTables()
```


## renameTable(before, after, options)

이 함수는 이미 존재하는 테이블의 이름을 바꿀 수 있다.

```.js
queryInterface.renameTable('Person', 'User')
```


## showAllTables(options)

이 함수는 데이터베이스에서 존재하는 테이블의 이름을 반환해 준다.

```.js
queryInterface.showAllTables().then(function(tableNames) {})
```


## describeTable(tableName, options)

이 함수는 테이블에서 모든 속성에 대한 정보를 포함하여 해시배열로 반환한다.

```.js
queryInterface.describeTable('Person').then(function(attributes) {
  /*
    속성은 다음과 같습니다.

    {
      name: {
        type:         'VARCHAR(255)', // 이것은 pg를 위한 'CHARACTER VARYING'일 것이다.  
        allowNull:    true,
        defaultValue: null
      },
      isBetaMember: {
        type:         'TINYINT(1)', // 이것은 pg를 위한 'BOOLEAN'일 것이다.  
        allowNull:    false,
        defaultValue: false
      }
    }
  */
})
```


## addColumn(tableName, attributeName, dataTypeOrOptions, options)

이 함수는 이미 존재하는 테이블에 컬럼을 더해준다. 해당 데이터 타입은 간단하거나 복잡 할 수 있다.

```.js
queryInterface.addColumn(
  'nameOfAnExistingTable',
  'nameOfTheNewAttribute',
  Sequelize.STRING
)

// or

queryInterface.addColumn(
  'nameOfAnExistingTable',
  'nameOfTheNewAttribute',
  {
    type: Sequelize.STRING,
    allowNull: false
  }
)
```

## removeColumn(tableName, attributeName, options)

이 함수는 이미 존재하는 테이블의 특정 컬럼을 제거 할 수 있다.

```.js
queryInterface.removeColumn('Person', 'signature')
```


## changeColumn(tableName, attributeName, dataTypeOrOptions, options)

이 함수는 속성의 메타 데이터를 바꿀 수 있다. 그것은 기본 값을 바꾸거나, null또는 해당 데이터 타입으로 허용할 수있다. 여러분들은 새로운 데이터 타입이 완벽하게 설명하고 있는지 확인을 하세요

```.js
queryInterface.changeColumn(
  'nameOfAnExistingTable',
  'nameOfAnExistingAttribute',
  {
    type: Sequelize.FLOAT,
    allowNull: false,
    defaultValue: 0.0
  }
)
```


## renameColumn(tableName, attrNameBefore, attrNameAfter, options)

이 함수는 속성의 이름을 바꿀 수 있다.

```.js
queryInterface.renameColumn('Person', 'signature', 'sig')
```


## addIndex(tableName, attributes, options)

이 함수는 테이블에서 특정 속성을 인덱스로 생성 할 수 있다. 해당 인덱스 이름은 아래에 보이는 옵션처럼 전달 되지 않는다면 자동적으로 생성이 될 것이다.

```.js
// This example will create the index person_firstname_lastname
queryInterface.addIndex('Person', ['firstname', 'lastname'])

// This example will create a unique index with the name SuperDuperIndex using the optional 'options' field.
// Possible options:
// - indicesType: UNIQUE|FULLTEXT|SPATIAL
// - indexName: The name of the index. Default is __
// - parser: For FULLTEXT columns set your parser
// - indexType: Set a type for the index, e.g. BTREE. See the documentation of the used dialect
// - logging: A function that receives the sql query, e.g. console.log
queryInterface.addIndex(
  'Person',
  ['firstname', 'lastname'],
  {
    indexName: 'SuperDuperIndex',
    indicesType: 'UNIQUE'
  }
)
```


## removeIndex(tableName, indexNameOrAttributes, options)

이 함수는 이미 존재하는 테이블의 인덱스를 제거 할 수 있다.

```.js
queryInterface.removeIndex('Person', 'SuperDuperIndex')

// or

queryInterface.removeIndex('Person', ['firstname', 'lastname'])
```


# Programmatic use

sequelize는 프로그래밍 방식으로 실행 후 처리, 마이그레이션 작업을 로깅하기 위해 [sister library](https://github.com/sequelize/umzug)을 가지고 있습니다.
