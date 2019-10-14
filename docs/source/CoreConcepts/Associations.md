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

