# 타입스크립트

v5부터 Sequelize는 고유 한 TypeScript 정의를 제공합니다. TS> = 3.1 만 지원됩니다.

Sequelize는 런타임 속성 할당에 크게 의존하므로 TypeScript는 즉시 유용하지 않습니다. 모델을 실행 가능하게하려면 적절한 양의 수동 형식 선언이 필요합니다.

## 설치

다음 유형 패키지를 수동으로 설치해야합니다.

* @types/node (this is universally required)
* @types/validator
* @types/bluebird

## 사용법

간단한 타입스크립트 예제

```js
import { Sequelize, Model, DataTypes, BuildOptions } from 'sequelize';
import { HasManyGetAssociationsMixin, HasManyAddAssociationMixin, HasManyHasAssociationMixin, Association, HasManyCountAssociationsMixin, HasManyCreateAssociationMixin } from 'sequelize';

class User extends Model {
  public id!: number; // Note that the `null assertion` `!` is required in strict mode.
  public name!: string;
  public preferredName!: string | null; // for nullable fields

  // timestamps!
  public readonly createdAt!: Date;
  public readonly updatedAt!: Date;

  // Since TS cannot determine model association at compile time
  // we have to declare them here purely virtually
  // these will not exist until `Model.init` was called.

  public getProjects!: HasManyGetAssociationsMixin<Project>; // Note the null assertions!
  public addProject!: HasManyAddAssociationMixin<Project, number>;
  public hasProject!: HasManyHasAssociationMixin<Project, number>;
  public countProjects!: HasManyCountAssociationsMixin;
  public createProject!: HasManyCreateAssociationMixin<Project>;

  // You can also pre-declare possible inclusions, these will only be populated if you
  // actively include a relation.
  public readonly projects?: Project[]; // Note this is optional since it's only populated when explicitly requested in code

  public static associations: {
    projects: Association<User, Project>;
  };
}

const sequelize = new Sequelize('mysql://root:asd123@localhost:3306/mydb');

class Project extends Model {
  public id!: number;
  public ownerId!: number;
  public name!: string;

  public readonly createdAt!: Date;
  public readonly updatedAt!: Date;
}

class Address extends Model {
  public userId!: number;
  public address!: string;

  public readonly createdAt!: Date;
  public readonly updatedAt!: Date;
}

Project.init({
  id: {
    type: DataTypes.INTEGER.UNSIGNED, // you can omit the `new` but this is discouraged
    autoIncrement: true,
    primaryKey: true,
  },
  ownerId: {
    type: DataTypes.INTEGER.UNSIGNED,
    allowNull: false,
  },
  name: {
    type: new DataTypes.STRING(128),
    allowNull: false,
  }
}, {
  sequelize,
  tableName: 'projects',
});

User.init({
  id: {
    type: DataTypes.INTEGER.UNSIGNED,
    autoIncrement: true,
    primaryKey: true,
  },
  name: {
    type: new DataTypes.STRING(128),
    allowNull: false,
  },
  preferredName: {
    type: new DataTypes.STRING(128),
    allowNull: true
  }
}, {
  tableName: 'users',
  sequelize: sequelize, // this bit is important
});

Address.init({
  userId: {
    type: DataTypes.INTEGER.UNSIGNED,
  },
  address: {
    type: new DataTypes.STRING(128),
    allowNull: false,
  }
}, {
  tableName: 'address',
  sequelize: sequelize, // this bit is important
});

// Here we associate which actually populates out pre-declared `association` static and other methods.
User.hasMany(Project, {
  sourceKey: 'id',
  foreignKey: 'ownerId',
  as: 'projects' // this determines the name in `associations`!
});

Address.belongsTo(User, {targetKey: 'id'});
User.hasOne(Address,{sourceKey: 'id'});

async function stuff() {
  // Please note that when using async/await you lose the `bluebird` promise context
  // and you fall back to native
  const newUser = await User.create({
    name: 'Johnny',
    preferredName: 'John',
  });
  console.log(newUser.id, newUser.name, newUser.preferredName);

  const project = await newUser.createProject({
    name: 'first!',
  });

  const ourUser = await User.findByPk(1, {
    include: [User.associations.projects],
    rejectOnEmpty: true, // Specifying true here removes `null` from the return type!
  });
  console.log(ourUser.projects![0].name); // Note the `!` null assertion since TS can't know if we included
                                          // the model or not
}
```

## sequelize.define 사용법

TypeScript는 `sequelize.define` 메서드를 사용하여 모델을 정의 할 때 `class` 정의를 생성하는 방법을 모릅니다. 따라서 수동 작업을 수행하고 인터페이스와 형식을 선언하고 결국 `.define` 결과를 정적 형식으로 캐스팅해야합니다.

```js
// We need to declare an interface for our model that is basically what our class would be
interface MyModel extends Model {
  readonly id: number;
}

// Need to declare the static model so `findOne` etc. use correct types.
type MyModelStatic = typeof Model & {
  new (values?: object, options?: BuildOptions): MyModel;
}

// TS can't derive a proper class definition from a `.define` call, therefor we need to cast here.
const MyDefineModel = <MyModelStatic>sequelize.define('MyDefineModel', {
  id: {
    primaryKey: true,
    type: DataTypes.INTEGER.UNSIGNED,
  }
});

function stuffTwo() {
  MyDefineModel.findByPk(1, {
    rejectOnEmpty: true,
  })
  .then(myModel => {
    console.log(myModel.id);
  });
}
```