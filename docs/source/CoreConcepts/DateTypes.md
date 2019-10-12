## 데이터 타입

아래는 sequelize에서 제공하는 데이터타입입니다. 전체 및 업데이트 된 목록은 [DataTypes](https://sequelize.org/master/variable/index.html#static-variable-DataTypes)을 참조하세요.

```js
Sequelize.STRING                      // VARCHAR(255)
Sequelize.STRING(1234)                // VARCHAR(1234)
Sequelize.STRING.BINARY               // VARCHAR BINARY
Sequelize.TEXT                        // TEXT
Sequelize.TEXT('tiny')                // TINYTEXT
Sequelize.CITEXT                      // CITEXT      PostgreSQL and SQLite only.

Sequelize.INTEGER                     // INTEGER
Sequelize.BIGINT                      // BIGINT
Sequelize.BIGINT(11)                  // BIGINT(11)

Sequelize.FLOAT                       // FLOAT
Sequelize.FLOAT(11)                   // FLOAT(11)
Sequelize.FLOAT(11, 10)               // FLOAT(11,10)

Sequelize.REAL                        // REAL        PostgreSQL only.
Sequelize.REAL(11)                    // REAL(11)    PostgreSQL only.
Sequelize.REAL(11, 12)                // REAL(11,12) PostgreSQL only.

Sequelize.DOUBLE                      // DOUBLE
Sequelize.DOUBLE(11)                  // DOUBLE(11)
Sequelize.DOUBLE(11, 10)              // DOUBLE(11,10)

Sequelize.DECIMAL                     // DECIMAL
Sequelize.DECIMAL(10, 2)              // DECIMAL(10,2)

Sequelize.DATE                        // DATETIME for mysql / sqlite, TIMESTAMP WITH TIME ZONE for postgres
Sequelize.DATE(6)                     // DATETIME(6) for mysql 5.6.4+. Fractional seconds support with up to 6 digits of precision
Sequelize.DATEONLY                    // DATE without time.
Sequelize.BOOLEAN                     // TINYINT(1)

Sequelize.ENUM('value 1', 'value 2')  // An ENUM with allowed values 'value 1' and 'value 2'
Sequelize.ARRAY(Sequelize.TEXT)       // Defines an array. PostgreSQL only.
Sequelize.ARRAY(Sequelize.ENUM)       // Defines an array of ENUM. PostgreSQL only.

Sequelize.JSON                        // JSON column. PostgreSQL, SQLite and MySQL only.
Sequelize.JSONB                       // JSONB column. PostgreSQL only.

Sequelize.BLOB                        // BLOB (bytea for PostgreSQL)
Sequelize.BLOB('tiny')                // TINYBLOB (bytea for PostgreSQL. Other options are medium and long)

Sequelize.UUID                        // UUID datatype for PostgreSQL and SQLite, CHAR(36) BINARY for MySQL (use defaultValue: Sequelize.UUIDV1 or Sequelize.UUIDV4 to make sequelize generate the ids automatically)

Sequelize.CIDR                        // CIDR datatype for PostgreSQL
Sequelize.INET                        // INET datatype for PostgreSQL
Sequelize.MACADDR                     // MACADDR datatype for PostgreSQL

Sequelize.RANGE(Sequelize.INTEGER)    // Defines int4range range. PostgreSQL only.
Sequelize.RANGE(Sequelize.BIGINT)     // Defined int8range range. PostgreSQL only.
Sequelize.RANGE(Sequelize.DATE)       // Defines tstzrange range. PostgreSQL only.
Sequelize.RANGE(Sequelize.DATEONLY)   // Defines daterange range. PostgreSQL only.
Sequelize.RANGE(Sequelize.DECIMAL)    // Defines numrange range. PostgreSQL only.

Sequelize.ARRAY(Sequelize.RANGE(Sequelize.DATE)) // Defines array of tstzrange ranges. PostgreSQL only.

Sequelize.GEOMETRY                    // Spatial column.  PostgreSQL (with PostGIS) or MySQL only.
Sequelize.GEOMETRY('POINT')           // Spatial column with geometry type. PostgreSQL (with PostGIS) or MySQL only.
Sequelize.GEOMETRY('POINT', 4326)     // Spatial column with geometry type and SRID.  PostgreSQL (with PostGIS) or MySQL only.
```

**BLOB** 타입은 문자열 또는 버퍼로 생성된 데이터의 추가를 허용합니다. BLOB 컬럼을 가진 모델에서 find 또는 findAl을 할 때 데이터는 항상 버퍼형태로 반환합니다.

시간대없이 PostgreSQL TIMESTAMP를 사용하여 다른 시간대로 구문 분석해야하는 경우 pg 라이브러리의 자체 구문 분석기를 사용하십시오.

```js
require('pg').types.setTypeParser(1114, stringValue => {
  return new Date(stringValue + '+0000');
  // e.g., UTC offset. Use any offset that you would like.
});
```

위에서 언급한 코드 이외에, integer, bigint, float 그리고 double은 부호없는 속성과 zerofill 속성을 지원하며 어떤 순서로든 조합할 수 잇습니다. PostgreSQL는 지원하지 않습니다.

```js
Sequelize.INTEGER.UNSIGNED              // INTEGER UNSIGNED
Sequelize.INTEGER(11).UNSIGNED          // INTEGER(11) UNSIGNED
Sequelize.INTEGER(11).ZEROFILL          // INTEGER(11) ZEROFILL
Sequelize.INTEGER(11).ZEROFILL.UNSIGNED // INTEGER(11) UNSIGNED ZEROFILL
Sequelize.INTEGER(11).UNSIGNED.ZEROFILL // INTEGER(11) UNSIGNED ZEROFILL
```

*오직 integer 예시입니다. 그러나 bigint와 float에서도 사용법은 동일합니다. *

객체표기 사용방법:

```js
// for enums:
class MyModel extends Model {}
MyModel.init({
  states: {
    type: Sequelize.ENUM,
    values: ['active', 'pending', 'deleted']
  }
}, { sequelize })
```

### Array(ENUM)

이것은 오직 PostgreSQL에서만 지원합니다.

Array(Enum) 타입은 특정한 취급방법을 요구한다. Sequelize는 데이터베이스와 통신할 때마다 ENUM 이름으로 배열값의 유형을 변경해야 합니다.

이 unum 이름은 `enum_<table_name>` 패턴을 따라야 합니다. `sync`를 사용한다면, 자동으로 옳바른 이름을 생성합니다.

### range 타입들

range 타입은 바인딩 포함/제외에 대한 추가 정보가 있으므로 튜플을 사용하여 자바스크립트로 표시하는 것은 매우 간단하지 않습니다.

범위를 값으로 제공할 땐 다음과 같이 사용할 수 있습니다.

```js
// defaults to '["2016-01-01 00:00:00+00:00", "2016-02-01 00:00:00+00:00")'
// inclusive lower bound, exclusive upper bound
Timeline.create({ range: [new Date(Date.UTC(2016, 0, 1)), new Date(Date.UTC(2016, 1, 1))] });

// control inclusion
const range = [
  { value: new Date(Date.UTC(2016, 0, 1)), inclusive: false },
  { value: new Date(Date.UTC(2016, 1, 1)), inclusive: true },
];
// '("2016-01-01 00:00:00+00:00", "2016-02-01 00:00:00+00:00"]'

// composite form
const range = [
  { value: new Date(Date.UTC(2016, 0, 1)), inclusive: false },
  new Date(Date.UTC(2016, 1, 1)),
];
// '("2016-01-01 00:00:00+00:00", "2016-02-01 00:00:00+00:00")'

Timeline.create({ range });
```

그러나, 범위의 값을 다시 가져올 떄마다 다음을 받습니다.

```js
// stored value: ("2016-01-01 00:00:00+00:00", "2016-02-01 00:00:00+00:00"]
range // [{ value: Date, inclusive: false }, { value: Date, inclusive: true }]
```

range 타입으로 인스턴트를 업데이트한 후 range 타입 사용 또는 `returning: true` 옵션을 사용하여 reload를 호출해야 합니다.

#### 특별한 케이스

```js
// empty range:
Timeline.create({ range: [] }); // range = 'empty'

// Unbounded range:
Timeline.create({ range: [null, null] }); // range = '[,)'
// range = '[,"2016-01-01 00:00:00+00:00")'
Timeline.create({ range: [null, new Date(Date.UTC(2016, 0, 1))] });

// Infinite range:
// range = '[-infinity,"2016-01-01 00:00:00+00:00")'
Timeline.create({ range: [-Infinity, new Date(Date.UTC(2016, 0, 1))] });
```

## 데이터 타입 확장


구현하려는 유형이 이미 [DataTypes에](https://sequelize.org/master/manual/data-types.html) 포함되어있을 가능성이 큽니다. 새 데이터 유형이 포함되지 않은 경우이 매뉴얼은 직접 작성하는 방법을 보여줍니다.

Sequelize는 데이터베이스에서 새로운 데이터 타입을 생성하지 않습니다. 이 튜토리얼은 새로운 데이터 타입을 인식하도록 하는 방법을 설명하고 해당 새 데이터 타입이 데이터베이스에 이미 생성되 있다고 가장합니다.

데이터 타입을 확장하기 위해, 인스턴트를 생성하기 전에 해야합니다. 이 예시는 `Sequelize.INTEGER(11).ZEROFILL.UNSIGNED` 타입으로 빌드업 `NEWTYPE`을 생성합니다.

```js
// myproject/lib/sequelize.js

const Sequelize = require('Sequelize');
const sequelizeConfig = require('../config/sequelize')
const sequelizeAdditions = require('./sequelize-additions')

// Function that adds new datatypes
sequelizeAdditions(Sequelize)

// In this exmaple a Sequelize instance is created and exported
const sequelize = new Sequelize(sequelizeConfig)

module.exports = sequelize
```

```js
// myproject/lib/sequelize-additions.js

module.exports = function sequelizeAdditions(Sequelize) {

  DataTypes = Sequelize.DataTypes

  /*
   * Create new types
   */
  class NEWTYPE extends DataTypes.ABSTRACT {
    // Mandatory, complete definition of the new type in the database
    toSql() {
      return 'INTEGER(11) UNSIGNED ZEROFILL'
    }

    // Optional, validator function
    validate(value, options) {
      return (typeof value === 'number') && (! Number.isNaN(value))
    }

    // Optional, sanitizer
    _sanitize(value) {
      // Force all numbers to be positive
      if (value < 0) {
        value = 0
      }

      return Math.round(value)
    }

    // Optional, value stringifier before sending to database
    _stringify(value) {
      return value.toString()
    }

    // Optional, parser for values received from the database
    static parse(value) {
      return Number.parseInt(value)
    }
  }

  DataTypes.NEWTYPE = NEWTYPE;

  // Mandatory, set key
  DataTypes.NEWTYPE.prototype.key = DataTypes.NEWTYPE.key = 'NEWTYPE'

  // Optional, disable escaping after stringifier. Not recommended.
  // Warning: disables Sequelize protection against SQL injections
  // DataTypes.NEWTYPE.escape = false

  // For convenience
  // `classToInvokable` allows you to use the datatype without `new`
  Sequelize.NEWTYPE = Sequelize.Utils.classToInvokable(DataTypes.NEWTYPE)

}
```

새로운 데이터 타입을 생성한 후, 각 데이터베이스에서 데이터 타입을 맵핑하고 조정해야 합니다.

## PostgreSQL

postgres 데이터베이스에서 새 데이터 유형의 이름이 `pg_new_type`이라고 가정합니다. 이 이름은 `DataTypes.NEWTYPE`에 매핑되어야합니다. 또한 하위 postgres 관련 데이터 타입을 생성해야합니다.

```js
// myproject/lib/sequelize-additions.js

module.exports = function sequelizeAdditions(Sequelize) {

  DataTypes = Sequelize.DataTypes

  /*
   * Create new types
   */

  ...

  /*
   * Map new types
   */

  // Mandatory, map postgres datatype name
  DataTypes.NEWTYPE.types.postgres = ['pg_new_type']

  // Mandatory, create a postgres-specific child datatype with its own parse
  // method. The parser will be dynamically mapped to the OID of pg_new_type.
  PgTypes = DataTypes.postgres

  PgTypes.NEWTYPE = function NEWTYPE() {
    if (!(this instanceof PgTypes.NEWTYPE)) return new PgTypes.NEWTYPE();
    DataTypes.NEWTYPE.apply(this, arguments);
  }
  inherits(PgTypes.NEWTYPE, DataTypes.NEWTYPE);

  // Mandatory, create, override or reassign a postgres-specific parser
  //PgTypes.NEWTYPE.parse = value => value;
  PgTypes.NEWTYPE.parse = DataTypes.NEWTYPE.parse;

  // Optional, add or override methods of the postgres-specific datatype
  // like toSql, escape, validate, _stringify, _sanitize...

}
```

### ranges

[postgres](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-DEFINING)에서 새로운 데이터 타입을 정의한 후, 그것을 Sequelize에 추가하는 것은 쉽지 않습니다.

이 예제에서 postgre range 타입의 이름은 `newtype_range`이고, postgres 데이터 타입 이름은 `pg_new_type` 입니다. `subtypes`와 `castTypes`의 키는 Sequelize 데이터 타입 `Sequelize DataTypes.NEWTYPE.key`의 키입니다.

```js
// myproject/lib/sequelize-additions.js

module.exports = function sequelizeAdditions(Sequelize) {

  DataTypes = Sequelize.DataTypes

  /*
   * Create new types
   */

  ...

  /*
   * Map new types
   */

  ...

  /*
   * Add suport for ranges
   */

  // Add postgresql range, newtype comes from DataType.NEWTYPE.key in lower case
  DataTypes.RANGE.types.postgres.subtypes.newtype = 'newtype_range';
  DataTypes.RANGE.types.postgres.castTypes.newtype = 'pg_new_type';

}
```

새로운 range는 모델 정의에서 `Sequelize.RANGE(Sequelize.NEWTYPE)` 또는 `DataTypes.RANGE(DataTypes.NEWTYPE)`로 사용될 수 있습니다.