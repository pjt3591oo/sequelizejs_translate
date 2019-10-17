# 관계

이 섹션에서는 관계타입에 대해서 설명합니다. sequelize는 관계를 사용하기 위해 4개의 타입이 있습니다.

1. BelongsTo
2. HasOne
3. HasMany
4. BelongsToMany



## 기본컨셉

### 소스 & 타겟

먼저, 대부분의 관계인 **소스**와 **타겟** 모델에서 사용하는 기본 컨셉으로 시작합니다. 두 모델의 관계를 추가하기 위한 사용을 가정합니다. 여기서 `User`와 `Project` 사이의 관계를 추가합니다

```js
class User extends Model {}
User.init({
  name: Sequelize.STRING,
  email: Sequelize.STRING
}, {
  sequelize,
  modelName: 'user'
});

class Project extends Model {}
Project.init({
  name: Sequelize.STRING
}, {
  sequelize,
  modelName: 'project'
});

User.hasOne(Project);
```

`User` 모델(함수가 호출되는 모델)은 **소스** 입니다. `Project` 모델(인자로 전달되는 모델)은 **타켓**입니다.

### 외래키(Foreign Keys)

sequelize에서 모델들 사이에 관계를 생성할 때, 제약을 가진 외래 참조키는 자동으로 생성됩니다. 아래 설정:

```js
class Task extends Model {}
Task.init({ title: Sequelize.STRING }, { sequelize, modelName: 'task' });
class User extends Model {}
User.init({ username: Sequelize.STRING }, { sequelize, modelName: 'user' });

User.hasMany(Task); // Will add userId to Task model
Task.belongsTo(User); // Will also add userId to Task model
```

다음과 같이 SQL을 생성합니다.

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" SERIAL,
  "username" VARCHAR(255),
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  PRIMARY KEY ("id")
);

CREATE TABLE IF NOT EXISTS "tasks" (
  "id" SERIAL,
  "title" VARCHAR(255),
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "userId" INTEGER REFERENCES "users" ("id") ON DELETE
  SET
    NULL ON UPDATE CASCADE,
    PRIMARY KEY ("id")
);
```

`tasks` 테이블에 `userId` 외래키를 삽입하여 `tasks` 테이블과 `users` 테이블 사이에 관계를 형성하고, `users` 테이블에 대한 참조를 표시합니다. 참조된 사용자가 삭제되면 기본적으로 `userId`는`NULL`로 설정되며, `userId`가 업데이트 되면 업데이트 됩니다. onUpdate 및 onDelete 옵션을 연관 호출에 전달하여이 옵션을 재정의 할 수 있습니다. 유효성 검사 옵션은 `RESTRICT, CASCADE, NO ACTION, SET DEFAULT, SET NULL`입니다.

1:1과 1:m 관계의 기본 옵션은 삭제를 위해 `SET NULL` 이고, 업데이트를 위해 `CASCADE` 입니다. n:m은 둘 다 기본값이 `CASCADE` 입니다. 즉, n : m 연결의 한 쪽에서 행을 삭제하거나 업데이트하면 해당 행을 참조하는 조인 테이블의 모든 행도 삭제되거나 업데이트됩니다.



#### 밑줄 옵션(underscored option): **snake case**

sequelize는 모델을 위한 밑줄 옵션을 허용합니다. 옵션이 `true`일 때 이 옵션은 모든 속성의 필드 옵션을 밑줄이 그어진 버전(snake case)의 이름으로 설정합니다. 이것은 관계에 의해 생성된 외래키에도 적용합니다.

`underscored` 옵셥을 사용하기 위해 마지막 예제를 수정해보자

```js
class Task extends Model {}
Task.init({
  title: Sequelize.STRING
}, {
  underscored: true,
  sequelize,
  modelName: 'task'
});

class User extends Model {}
User.init({
  username: Sequelize.STRING
}, {
  underscored: true,
  sequelize,
  modelName: 'user'
});

// Will add userId to Task model, but field will be set to `user_id`
// This means column name will be `user_id`
User.hasMany(Task);

// Will also add userId to Task model, but field will be set to `user_id`
// This means column name will be `user_id`
Task.belongsTo(User);
```

다음과 같이 SQL을 만듭니다.

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" SERIAL,
  "username" VARCHAR(255),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  PRIMARY KEY ("id")
);

CREATE TABLE IF NOT EXISTS "tasks" (
  "id" SERIAL,
  "title" VARCHAR(255),
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL,
  "user_id" INTEGER REFERENCES "users" ("id") ON DELETE
  SET
    NULL ON UPDATE CASCADE,
    PRIMARY KEY ("id")
);
```

밑줄 옵션은 속성을 카멜 케이스에서 스네이크 케이스로 변경합니다.



#### 순환 의존성 및 비 활성화 제약조건

