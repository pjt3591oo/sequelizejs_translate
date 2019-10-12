# working With Legacy tables

Sequelize는 즉시 사용 가능하지만 약간의 의견이있는 것처럼 보이지만 레거시 테이블을 작업하고 테이블과 필드 이름을 정의하여 (다른 방법으로 생성 된) 응용 프로그램을 쉽게 교정 할 수 있습니다.

## Tables
```js
class User extends Model {}
User.init({
  // ...
}, {
  modelName: 'user',
  tableName: 'users',
  sequelize,
});
```

## Fields
```js
class MyModel extends Model {}
MyModel.init({
  userId: {
    type: Sequelize.INTEGER,
    field: 'user_id'
  }
}, { sequelize });
```


## Primary keys

sequelize는 여러분들의 테이블은 기본적으로 기본 키 속성인 `id`를 가지고 있을 것입니다.

여러분들은 기본 키를 정의 하기위해 다음과 같이하세요.

```js
class Collection extends Model {}
Collection.init({
  uid: {
    type: Sequelize.INTEGER,
    primaryKey: true,
    autoIncrement: true // Automatically gets converted to SERIAL for postgres
  }
}, { sequelize });

class Collection extends Model {}
Collection.init({
  uuid: {
    type: Sequelize.UUID,
    primaryKey: true
  }
}, { sequelize });
```

그리고 여러분들의 모델이 기본키를 전혀 가지고 있지 않는다면 `Model.removeAttribute('id');`을 사용 할 수 있습니다.

## Foreign keys
```js
// 1:1
Organization.belongsTo(User, { foreignKey: 'owner_id' });
User.hasOne(Organization, { foreignKey: 'owner_id' });

// 1:M
Project.hasMany(Task, { foreignKey: 'tasks_pk' });
Task.belongsTo(Project, { foreignKey: 'tasks_pk' });

// N:M
User.belongsToMany(Role, { through: 'user_has_roles', foreignKey: 'user_role_user_id' });
Role.belongsToMany(User, { through: 'user_has_roles', foreignKey: 'roles_identifier' });
```
