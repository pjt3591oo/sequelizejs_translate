# Home

Sequelize는 Postgres, MySQL, MariaDB, SQLite 그리고 Microsoft SQL Server를 위한 promise기반의 ORM입니다. Sequelize는 견고한 트랜젝션, 관계, 지연로딩, 읽기복제를 지원합니다.

Sequelize는 [SEMVER](https://semver.org/)를 따릅니다. ES6를 사용하기 위해 Node V6을 지원합니다.

Sequelize v5는 2019-03-13에 배포되었습니다. [공식적인 타입스크립트 포함](https://sequelize.org/master/manual/typescript)

당신은 Sequelize를 위한 최신 튜토리얼 및 가이드를 보고 있습니다. 당신은 [API 레퍼런스](https://sequelize.org/master/identifiers)에 흥미를 가질것 입니다.

## 예시 

```js
const { Sequelize, Model, DataTypes } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory:');

class User extends Model {}
User.init({
  username: DataTypes.STRING,
  birthday: DataTypes.DATE
}, { sequelize, modelName: 'user' });

sequelize.sync()
  .then(() => User.create({
    username: 'janedoe',
    birthday: new Date(1980, 6, 20)
  }))
  .then(jane => {
    console.log(jane.toJSON());
  });
```

더 많은 Sequelize 사용법을 배우기 위해 좌측에 있는 튜토리얼 메뉴를 읽으세요. [시작하기](https://sequelize.org/master/manual/getting-started)와 함께 시작합니다.