테이블 사이에 제약조건을 추가하면 `sequelize.sync`를 사용할 때 특정 순서로 데이터베이스에 테이블을 작성해야 합니다. `Task`가 `User`를 참조한다면 `users` 테이블을 `tasks` 테이블이 생성되기 전에 생성해야 합니다. 이것은 종종  순환참조가 발생될 수 있으며, 이 경우 sequelize가 동기화 순서를 찾을 수 없습니다. 문서와 버전의 시나리오를 상상해보세요. 한 문서는 여러 버전을 가질 수 있습니다. 그리고 편의상 문서는 현재 버전에 대한 참조가 있습니다.

```js
class Document extends Model {}
Document.init({
  author: Sequelize.STRING
}, { sequelize, modelName: 'document' });
class Version extends Model {}
Version.init({
  timestamp: Sequelize.DATE
}, { sequelize, modelName: 'version' });

Document.hasMany(Version); // This adds documentId attribute to version
Document.belongsTo(Version, {
  as: 'Current',
  foreignKey: 'currentVersionId'
}); // This adds currentVersionId attribute to document
```

그러나 앞의 코드는 에러를 발생합니다.: `Cyclic dependency found. documents is dependent of itself. Dependency chain: documents -> versions => documents`

이를 완화하기 위해, 우리는 관계시 `constraints: false` 옵션을 전달할 수 있습니다.

```js
Document.hasMany(Version);
Document.belongsTo(Version, {
  as: 'Current',
  foreignKey: 'currentVersionId',
  constraints: false
});
```

그러면 테이블은 옳바르게 동기화합니다.

```sql
CREATE TABLE IF NOT EXISTS "documents" (
  "id" SERIAL,
  "author" VARCHAR(255),
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "currentVersionId" INTEGER,
  PRIMARY KEY ("id")
);

CREATE TABLE IF NOT EXISTS "versions" (
  "id" SERIAL,
  "timestamp" TIMESTAMP WITH TIME ZONE,
  "createdAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "updatedAt" TIMESTAMP WITH TIME ZONE NOT NULL,
  "documentId" INTEGER REFERENCES "documents" ("id") ON DELETE
  SET
    NULL ON UPDATE CASCADE,
    PRIMARY KEY ("id")
);

```



#### 제약없는 외래키 적용

제한 조건이나 연관을 추가하지 않고 다른 테이블을 참조하려는 경우가 있습니다. 이 경우 참조 속성을 스키마 정의에 수동으로 추가하고 이들 사이의 관계를 표시 할 수 있습니다

```js
class Trainer extends Model {}
Trainer.init({
  firstName: Sequelize.STRING,
  lastName: Sequelize.STRING
}, { sequelize, modelName: 'trainer' });

// Series will have a trainerId = Trainer.id foreign reference key
// after we call Trainer.hasMany(series)
class Series extends Model {}
Series.init({
  title: Sequelize.STRING,
  subTitle: Sequelize.STRING,
  description: Sequelize.TEXT,
  // Set FK relationship (hasMany) with `Trainer`
  trainerId: {
    type: Sequelize.INTEGER,
    references: {
      model: Trainer,
      key: 'id'
    }
  }
}, { sequelize, modelName: 'series' });

// Video will have seriesId = Series.id foreign reference key
// after we call Series.hasOne(Video)
class Video extends Model {}
Video.init({
  title: Sequelize.STRING,
  sequence: Sequelize.INTEGER,
  description: Sequelize.TEXT,
  // set relationship (hasOne) with `Series`
  seriesId: {
    type: Sequelize.INTEGER,
    references: {
      model: Series, // Can be both a string representing the table name or a Sequelize model
      key: 'id'
    }
  }
}, { sequelize, modelName: 'video' });

Series.hasOne(Video);
Trainer.hasMany(Series);
```



## 1:1 관계

일대일 관계은 단일 외래 키로 연결된 정확히 두 모델 간의 연관입니다.



### BelongsTo

BelongsTo 관계은 소스 모델에 일대일 관계에 대한 외부 키가 존재하는 연관입니다.

간단한 예는 플레이어에 **Team**의 외래키가 있는 일부 **Player**입니다.

```js
class Player extends Model {}
Player.init({/* attributes */}, { sequelize, modelName: 'player' });
class Team extends Model {}
Team.init({/* attributes */}, { sequelize, modelName: 'team' });

Player.belongsTo(Team); // Will add a teamId attribute to Player to hold the primary key value for Team
```



#### 외래키

belongsTo 관계를 위한 외래키는 타켓 모델의 이름이나 타겟 모델 PK 이름으로부터 생성됩니다.

기본 케스팅은 `camelCase` 입니다. 소스모델이 `underscored: true`로 설정되 있으면 외래키는 `snake_case`로 생성됩니다. 

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

// will add companyId to user
User.belongsTo(Company);

