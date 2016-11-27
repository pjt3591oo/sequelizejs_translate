
상자 밖에서 Sequelize는 독선적인 것은 사소해 보일 수 있고, 테이블 그리고 필드네임으로 정의된(그렇지 않으면 생성된)것을 여러분의 어플리케이션으로 전달해 줄 것입니다.

# Tables
```.js
sequelize.define('user', {

}, {
  tableName: 'users'
});
```

# Fields
```.js
sequelize.define('modelName', {
  userId: {
    type: Sequelize.INTEGER,
    field: 'user_id'
  }
});
```


# Primary keys

sequelize는 여러분들의 테이블은 기본적으로 기본 키 속성인 `id`를 가지고 있을 것입니다.

여러분들은 기본 키를 정의 하기위해 다음과 같이하세요.

```.js
sequelize.define('collection', {
  uid: {
    type: Sequelize.INTEGER,
    primaryKey: true,
    autoIncrement: true // Automatically gets converted to SERIAL for postgres
  }
});

sequelize.define('collection', {
  uuid: {
    type: Sequelize.UUID,
    primaryKey: true
  }
});
```

그리고 여러분들의 모델이 기본키를 전혀 가지고 있지 않는다면 `Model.removeAttribute('id');`을 사용 할 수 있습니다.

# Foreign keys
```.js
// 1:1
Organization.belongsTo(User, {foreignKey: 'owner_id'});
User.hasOne(Organization, {foreignKey: 'owner_id'});

// 1:M
Project.hasMany(Task, {foreignkey: 'tasks_pk'});
Task.belongsTo(Project, {foreignKey: 'tasks_pk'});

// N:M
User.hasMany(Role, {through: 'user_has_roles', foreignKey: 'user_role_user_id'});
Role.hasMany(User, {through: 'user_has_roles', foreignKey: 'roles_identifier'});
```
