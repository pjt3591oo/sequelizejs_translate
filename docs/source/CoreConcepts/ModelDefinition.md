# 모델정의

모델과 테이블 맵핑을 정의하기 위해 `define` 메서드를 사용합니다. 각 컬럼은 데이터 타입을 가지고 있으며 [데이터 타입](https://pjt3591oo.github.io/sequelizejs_translate/build/html/CoreConcepts/DateTypes.html)에서 확인가능합니다.

```js
class Project extends Model {}
Project.init({
  title: Sequelize.STRING,
  description: Sequelize.TEXT
}, { sequelize, modelName: 'project' });

class Task extends Model {}
Task.init({
  title: Sequelize.STRING,
  description: Sequelize.TEXT,
  deadline: Sequelize.DATE
}, { sequelize, modelName: 'task' })
```

데이터 타입 이외에도 설정할 수 있는 옵션이 많습니다.

```js
class Foo extends Model {}
Foo.init({
 // instantiating will automatically set the flag to true if not set
 flag: { type: Sequelize.BOOLEAN, allowNull: false, defaultValue: true },

 // default values for dates => current time
 myDate: { type: Sequelize.DATE, defaultValue: Sequelize.NOW },

 // setting allowNull to false will add NOT NULL to the column, which means an error will be
 // thrown from the DB when the query is executed if the column is null. If you want to check that a value
 // is not null before querying the DB, look at the validations section below.
 title: { type: Sequelize.STRING, allowNull: false },

 // Creating two objects with the same value will throw an error. The unique property can be either a
 // boolean, or a string. If you provide the same string for multiple columns, they will form a
 // composite unique key.
 uniqueOne: { type: Sequelize.STRING,  unique: 'compositeIndex' },
 uniqueTwo: { type: Sequelize.INTEGER, unique: 'compositeIndex' },

 // The unique property is simply a shorthand to create a unique constraint.
 someUnique: { type: Sequelize.STRING, unique: true },

 // It's exactly the same as creating the index in the model's options.
 { someUnique: { type: Sequelize.STRING } },
 { indexes: [ { unique: true, fields: [ 'someUnique' ] } ] },

 // Go on reading for further information about primary keys
 identifier: { type: Sequelize.STRING, primaryKey: true },

 // autoIncrement can be used to create auto_incrementing integer columns
 incrementMe: { type: Sequelize.INTEGER, autoIncrement: true },

 // You can specify a custom column name via the 'field' attribute:
 fieldWithUnderscores: { type: Sequelize.STRING, field: 'field_with_underscores' },

 // It is possible to create foreign keys:
 bar_id: {
   type: Sequelize.INTEGER,

   references: {
     // This is a reference to another model
     model: Bar,

     // This is the column name of the referenced model
     key: 'id',

     // This declares when to check the foreign key constraint. PostgreSQL only.
     deferrable: Sequelize.Deferrable.INITIALLY_IMMEDIATE
   }
 },

 // It is possible to add comments on columns for MySQL, PostgreSQL and MSSQL only
 commentMe: {
   type: Sequelize.INTEGER,

   comment: 'This is a column name that has a comment'
 }
}, {
  sequelize,
  modelName: 'foo'
});
```

주석 옵션은 테이블에서도 사용할 수 있습니다. [모델 설정](https://sequelize.org/master/manual/models-definition.html#configuration)을 참조하세요.

## 타임스탬프

기본적으로 Sequelize는 `createdAt` 및 `updatedAt` 속성을 모델에 추가하여, 데이터의 추가/갱신 시점을 알 수 있습니다.

Sequelize 마이그레이션을 사용할 경우 `createdAt` 및 `updatedAt` 필드를 마이그레이션 정의에 추가해야 합니다.

```js
module.exports = {
  up(queryInterface, Sequelize) {
    return queryInterface.createTable('my-table', {
      id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true,
      },

      // Timestamps
      createdAt: Sequelize.DATE,
      updatedAt: Sequelize.DATE,
    })
  },
  down(queryInterface, Sequelize) {
    return queryInterface.dropTable('my-table');
  },
}
```

만약 타임스탬프를 사용하지 않기를 원한다면, 타임스탬프만 원하거나 열 이름이 다른 기존 데이터베이스를 사용하는 경우 [설정방법](https://sequelize.org/master/manual/models-definition.html#configuration)으로 이동하여 해당 방법을 확인할 수 있습니다.

## 지연

외래키 열을 지정하면 PostgreSQL에서 연기 가능한 타입을 선언할 수 있습니다. 다음과 같은 옵션을 사용할 수 있습니다.

```js
// Defer all foreign key constraint check to the end of a transaction
Sequelize.Deferrable.INITIALLY_DEFERRED

// Immediately check the foreign key constraints
Sequelize.Deferrable.INITIALLY_IMMEDIATE

// Don't defer the checks at all
Sequelize.Deferrable.NOT
```

마지막 옵션은 PostgreSQL에서 기본값 입니다. 그리고 트랜잭션에서 규칙을 동적으로 바꿀 수 없습니다. 더 자세한 정보는 [트랜잭션 섹션](https://sequelize.org/master/manual/transactions.html#options)에서 확인가능 합니다.

## getter & setter

이것은 모델안에 '객체속성' getters와 setter를 정의할 수 있습니다. 이것들은 데이터베이스 필드에 매핑되는 '보호' 속성과 '의사(인위적인)' 속성 정의에 모두 사용할 수 있습니다.

getters와 setters는 2가지 방법으로 정의할 수 있습니다

  * 단일 속성 정의의 일부
  * 한 모델 옵션의 일부

> N.B: getter 또는 setter가 두 위치에 모두 정의되 있으면 속성 정의에 있는 함수가 항상 우선합니다.

### 단일 속성의 일부 정의

```js
class Employee extends Model {}
Employee.init({
  name: {
    type: Sequelize.STRING,
    allowNull: false,
    get() {
      const title = this.getDataValue('title');
      // 'this' allows you to access attributes of the instance
      return this.getDataValue('name') + ' (' + title + ')';
    },
  },
  title: {
    type: Sequelize.STRING,
    allowNull: false,
    set(val) {
      this.setDataValue('title', val.toUpperCase());
    }
  }
}, { sequelize, modelName: 'employee' });

Employee
  .create({ name: 'John Doe', title: 'senior engineer' })
  .then(employee => {
    console.log(employee.get('name')); // John Doe (SENIOR ENGINEER)
    console.log(employee.get('title')); // SENIOR ENGINEER
  })
```

### 모델 옵션 일부 정의

아래는 모델 옵션에서 getter와 setter를 정의한 예시입니다.

`fullName` getter는 모델에서 인위적으로 속성을 정의하는 방법의 예시입니다. - 실제로 데이터 베이스의 부분이 아닌 속성입니다. 사실, `fullName`과 같이 인위적으로 만든 속성은 두 가지 방법으로 정의할 수 있습니다. 모델 getther를 사용하는 방법과 [가상 데이터타입]()과 함께 컬럼을 사용하는 방법입니다. 가상 데이터 타입은 유효성 검사가 가능하지만 가상 속성에 대한 getther는 유효성 검사가 불가능 합니다.

`fullName` getter 함수에서 `this.firstname`과 `this.lastname` 참조는 getter 함수에 대한 호출을 트리거 합니다. 만약 이를 원치않는다면 `getDataValue()` 메서드를 사용하여 로우값에 접근할 수 있습니다.

```js
class Foo extends Model {
  get fullName() {
    return this.firstname + ' ' + this.lastname;
  }

  set fullName(value) {
    const names = value.split(' ');
    this.setDataValue('firstname', names.slice(0, -1).join(' '));
    this.setDataValue('lastname', names.slice(-1).join(' '));
  }
}
Foo.init({
  firstname: Sequelize.STRING,
  lastname: Sequelize.STRING
}, {
  sequelize,
  modelName: 'foo'
});

// Or with `sequelize.define`
sequelize.define('Foo', {
  firstname: Sequelize.STRING,
  lastname: Sequelize.STRING
}, {
  getterMethods: {
    fullName() {
      return this.firstname + ' ' + this.lastname;
    }
  },

  setterMethods: {
    fullName(value) {
      const names = value.split(' ');

      this.setDataValue('firstname', names.slice(0, -1).join(' '));
      this.setDataValue('lastname', names.slice(-1).join(' '));
    }
  }
});
```

### getter와 setter 정의된 내부에서 사용을 위한 헬퍼함수

* 기본 속성값 가져오기 - 항상 `this.getDataValue()` 사용

```js
/* a getter for 'title' property */
get() {
  return this.getDataValue('title')
}
```

* 기본 속성값 설정 - 항상 `this.setDataValue()` 사용

```js
/* a setter for 'title' property */
set(title) {
  this.setDataValue('title', title.toString().toLowerCase());
}
```

**N.B**: `setDataValue()`와 `getDataValue()`를 사용하는 것은 중요합니다(기본 '데이터 값' 속성에 직접 접근하는 것과 다름). - 그렇게하면 사용자 정의 getter와 setter가 기본 모델 구현의 변경으로부터 보호합니다.

## 검증

모델 검증을 통해 모델의 각 속성에 대한 형식/컨텐츠/상속 검증을 지정할 수 있습니다.

`create`, `update` 그리고 `save`시 검증은 자동으로 실행합니다. `validate()`를 호출하여 인스턴스를 수동으로 호출할 수 있습니다.

### 속성별 검증

아래와 같이 사용자 정의 검증을 정의하거나, [validator.js](https://github.com/validatorjs/validator.js)로 구현된 여러 내장 유효성 검사를 사용할 수 있습니다.

```js
class ValidateMe extends Model {}
ValidateMe.init({
  bar: {
    type: Sequelize.STRING,
    validate: {
      is: ["^[a-z]+$",'i'],     // will only allow letters
      is: /^[a-z]+$/i,          // same as the previous example using real RegExp
      not: ["[a-z]",'i'],       // will not allow letters
      isEmail: true,            // checks for email format (foo@bar.com)
      isUrl: true,              // checks for url format (http://foo.com)
      isIP: true,               // checks for IPv4 (129.89.23.1) or IPv6 format
      isIPv4: true,             // checks for IPv4 (129.89.23.1)
      isIPv6: true,             // checks for IPv6 format
      isAlpha: true,            // will only allow letters
      isAlphanumeric: true,     // will only allow alphanumeric characters, so "_abc" will fail
      isNumeric: true,          // will only allow numbers
      isInt: true,              // checks for valid integers
      isFloat: true,            // checks for valid floating point numbers
      isDecimal: true,          // checks for any numbers
      isLowercase: true,        // checks for lowercase
      isUppercase: true,        // checks for uppercase
      notNull: true,            // won't allow null
      isNull: true,             // only allows null
      notEmpty: true,           // don't allow empty strings
      equals: 'specific value', // only allow a specific value
      contains: 'foo',          // force specific substrings
      notIn: [['foo', 'bar']],  // check the value is not one of these
      isIn: [['foo', 'bar']],   // check the value is one of these
      notContains: 'bar',       // don't allow specific substrings
      len: [2,10],              // only allow values with length between 2 and 10
      isUUID: 4,                // only allow uuids
      isDate: true,             // only allow date strings
      isAfter: "2011-11-05",    // only allow date strings after a specific date
      isBefore: "2011-11-05",   // only allow date strings before a specific date
      max: 23,                  // only allow values <= 23
      min: 23,                  // only allow values >= 23
      isCreditCard: true,       // check for valid credit card numbers

      // Examples of custom validators:
      isEven(value) {
        if (parseInt(value) % 2 !== 0) {
          throw new Error('Only even values are allowed!');
        }
      }
      isGreaterThanOtherField(value) {
        if (parseInt(value) <= parseInt(this.otherField)) {
          throw new Error('Bar must be greater than otherField.');
        }
      }
    }
  }
}, { sequelize });
```

내장된 유효성 검사함수에 여러 인자를 전달해야 하는경우, 배열 형태로 전달합니다. 그러나 하나의 어레이 형태(예: `isIn`에 허용되는 문자열 배열)로 전달되면 하나의 배열대신 여러 문자열 인자로 해석합니다. 이 문제를 해결하려면 위에 표시된대로 `[['one', 'two']]`와 같은 단일 길이의 인수 배열을 전달합니다.

사용자 정의 에러를 사용하기 위해 [validator.js](https://github.com/validatorjs/validator.js)에 의해 제공되는 것 대신, 일반값 또는 배열 인자 대신 객체를 사용합니다. 예를들어 인자가 필요없는 검증은 다음과 같이 메시지를 줄 수 있습니다.

```js
isInt: {
  msg: "Must be an integer number of pennies"
}
```

또는 인자가 필요하다면 `args` 속성을 이용하여 전달할 수 있습니다.

```js
isIn: {
  args: [['en', 'zh']],
  msg: "Must be English or Chinese"
}
```

사용자 정의 에러 메시지 함수를 사용할 때 오류 메시지는 throw된 `Error` 객체에 포함 된 모든 메시지입니다.

내장된 검증 메서드를 더 자세히 알아보려면 [validators.js 프로젝트](https://github.com/validatorjs/validator.js)를 확인하세요.

**힌트**: 로깅을 위해 사용자 정의 함수를 만들 수 있습니다. 그냥 함수를 전달합니다. 첫 번째 인자로 로그정보 문자열을 전달합니다.

### 속성별 유효선 검사 및 allowNull

모델의 특정 필드가 null을 허용하지 않도록 설정되고(`allowNull: false`) 해당 값이 null로 설정된 경우 모든 유효성 검사기가 건너뛰고 `ValidationError`를 발생합니다.

반면에, null을 허용하고(`allow: true`) 해당 값이 `null`이라면 내장 유효성 검사는 사용자 정의 유효성 검사가 동작되는동안 건너뜁니다.

예를들어 길이가 5 ~ 10자 사이인지 확인하지만 `null`도 허용하는 문자열 필드를 가질 수 있습니다 (값이 `null`인 경우 길이 검사는 자동으로 건너뜁니다):

```js
class User extends Model {}
User.init({
  username: {
    type: Sequelize.STRING,
    allowNull: true,
    validate: {
      len: [5, 10]
    }
  }
}, { sequelize });
```

건너 뛰지않고 사용자 정의 유효성 검사를 사용하여 null 값을 허용하는지 조건 검사를 할 수 있습니다.

```js
class User extends Model {}
User.init({
  age: Sequelize.INTEGER,
  name: {
    type: Sequelize.STRING,
    allowNull: true,
    validate: {
      customValidator(value) {
        if (value === null && this.age !== 10) {
          throw new Error("name can't be null unless age is 10");
        }
      })
    }
  }
}, { sequelize });
```

`notNull` 유효성 검사 셋팅을 통해 `allowNull` 에러 메시지를 사용자 정의할 수 있습니다.

```js
class User extends Model {}
User.init({
  name: {
    type: Sequelize.STRING,
    allowNull: false,
    validate: {
      notNull: {
        msg: 'Please enter your name'
      }
    }
  }
}, { sequelize });
```

### 모델 전체 검증

필드별 유효성 검사 후 모델을 확인하기 위해 유효성 검사를 정의할 수 있습니다. 이것을 사용하면 위도 및 경도가 설정되어 있고 둘 중 하나만 설정되어 있으면 실패 할 수 있습니다.

모델 유효성 검사 메서드들은 모델 객체의 컨텍스트와 함께 호출되며, 오류가 발생하면 실패한 것으로 간주하고 그렇지 않으면 전달합니다. 이것은 사용자 정의된 필드별 유효성 검사와 동일합니다.

수집된 에러 메시지는 필드 유효성 검사 오류와 유효성 검사 결과 객체에 있으며, 유효성 검사 옵션 객체에서 유효성이 검사되지 않은 방법의 키 이름이 지정된 키와 함께 나타납니다. 한 번에 각 모델 유효성 검사 방법에 대해 하나의 오류 메시지 만있을 수 있지만 필드 오류와의 일관성을 최대화하기 위해 배열에서 단일 문자열 오류로 표시됩니다.

예시:

```js
class Pub extends Model {}
Pub.init({
  name: { type: Sequelize.STRING },
  address: { type: Sequelize.STRING },
  latitude: {
    type: Sequelize.INTEGER,
    allowNull: true,
    defaultValue: null,
    validate: { min: -90, max: 90 }
  },
  longitude: {
    type: Sequelize.INTEGER,
    allowNull: true,
    defaultValue: null,
    validate: { min: -180, max: 180 }
  },
}, {
  validate: {
    bothCoordsOrNone() {
      if ((this.latitude === null) !== (this.longitude === null)) {
        throw new Error('Require either both latitude and longitude or neither')
      }
    }
  },
  sequelize,
})
```

이 예시의 경우 위도와 경도를 제공하지만 둘 다가 아닌경우 객체 유효성 검사는 실패합니다. 만약, 위도의 값은 범위를 넘거나 경도가 없다면 `raging_bullock_arms.validate()`가 반환될 것 입니다.

```js
{
  'latitude': ['Invalid number: latitude'],
  'bothCoordsOrNone': ['Require either both latitude and longitude or neither']
}
```

이러한 유효성 검사는 하나의 속성에 정의된 사용자 지정 유효성 검사로 수행할 수 있지만 모델은 광범위한 검증 방법이 간결합니다.

## 설정

Sequelize가 열 이름을 처리하는 방법이 영향을 줄 수 있습니다.

```js
class Bar extends Model {}
Bar.init({ /* bla */ }, {
  // The name of the model. The model will be stored in `sequelize.models` under this name.
  // This defaults to class name i.e. Bar in this case. This will control name of auto-generated
  // foreignKey and association naming
  modelName: 'bar',

  // don't add the timestamp attributes (updatedAt, createdAt)
  timestamps: false,

  // don't delete database entries but set the newly added attribute deletedAt
  // to the current date (when deletion was done). paranoid will only work if
  // timestamps are enabled
  paranoid: true,

  // Will automatically set field option for all attributes to snake cased name.
  // Does not override attribute with field option already defined
  underscored: true,

  // disable the modification of table names; By default, sequelize will automatically
  // transform all passed model names (first parameter of define) into plural.
  // if you don't want that, set the following
  freezeTableName: true,

  // define the table's name
  tableName: 'my_very_custom_table_name',

  // Enable optimistic locking.  When enabled, sequelize will add a version count attribute
  // to the model and throw an OptimisticLockingError error when stale instances are saved.
  // Set to true or a string with the attribute name you want to use to enable.
  version: true,

  // Sequelize instance
  sequelize,
})
```

sequelize가 타임스탬프를 서치하고 싶지만 그 중 일부만 원하거나 타임스탬프를 다른 것으로 호출하기 위해 각 열을 개별적으로 재정의할 수 있습니다.

```js
class Foo extends Model {}
Foo.init({ /* bla */ }, {
  // don't forget to enable timestamps!
  timestamps: true,

  // I don't want createdAt
  createdAt: false,

  // I want updatedAt to actually be called updateTimestamp
  updatedAt: 'updateTimestamp',

  // And deletedAt to be called destroyTime (remember to enable paranoid for this to work)
  deletedAt: 'destroyTime',
  paranoid: true,

  sequelize,
})
```

데이터베이스 엔진도 바꿀 수 있습니다. InnoDB는 기본값 입니다.

```js
class Person extends Model {}
Person.init({ /* attributes */ }, {
  engine: 'MYISAM',
  sequelize
})

// or globally
const sequelize = new Sequelize(db, user, pw, {
  define: { engine: 'MYISAM' }
})
```

마지막으로 MySQL 및 PG에서 테이블에 주석을 지정할 수 있습니다.

```js
class Person extends Model {}
Person.init({ /* attributes */ }, {
  comment: "I'm a table comment!",
  sequelize
})
```

## 임포트

`import` 메서드를 사용하여 모델 정의를 단일 파일에 저장할 수 있습니다. 반환된 객체는 가져온 파일 함수에 정의된 것과 정확하게 같습니다. `V1:5.0`에서 임포트는 캐시가 되므로 두 번이상 자주 호출할 때 문제가 발생하지 않습니다.

```js
// in your server file - e.g. app.js
const Project = sequelize.import(__dirname + "/path/to/models/project")

// The model definition is done in /path/to/models/project.js
// As you might notice, the DataTypes are the very same as explained above
module.exports = (sequelize, DataTypes) => {
  class Project extends sequelize.Model { }
  Project.init({
    name: DataTypes.STRING,
    description: DataTypes.TEXT
  }, { sequelize });
  return Project;
}
```

`import` 메서드는 콜백을 인자로 받을 수 있습니다.

```js
sequelize.import('project', (sequelize, DataTypes) => {
  class Project extends sequelize.Model {}
  Project.init({
    name: DataTypes.STRING,
    description: DataTypes.TEXT
  }, { sequelize })
  return Project;
})
```

이 추가 기능은 다음과 같은 경우에 유용합니다. 예를들어 `/path/to/models/project`가 옳바른 경로처럼 보이지만 `Error: cannot find module` 에러가 발생할 때 유용합니다.

```js
Error: Cannot find module '/home/you/meteorApp/.meteor/local/build/programs/server/app/path/to/models/project.js'
```

이것은 Meteor의 require 버전 전달에의해 해결할 수 있습니다. 따라서 이것이 실패하는 동안...

```js
const AuthorModel = db.import('./path/to/models/project');
```

... 이것은 성공해야 합니다....

```js
const AuthorModel = db.import('project', require('./path/to/models/project'));
```

## Optimistic Locking

Sequelize는 모델 인스턴스 버전 수를 통해 `Optimistic Lock`을 기본적으로 지원합니다. `Optimistic locking`은 기본적으로 비활성화고 특정 모델 정의 또는 저역 모델 구성에서 버전 속성을 true로 설정하여 활성화 할 수 있습니다. 더 자세한 정보는 [모델설정](https://sequelize.org/master/manual/models-definition.html#configuration)을 보세요.

`Optimistic Locking`은 편집을 위해 모델 레코드를 동시에 접근할 수 있으며 충돌로 인해 데이터를 덮어 쓰지 않습니다. 다른 프로세스가 읽은 이후 레코드를 변경했는지 확인하여 충돌을 감지하면 `OptimisticLockError`를 발생합니다.

## 데이터베이스 동기화

새 프로젝트를 시작할 때 데이터베이스 구조가 없고 Sequelize를 사용할 필요가 없습니다. 무델 구조를 지정하고 라이브러리에서 나머지 작업을 수행하세요. 현재 테이블 작성 및 삭제를 지원합니다.

```js
// Create the tables:
Project.sync()
Task.sync()

// Force the creation!
Project.sync({force: true}) // this will drop the table first and re-create it afterwards

// drop the tables:
Project.drop()
Task.drop()

// event handling:
Project.[sync|drop]().then(() => {
  // ok ... everything is nice!
}).catch(error => {
  // oooh, did you enter wrong database credentials?
})
```

모든 테이블을 동기화 및 삭제하면 많은 행을 쓸 수 있으므로 Sequelize가 작업을 수행하도록 할 수 있습니다.

```js
// Sync all models that aren't already in the database
sequelize.sync()

// Force sync all models
sequelize.sync({force: true})

// Drop all tables
sequelize.drop()

// emit handling:
sequelize.[sync|drop]().then(() => {
  // woot woot
}).catch(error => {
  // whooops
})
```

`.sync({force: true})`는 파괴적인 작업이므로 일치 옵션을 추가하여 안전한 검사로 사용할 수 있습니다. match 옵션은 동기화하기 전에 데이터베이스 이름과 정규식을 일치 시키도록 sequelize에 지시합니다. `force:true`는 테스트에 사용되지만 라이브 코드는 사용되지 않는 경우에 대한 안전검사 합니다.

```js
// This will run .sync() only if database name ends with '_test'
sequelize.sync({ force: true, match: /_test$/ });
```

## 모델확장

Sequelize 모델은 ES6 클래스입니다. 사용자 정의 인스턴스 또는 클래스 레벨 메서드를 쉽게 추가할 수 있습니다.

```js
class User extends Model {
  // Adding a class level method
  static classLevelMethod() {
    return 'foo';
  }

  // Adding an instance level method
  instanceLevelMethod() {
    return 'bar';
  }
}
User.init({ firstname: Sequelize.STRING }, { sequelize });
```

물론 인스턴스 데이터에 접근하여 가상 getter를 생성할 수 있습니다.

```js
class User extends Model {
  getFullname() {
    return [this.firstname, this.lastname].join(' ');
  }
}
User.init({ firstname: Sequelize.STRING, lastname: Sequelize.STRING }, { sequelize });

// Example:
User.build({ firstname: 'foo', lastname: 'bar' }).getFullname() // 'foo bar'
```

### indexes

Sequelize는 `Model.sync()` 또는 `sequelize.sync`에서 생성될 모델 정의에서 인덱스 추가를 제공합니다.

```js
class User extends Model {}
User.init({}, {
  indexes: [
    // Create a unique index on email
    {
      unique: true,
      fields: ['email']
    },

    // Creates a gin index on data with the jsonb_path_ops operator
    {
      fields: ['data'],
      using: 'gin',
      operator: 'jsonb_path_ops'
    },

    // By default index name will be [table]_[fields]
    // Creates a multi column partial index
    {
      name: 'public_by_author',
      fields: ['author', 'status'],
      where: {
        status: 'public'
      }
    },

    // A BTREE index with an ordered field
    {
      name: 'title_index',
      using: 'BTREE',
      fields: ['author', {attribute: 'title', collate: 'en_US', order: 'DESC', length: 5}]
    }
  ],
  sequelize
});
```