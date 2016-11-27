
Sequelize는 node.js와 io.js를 위한 promise기반의 ORM입니다. Sequelize는 PostgeSQL, MySQL, MariaDB, SQLite ,MSSQL의 디비를 지원하며 트랜젝션, 관계, 백업을 지원합니다.

## 예시 - 사용법

```.js
var Sequelize = require('sequelize');
var sequelize = new Sequelize('database', 'username', 'password');

var User = sequelize.define('user', {
  username: Sequelize.STRING,
  birthday: Sequelize.DATE
});

sequelize.sync().then(function() {
  return User.create({
    username: 'janedoe',
    birthday: new Date(1980, 6, 20)
  });
}).then(function(jane) {
  console.log(jane.get({
    plain: true
  }));
});
```