class User extends Model {}
User.init({/* attributes */}, { underscored: true, sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({
  uuid: {
    type: Sequelize.UUID,
    primaryKey: true
  }
}, { sequelize, modelName: 'company' });

// will add companyUuid to user with field company_uuid
User.belongsTo(Company);
```

`as`가 정의 된 경우 대상 모델 이름 대신 사용됩니다.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class UserRole extends Model {}
UserRole.init({/* attributes */}, { sequelize, modelName: 'userRole' });

User.belongsTo(UserRole, {as: 'role'}); // Adds roleId to user rather than userRoleId
```

모든 경우에서 `foreignKey` 옵션을 이용하여 외래키를 재정의 할 수 있습니다. 외래키 옵션을 사용할 때 Sequelize는 그대로 사용합니다.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

User.belongsTo(Company, {foreignKey: 'fk_company'}); // Adds fk_company to User
```



#### 타겟키

타겟키는 소스 모델의 외래키 컬럼이 가리키는 타켓 모델의 열입니다. belongsTo 관계의 타겟키는 타겟 모델의 PK가 기본값 입니다. 사용자 정의 컬럼을 정의하기 위해, `targetKey` 옵션을 사용할 수 있습니다.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

User.belongsTo(Company, {foreignKey: 'fk_companyname', targetKey: 'name'}); // Adds fk_companyname to User
```



### HasOne

HasOne 관계은 일대일 관계에 대한 외래 키가 대상 모델에 존재하는 관계입니다.

```js
class User extends Model {}
User.init({/* ... */}, { sequelize, modelName: 'user' })
class Project extends Model {}
Project.init({/* ... */}, { sequelize, modelName: 'project' })

// One-way associations
Project.hasOne(User)

/*
  In this example hasOne will add an attribute projectId to the User model!
  Furthermore, Project.prototype will gain the methods getUser and setUser according
  to the first parameter passed to define. If you have underscore style
  enabled, the added attribute will be project_id instead of projectId.

  The foreign key will be placed on the users table.

  You can also define the foreign key, e.g. if you already have an existing
  database and want to work on it:
*/

Project.hasOne(User, { foreignKey: 'initiator_id' })

/*
  Because Sequelize will use the model's name (first parameter of define) for
  the accessor methods, it is also possible to pass a special option to hasOne:
*/

Project.hasOne(User, { as: 'Initiator' })
// Now you will get Project.getInitiator and Project.setInitiator

// Or let's define some self references
class Person extends Model {}
Person.init({ /* ... */}, { sequelize, modelName: 'person' })

Person.hasOne(Person, {as: 'Father'})
// this will add the attribute FatherId to Person

// also possible:
Person.hasOne(Person, {as: 'Father', foreignKey: 'DadId'})
// this will add the attribute DadId to Person

// In both cases you will be able to do:
Person.setFather
Person.getFather

// If you need to join a table twice you can double join the same table
Team.hasOne(Game, {as: 'HomeTeam', foreignKey : 'homeTeamId'});
Team.hasOne(Game, {as: 'AwayTeam', foreignKey : 'awayTeamId'});

Game.belongsTo(Team);
```

HasOne 관계라고 하더라도, 대부분의 1 : 1 관계에서는 일반적으로 hasOne의 타겟모델이 belongssTo 관계를 웝합니다,



#### 소스키

소스 키는 타겟모델의 외부 키 속성이 가리키는 소스모델의 속성이다. 기본적으로 haveOne 관계의 소스키는 소스모델의 기본 속성이 된다. 사용자 정의 속성을 사용하기 위해 `sourceKey` 옵션을 사용합니다.

```js
class User extends Model {}
User.init({/* attributes */}, { sequelize, modelName: 'user' })
class Company extends Model {}
Company.init({/* attributes */}, { sequelize, modelName: 'company' });

// Adds companyName attribute to User
// Use name attribute from Company as source attribute
Company.hasOne(User, {foreignKey: 'companyName', sourceKey: 'name'});
```



#### HasOne과 BelongsTo의 차이

Sequelize에서 1:1 관계는 HasOne과 BelongsTo를 사용할 수 있습니다. 그들은 서로 다른 시나리오에서 적합합니다. 예시를 통해 배워봅니다.

Player와 Team을 연결하는 두 개의 테이블이 있다고 가정합니다. 모델을 정의 할 수 있습니다.

```js
class Player extends Model {}
Player.init({/* attributes */}, { sequelize, modelName: 'player' })
class Team extends Model {}
Team.init({/* attributes */}, { sequelize, modelName: 'team' });
```

Sequelize에서 두 개의 모델을 연결할 때, Sequelize에서 두 모델을 연결하면 소스와 대상 모델의 쌍으로 참조 할 수 있습니다. 

**Player**는 소스, **Team** 타켓으로 가질 수 있습니다.

```js
Player.belongsTo(Team);
//Or
Player.hasOne(Team);
```

**Team**는 소스, **Player** 타켓으로 가질 수 있습니다.

```js
Team.belongsTo(Player);
//Or
Team.hasOne(Player);
```

HasOne과 BelongsTo는 각각 다른 모델로 부터 관계키를 추가합니다. hasOne은 타겟모델에서 관계키를 추가하며 BelongsTo는 소스모델에서 관계키를 추가합니다.

다음은 BelongsTo와 HasOne 사용하는 경우의 예시 입니다.

```js
class Player extends Model {}
Player.init({/* attributes */}, { sequelize, modelName: 'player' })
class Coach extends Model {}
Coach.init({/* attributes */}, { sequelize, modelName: 'coach' })
class Team extends Model {}
Team.init({/* attributes */}, { sequelize, modelName: 'team' });
```

`Player` 모델은 team 에 대한 정보를 `teamId` 컬럼으로 가지고 있다고 가정하자. 각의 팀의 `Coach`의 정보는 `Team` 안에 `coachId`가 저장된다. 이 두 시나리오는 각각 다른 모델에 외래키 관계가 존재하기 때문에 다른 종류의 1:1 관계를 요구한다.

**소스**모델에 관계에 대한 정보가 보여질 때 우리는 `hasOne`을 사용할 수 있습니다. 이 경우 `Team` 모델은 `Coach`에 대한 정보를 `coachId` 필드로 저장하므로 `Coach`는 `hasOne`에 적합합니다.



## One-To-Many 관계(1:m, hasMany)

One-To-Many 관계는 하나의 소스를 여러 타겟과 연결합니다. 그러나 타겟들은 하나의 특정 소스와 정확하게 연결됩니다.

```js
class User extends Model {}
User.init({/* ... */}, { sequelize, modelName: 'user' })
class Project extends Model {}
Project.init({/* ... */}, { sequelize, modelName: 'project' })

// OK. Now things get more complicated (not really visible to the user :)).
// First let's define a hasMany association
Project.hasMany(User, {as: 'Workers'})
```

`User`에 `projectId`를 추가합니다. 언더스코어 컬럼을위한 셋팅에 따라 테이블은 `projectId` 또는 `project_id` 둘 중 하나로 저장합니다. 프로젣트의 인스턴스들은 `getWorkers`와 `setWorkers` 접근자를 가져올 수 있습니다.

가끔 관계된 레코드를 다른 컬럼으로 사용하고 싶은 경우, `sourceKey` 옵션을 사용합니다.

```js
class City extends Model {}
City.init({ countryCode: Sequelize.STRING }, { sequelize, modelName: 'city' });
class Country extends Model {}
Country.init({ isoCode: Sequelize.STRING }, { sequelize, modelName: 'country' });

// Here we can connect countries and cities base on country code
Country.hasMany(City, {foreignKey: 'countryCode', sourceKey: 'isoCode'});
City.belongsTo(Country, {foreignKey: 'countryCode', targetKey: 'isoCode'});
```

지금까지 우리는 일반적인 관계에 대해 다뤄보았습니다. 그러나 우리는 더 많은 것을 원합니다. 다음 섹션에서는 다 대 다관계에 대해서 다뤄봅니다.



## Belongs-To-Many 관계

Belongs-To-Many 관계는 소스를 다른 타겟들과 연결할 때 사용됩니다. 또한 타겟은 여러 소스들과 연결할 수 있습니다.

```js
Project.belongsToMany(User, {through: 'UserProject'});
User.belongsToMany(Project, {through: 'UserProject'});
```

이것은 `projectId`와 `userId`를 외래키로 가지고 있는 UserProject 모델을 새로 생성합니다. 속성이 camelcase 여부는 테이블에 의해 결합 된 두 가지 모델 (이 경우 User 및 Project)에 따라 다릅니다.

through를 정의하는 것은 **필수**입니다. Sequelize는 이전에 이름을 자동으로 생성하려고 시도했지만 항상 가장 논리적으로 설정되는 것은 아닙니다. 

이것은 `getUsers`, `setUsers`, `addUser`, `addUsers` 메서드를 `Project`에 추가하고, `getProjects`, `setProjects`, `addProject`, `addProjects` 메서드를 `User`에 추가합니다.

때때로 관계에서 사용할 때 모델의 이름을 바꾸고 싶을 수도 있습니다. 별명 (`as`) 옵션을 사용하여 사용자를 작업자 및 프로젝트로 작업으로 정의 해 봅시다. 사용할 외래키도 수동으로 정의합니다.

```js
User.belongsToMany(Project, { as: 'Tasks', through: 'worker_tasks', foreignKey: 'userId' })
Project.belongsToMany(User, { as: 'Workers', through: 'worker_tasks', foreignKey: 'projectId' })
```

`foreignKey`는 **through** 관계에서 소스모델 키 설정할 수 있습니다. `otherKey`는**through** 관계에서 타겟모델 키 설정할 수 있습니다.

```js
User.belongsToMany(Project, { as: 'Tasks', through: 'worker_tasks', foreignKey: 'userId', otherKey: 'projectId'})
```

물론 belongsToMany로 자기 참조를 정의 할 수도 있습니다 .

```js
Person.belongsToMany(Person, { as: 'Children', through: 'PersonChildren' })
```



### 소스와 타겟키

PK를 사용하지 않는 많은 관계를 생성하고 싶다면 일부 설정 작업이 필요합니다. 두 개의 끝에 `sourceKey`(선택적으로 `targetKey`)를 적절하게 선택해야 합니다. 또한 관계에 대해 적절한 색인을 작성했는지 확인해야합니다. 다음은 예입니다.

```js
const User = this.sequelize.define('User', {
  id: {
    type: DataTypes.UUID,
    allowNull: false,
    primaryKey: true,
    defaultValue: DataTypes.UUIDV4,
    field: 'user_id'
  },
  userSecondId: {
    type: DataTypes.UUID,
    allowNull: false,
    defaultValue: DataTypes.UUIDV4,
    field: 'user_second_id'
  }
}, {
  tableName: 'tbl_user',
  indexes: [
    {
      unique: true,
      fields: ['user_second_id']
    }
  ]
});

const Group = this.sequelize.define('Group', {
  id: {
    type: DataTypes.UUID,
    allowNull: false,
    primaryKey: true,
    defaultValue: DataTypes.UUIDV4,
    field: 'group_id'
  },
  groupSecondId: {
    type: DataTypes.UUID,
    allowNull: false,
    defaultValue: DataTypes.UUIDV4,
    field: 'group_second_id'
  }
}, {
  tableName: 'tbl_group',
  indexes: [
    {
      unique: true,
      fields: ['group_second_id']
    }
  ]
});

User.belongsToMany(Group, {
  through: 'usergroups',
  sourceKey: 'userSecondId'
});
Group.belongsToMany(User, {
  through: 'usergroups',
  sourceKey: 'groupSecondId'
});
```

조인 테이블에 추가 속성을 원할 경우 관계을 정의하기 전에 순서에 따라 조인 테이블의 모델을 정의한 다음 새 모델을 만드는 대신 조인에 해당 모델을 사용해야한다고 sequelize에 알릴 수 있습니다.

```js
class User extends Model {}
User.init({}, { sequelize, modelName: 'user' })
class Project extends Model {}
Project.init({}, { sequelize, modelName: 'project' })
class UserProjects extends Model {}
UserProjects.init({
  status: DataTypes.STRING
}, { sequelize, modelName: 'userProjects' })

User.belongsToMany(Project, { through: UserProjects })
Project.belongsToMany(User, { through: UserProjects })
```

사용자에게 새로운 프로젝트를 추가하기 위해 조인 테이블을 위해 포함된 속성에 `options.through`를 전달할 수 있습니다.

```js
user.addProject(project, { through: { status: 'started' }})
```

앞의 코드는 UserProjects 테이블에 projectId와 userId를 추가하고 이전에 정의 된 PK 속성을 제거합니다. 테이블은 두 테이블의 키 조합으로 고유하게 식별되며 다른 PK 컬럼을 가질 이유가 없습니다. `UserProjects` 모델에서 PK를 억지로 추가하기 위해 수동으로 추가할 수 있습니다.

```js
class UserProjects extends Model {}
UserProjects.init({
  id: {
    type: Sequelize.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  status: DataTypes.STRING
}, { sequelize, modelName: 'userProjects' })
```

Belongs-To-Many를 통해 관계를 기반으로 쿼리 할 수 있고 특정 속성을 지정할 수 있습니다. **through**와 `findAll`을 사용하는 예제입니다.

```js
User.findAll({
  include: [{
    model: Project,
    through: {
      attributes: ['createdAt', 'startedAt', 'finishedAt'],
      where: {completed: true}
    }
  }]
});
```

Belongs-To-Many는 PK가 모델을 통해 존재하지 않을 때 유니크 한 키를 생성한다. 유니크키는 **uniqueKey** 옵션을 사용하여 재정의할 수 있습니다.



## 네이밍 전략

기본적으로 sequelize는 모델 이름 (`sequelize.define`에 전달 된 이름)을 사용하여 연결에 사용될 때 모델 이름을 알아냅니다. 예를 들어, 예를 들어, `user` 라는 모델은 관련 모델의 인스턴스에 `get/set/add` 기능을 추가하고, `User`라는 모델은 동일한 기능을 추가하지만 속성은 `.user`로 명명됩니다. `.User`(상위 사례 U에 주목)가 즉시 로드됩니다.

이미 봤듯이, 관계에서 `a` 를 이용하여 별칭을 만들 수 있습니다. 단일 연관성(하나에 속하고 속함)에서는 별칭이 단수여야 하지만, 많은 연관성(많은 연관성)의 경우 복수여야 합니다. 그런 다음 Sequelize는 [inflection](https://www.npmjs.com/package/inflection) 라이브러리를 사용하여 별칭을 단수 형태로 변환합니다. 그러나 이것은 불규칙하거나 비영어적인 단어에서 항상 효과가 있는 것은 아닙니다. 이 경우 별칭의 복수 형식과 단수 형식을 모두 제공할 수 있습니다.

```js
User.belongsToMany(Project, { as: { singular: 'task', plural: 'tasks' }})
// Notice that inflection has no problem singularizing tasks, this is just for illustrative purposes.
```

모델이 항상 관계에서 동일한 별칭을 사용한다는 것을 알고 있는 경우 모델을 생성할 때 해당 별칭을 제공할 수 있습니다.

```js
class Project extends Model {}
Project.init(attributes, {
  name: {
    singular: 'task',
    plural: 'tasks',
  },
  sequelize,
  modelName: 'project'
})

User.belongsToMany(Project);
```

유저 인스턴스에 `add/set/get` 함수를 추가합니다.

기억하세요, 관계된 이름을 바꾸기 위해 `as`를 사용하는 것은 외래키 이름을 바꾸는 것 입니다. `as`를 사용할 때, 특정 외래키를 지정하는 것이 안전합니다.

```js
Invoice.belongsTo(Subscription)
Subscription.hasMany(Invoice)
```

`as` 없이, 정확하게 `subscriptionId`를 추가합니다. 그러나 `Invoice.belongsTo(Subscription, { as: 'TheSubscription' })`를 호출한다면, `subscriptionsId`와 `theSubscriptionId`를 가질 수 있습니다. 왜냐하면 sequelize는 두개의 호출이 같은 관계를 가진다는 것을 알지 못합니다.  이를 해결할 수 있는 방법은 `foreignKey`를 사용합니다.

```js
Invoice.belongsTo(Subscription, { as: 'TheSubscription', foreignKey: 'subscription_id' })
Subscription.hasMany(Invoice, { foreignKey: 'subscription_id' })
```



## 객체 연관

Sequelize는 마법같은 많은 일들을 하기 때문에 관계설정을 한 후 `Sequelize.sync`를 호출할 수 있습니다. 다음을 따르면 할 수 있습니다.

```js
Project.hasMany(Task)
Task.belongsTo(Project)

Project.create()...
Task.create()...
Task.create()...

// save them... and then:
project.setTasks([task1, task2]).then(() => {
  // saved!
})

// ok, now they are saved... how do I get them later on?
project.getTasks().then(associatedTasks => {
  // associatedTasks is an array of tasks
})

// You can also pass filters to the getter method.
// They are equal to the options you can pass to a usual finder method.
project.getTasks({ where: 'id > 10' }).then(tasks => {
  // tasks with an id greater than 10 :)
})

// You can also only retrieve certain fields of a associated object.
project.getTasks({attributes: ['title']}).then(tasks => {
  // retrieve tasks with the attributes "title" and "id"
})
```

생성된 관계를 삭제하기 위해 특정 Id 없이 설정 메서드를 호출할 수 있습니다.

```js
// remove the association with task1
project.setTasks([task2]).then(associatedTasks => {
  // you will get task2 only
})

// remove 'em all
project.setTasks([]).then(associatedTasks => {
  // you will get an empty array
})

// or remove 'em more directly
project.removeTask(task1).then(() => {
  // it's gone
})

// and add 'em again
project.addTask(task1).then(() => {
  // it's back again
})
```

물론 그 반대의 경우도 가능합니다.

```js
// project is associated with task1 and task2
task2.setProject(null).then(() => {
  // and it's gone
})
```

hasOne/belongsTo은 기본적으로 같습니다.

```js
Task.hasOne(User, {as: "Author"})
Task.setAuthor(anAuthor)
```

사용자 정의에 따른 테이블 조인과 함께 관계를 추가하는 것은 두 가지 방법으로 할 수 있습니다 (이전 장에서 정의 된 연관을 계속):

```js
// Either by adding a property with the name of the join table model to the object, before creating the association
project.UserProjects = {
  status: 'active'
}
u.addProject(project)

// Or by providing a second options.through argument when adding the association, containing the data that should go in the join table
u.addProject(project, { through: { status: 'active' }})


// When associating multiple objects, you can combine the two options above. In this case the second argument
// will be treated as a defaults object, that will be used if no data is provided
project1.UserProjects = {
    status: 'inactive'
}

u.setProjects([project1, project2], { through: { status: 'active' }})
// The code above will record inactive for project one, and active for project two in the join table
```

사용자 정의에 따라 조인 테이블을 가진 관계에서 데이터를 가져올 때, 조인 테이블에서 가져온 데이터는 DAO 인스턴스가 반환됩니다.

```js
u.getProjects().then(projects => {
  const project = projects[0]

  if (project.UserProjects.status === 'active') {
    // .. do magic

    // since this is a real DAO instance, you can save it directly after you are done doing magic
    return project.UserProjects.save()
  }
})
```

조인 테이블에서 특정 속성이 필요하다면 배열 형태로 속성을 제공할 수 있습니다.

```js
// This will select only name from the Projects table, and only status from the UserProjects table
user.getProjects({ attributes: ['name'], joinTableAttributes: ['status']})
```



## 관계 확인

이미 다른 관계(오직 M:N)를 가졌는지 확인할 수 있습니다.

```js
// check if an object is one of associated ones:
Project.create({ /* */ }).then(project => {
  return User.create({ /* */ }).then(user => {
    return project.hasUser(user).then(result => {
      // result would be false
      return project.addUser(user).then(() => {
        return project.hasUser(user).then(result => {
          // result would be true
        })
      })
    })
  })
})

// check if all associated objects are as expected:
// let's assume we have already a project and two users
project.setUsers([user1, user2]).then(() => {
  return project.hasUsers([user1]);
}).then(result => {
  // result would be true
  return project.hasUsers([user1, user2]);
}).then(result => {
  // result would be true
})
```



## 고급개념



### 범위

이 섹션은 연결 범위와 관련이 있습니다. 관계 범위와 관련 모델의 범위에 대한 정의는 [범위](https://sequelize.org/master/manual/scopes.html)를 참조하십시오.

관계 범위를 사용하면 관계에 범위 (`get` 및 `create`을 위한 기본 속성 세트)를 배치 할 수 있습니다. 범위는 관계된 모델 (연결 타겟)과 n : m 관계에 대한 through 테이블에 둘 수 있습니다.  

#### 1:n

우리는 Comment, Post, Image 모델을 가지고 있습니다. 댓글은 `commentableId`과 `commentable`을 포스트와 이미지와 관계를 가집니다. 우리는 Post와 Image는 `Commentable` 입니다.

```js
class Post extends Model {}
Post.init({
  title: Sequelize.STRING,
  text: Sequelize.STRING
}, { sequelize, modelName: 'post' });

class Image extends Model {}
Image.init({
  title: Sequelize.STRING,
  link: Sequelize.STRING
}, { sequelize, modelName: 'image' });

class Comment extends Model {
  getItem(options) {
    return this[
      'get' +
        this.get('commentable')
          [0]
          .toUpperCase() +
        this.get('commentable').substr(1)
    ](options);
  }
}

Comment.init({
  title: Sequelize.STRING,
  commentable: Sequelize.STRING,
  commentableId: Sequelize.INTEGER
}, { sequelize, modelName: 'comment' });

Post.hasMany(Comment, {
  foreignKey: 'commentableId',
  constraints: false,
  scope: {
    commentable: 'post'
  }
});

Comment.belongsTo(Post, {
  foreignKey: 'commentableId',
  constraints: false,
  as: 'post'
});

Image.hasMany(Comment, {
  foreignKey: 'commentableId',
  constraints: false,
  scope: {
    commentable: 'image'
  }
});

Comment.belongsTo(Image, {
  foreignKey: 'commentableId',
  constraints: false,
  as: 'image'
});
```

`constraints: false`는 `commentableId` 열이 여러 테이블을 참조하므로 REFERENCES 제약 조건을 추가 할 수 없으므로 참조 제약 조건을 비활성화합니다

Image -> Comment, Post -> Comment 관계는 각각 `commentable: 'img'` 와, `commentable: 'post'`를 이용하여  범위(scope)를 지정합니다. 이 범위(scope)는 관계 함수를 사용할 때 자동으로 적용됩니다.

```js
image.getComments()
// SELECT "id", "title", "commentable", "commentableId", "createdAt", "updatedAt" FROM "comments" AS
// "comment" WHERE "comment"."commentable" = 'image' AND "comment"."commentableId" = 1;

image.createComment({
  title: 'Awesome!'
})
// INSERT INTO "comments" ("id","title","commentable","commentableId","createdAt","updatedAt") VALUES
// (DEFAULT,'Awesome!','image',1,'2018-04-17 05:36:40.454 +00:00','2018-04-17 05:36:40.454 +00:00')
// RETURNING *;

image.addComment(comment);
// UPDATE "comments" SET "commentableId"=1,"commentable"='image',"updatedAt"='2018-04-17 05:38:43.948
// +00:00' WHERE "id" IN (1)
```

`Comment`의 `getItem` 함수는 완벽한 그림입니다. 댓글 가능한 문자열을 `getImage`,`getPost` 호출로 변환하여 댓글이 게시글(post)에 해당하는지 이미지(image)에 해당하는지 추상화합니다. `getItem(options)`에 일반적인 옵션 객체를 인자로 전달하여 위치 또는 조건을 지정할 수 있습니다.



#### n:m

다형성 모델의 아이디어를 계속하면서 태그 테이블을 고려하십시오. 항목에는 여러 개의 태그가있을 수 있으며 태그는 여러 개의 항목과 관련 될 수 있습니다.

간결하게하기 위해 이 예제는 Post 모델 만 표시하지만 실제로 Tag는 다른 여러 모델과 관련이 있습니다.

```js
class ItemTag extends Model {}
ItemTag.init({
  id: {
    type: Sequelize.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  tagId: {
    type: Sequelize.INTEGER,
    unique: 'item_tag_taggable'
  },
  taggable: {
    type: Sequelize.STRING,
    unique: 'item_tag_taggable'
  },
  taggableId: {
    type: Sequelize.INTEGER,
    unique: 'item_tag_taggable',
    references: null
  }
}, { sequelize, modelName: 'item_tag' });

class Tag extends Model {}
Tag.init({
  name: Sequelize.STRING,
  status: Sequelize.STRING
}, { sequelize, modelName: 'tag' });

Post.belongsToMany(Tag, {
  through: {
    model: ItemTag,
    unique: false,
    scope: {
      taggable: 'post'
    }
  },
  foreignKey: 'taggableId',
  constraints: false
});

Tag.belongsToMany(Post, {
  through: {
    model: ItemTag,
    unique: false
  },
  foreignKey: 'tagId',
  constraints: false
});
```

범위가 지정된 열 (`taggable`)이 이제 through 모델 (ItemTag)에 있습니다.

예를 들어 through 모델 (ItemTag)과 타겟모델 (Tag)의 범위를 적용하여 게시물에 대해 보류중인 모든 태그를 가져 오기 위해 보다 제한적인 관계를 정의 할 수도 있습니다.

```js
Post.belongsToMany(Tag, {
  through: {
    model: ItemTag,
    unique: false,
    scope: {
      taggable: 'post'
    }
  },
  scope: {
    status: 'pending'
  },
  as: 'pendingTags',
  foreignKey: 'taggableId',
  constraints: false
});

post.getPendingTags();
```

```sql
SELECT
  "tag"."id",
  "tag"."name",
  "tag"."status",
  "tag"."createdAt",
  "tag"."updatedAt",
  "item_tag"."id" AS "item_tag.id",
  "item_tag"."tagId" AS "item_tag.tagId",
  "item_tag"."taggable" AS "item_tag.taggable",
  "item_tag"."taggableId" AS "item_tag.taggableId",
  "item_tag"."createdAt" AS "item_tag.createdAt",
  "item_tag"."updatedAt" AS "item_tag.updatedAt"
FROM
  "tags" AS "tag"
  INNER JOIN "item_tags" AS "item_tag" ON "tag"."id" = "item_tag"."tagId"
  AND "item_tag"."taggableId" = 1
  AND "item_tag"."taggable" = 'post'
WHERE
  ("tag"."status" = 'pending');
```

`constraints: false` 는 `taggableId` 열에 대한 참조 제약 조건을 비활성화합니다. 열은 다형성이기 때문에 특정 테이블을 `참조`한다고 말할 수 없습니다.



### 관계들의 생성

모든 요소가 새로운 경우 한 단계로 중첩 된 연관으로 인스턴스를 작성할 수 있습니다.

#### BelongsTo / HasMany / HasOne 관계

다음 모델을 고려하세요.

```js
class Product extends Model {}
Product.init({
  title: Sequelize.STRING
}, { sequelize, modelName: 'product' });
class User extends Model {}
User.init({
  firstName: Sequelize.STRING,
  lastName: Sequelize.STRING
}, { sequelize, modelName: 'user' });
class Address extends Model {}
Address.init({
  type: Sequelize.STRING,
  line1: Sequelize.STRING,
  line2: Sequelize.STRING,
  city: Sequelize.STRING,
  state: Sequelize.STRING,
  zip: Sequelize.STRING,
}, { sequelize, modelName: 'address' });

Product.User = Product.belongsTo(User);
User.Addresses = User.hasMany(Address);
// Also works for `hasOne`
```

다음과 같은 방법으로 새로운 `Project`, `User` 및 하나 이상의 `Address` 를 한 단계로 만들 수 있습니다.

```js
return Product.create({
  title: 'Chair',
  user: {
    firstName: 'Mick',
    lastName: 'Broadstone',
    addresses: [{
      type: 'home',
      line1: '100 Main St.',
      city: 'Austin',
      state: 'TX',
      zip: '78704'
    }]
  }
}, {
  include: [{
    association: Product.User,
    include: [ User.Addresses ]
  }]
});
```

여기서 우리의 사용자 모델은 소문자 u를 가진 `user`라고 불립니다. 이것은 객체의 속성도 `user` 여야 함을 의미합니다. `sequelize.define`에 지정된 이름이` User` 인 경우 개체의 키도 `User` 여야합니다. 다수의 hasMany 연관을 제외하고는 주소에 대해서도 마찬가지입니다.



#### BelongsTo association with an alias

이전 예는 연관 별명을 지원하도록 확장 될 수 있습니다.

```js
const Creator = Product.belongsTo(User, { as: 'creator' });

return Product.create({
  title: 'Chair',
  creator: {
    firstName: 'Matt',
    lastName: 'Hansen'
  }
}, {
  include: [ Creator ]
});
```



#### HasMany / BelongsToMany association

제품을 여러 태그와 연결하는 기능을 소개하겠습니다. 모델 설정은 다음과 같습니다.

```js
class Tag extends Model {}
Tag.init({
  name: Sequelize.STRING
}, { sequelize, modelName: 'tag' });

Product.hasMany(Tag);
// Also works for `belongsToMany`.
```

여러 태그와 함께 제품을 사용할 수 있습니다.

```js
Product.create({
  id: 1,
  title: 'Chair',
  tags: [
    { name: 'Alpha'},
    { name: 'Beta'}
  ]
}, {
  include: [ Tag ]
})
```

그리고, as를 사용하는 형태로 예시를 수정할 수 있습니다.

```js
const Categories = Product.hasMany(Tag, { as: 'categories' });

Product.create({
  id: 1,
  title: 'Chair',
  categories: [
    { id: 1, name: 'Alpha' },
    { id: 2, name: 'Beta' }
  ]
}, {
  include: [{
    association: Categories,
    as: 'categories'
  }]
})
```

