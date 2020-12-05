# sequelize-typescript 中文文档
提供给 sequelize (v5) 使用的装饰器和一些其他功能。

sequelize-typescript [项目地址](https://github.com/RobinBuschmann/sequelize-typescript)


 - [安装](#安装)
 - [升级到 `sequelize-typescript@1`](#升级到`sequelize-typescript@1`)
 - [模型定义](#模型定义)
   - [`@Table` API](#table-api)
   - [`@Column` API](#column-api)
 - [使用](#使用)
   - [配置](#配置)
   - [globs 匹配](#globs-匹配)
   - [模型路径解析](#模型路径解析)
 - [模型关联](#模型关联)
   - [一对多](#一对多)
   - [多对多](#多对多)
   - [一对一](#一对一)
   - [`@ForeignKey`, `@BelongsTo`, `@HasMany`, `@HasOne`, `@BelongsToMany` API](#foreignkey-belongsto-hasmany-hasone-belongstomany-api)
   - [使用 getter 和 setter](#type-safe-usage-of-auto-generated-functions)
   - [同一模型的多种关联](#multiple-relations-of-same-models)
 - [索引](#indexes)
   - [`@Index` API](#index)
   - [`createIndexDecorator()` API](#createindexdecorator)
 - [Repository 模式](#repository-mode)
   - [如何开启 repository 模式?](#how-to-enable-repository-mode)
   - [如何使用 repository 模式?](#how-to-use-repository-mode)
   - [如何在 repository 模式下关联?](#how-to-use-associations-with-repository-mode)
   - [repository 模式的限制](#limitations-of-repository-mode)
 - [模型校验](#model-validation)
 - [作用域](#scopes)
 - [Hooks 钩子](#hooks)
 - [为何使用 `() => Model`?](#user-content-why---model)
 - [风格推荐和限制](#recommendations-and-limitations)

## 安装
*sequelize-typescript* 依赖 [sequelize](https://github.com/sequelize/sequelize), 点击查看 sequelize 的 [typescript 编码指南](https://docs.sequelizejs.com/manual/typescript.html) 和 [reflect-metadata](https://www.npmjs.com/package/reflect-metadata) 的详细文档
```
npm install sequelize
npm install @types/bluebird @types/node @types/validator
npm install reflect-metadata
```
```
npm install sequelize-typescript
```
需要在 `tsconfig.json` 文件下配置如下项：
```json
"target": "es6", // or a more recent ecmascript version
"experimentalDecorators": true,
"emitDecoratorMetadata": true
```

### ⚠️ sequelize@4
`sequelize@4` 依赖 `sequelize-typescript@0.6`. 查看 `0.6` 版本的
[文档](https://github.com/RobinBuschmann/sequelize-typescript/tree/0.6.X) .
```
npm install sequelize-typescript@0.6
```

## 升级到`sequelize-typescript@1`
`sequelize-typescript@1` 只能在 `sequelize@5>=` 版本下运行。
如果你使用 `sequelize@4` 或者 `sequelize@3` ， 请使用 `sequelize-typescript@0.6`。

### 破坏性更新
`sequelize@5` 的破坏性更新同样会影响 `sequelize-typescript@1`。
查看 [升级到 V5](https://github.com/sequelize/sequelize/blob/master/docs/upgrade-to-v5.md) 详情。

#### 官方 TypeScript 支持
sequelize-typescript 目前使用 sequlize 官方的 TypeScript 定义
(点击 [查看](https://github.com/sequelize/sequelize/blob/master/docs/upgrade-to-v5.md#typescript-support)).
请注意以下事项：
- 绝大部分之前版本 sequelize-typescript 所提供的 interface 被官方所替代
- 不再使用`@types/sequelize`
- `@types/bluebird` 不再是一个显式依赖
- 官方类型声明没 sequelize-typescript 之前版本的严格

#### Sequelize 选项
- `SequelizeConfig` 重命名为 `SequelizeOptions`
- `modelPaths` 重命名为 `models`

#### 作用域 选项
`@Scopes` 和 `@DefaultScope` 装饰器目前接收匿名函数作为参数
```ts
@DefaultScope(() => ({...}))
@Scopes(() => ({...}))
```
已废弃：
```ts
@DefaultScope({...})
@Scopes({...}))
```

### Repository 模式
`sequelize-typescript@1` 带来了 repository 模式. 点击 [这里](#repository-mode) 查看详情.


## 模型定义
```typescript
import {Table, Column, Model, HasMany} from 'sequelize-typescript';

@Table
class Person extends Model<Person> {

  @Column
  name: string;

  @Column
  birthday: Date;

  @HasMany(() => Hobby)
  hobbies: Hobby[];
}
``` 
模型需要从 `Model` 类 扩展来，并且需要被装饰器 `@Table` 所注解。所有在数据库上表现为列属性需要 `@Column` 注解。

点击 [查看](https://github.com/RobinBuschmann/sequelize-typescript-example) 更多进阶示例。

### `@Table`
`@Table` 注解能在不传任何参数的情况下使用，可以通过传入一个对象来指定模型如何定义
（Sequelize 中所有的 [模型定义选项](https://sequelize.org/v5/manual/models-definition.html#configuration) 都能使用）：
```typescript
@Table({
  timestamps: true,
  ...
})
class Person extends Model<Person> {}
```
#### Table API

| 装饰器                           | 描述                                                                                                                                                                                                              |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@Table`                         | 自动设置 `options.tableName=<CLASS_NAME>` 和  `options.    modelName=<CLASS_NAME>`                                                                                                                                |
| `@Table(options: DefineOptions)` | 按 [模型定义选项](https://sequelize.org/v5/manual/models-definition.html#configuration) 设置 (如果没有被模型定义选项所指定的话，会同时设置 `options.tableName=<CLASS_NAME>` 和  `options.modelName=<CLASS_NAME>`) |

#### 主键
主键 (`id`) 会从 `Model` 类中所继承. 这个主键默认被设置为 `INTEGER` 且
`autoIncrement=true` (这种行为是 sequelize 原生的). 这个 id 会被其他标记为主键的值所覆盖。所以请设置 `@Column({primaryKey: true})`， 或者同时使用 `@PrimaryKey` 和 `@Column`.

#### `@CreatedAt`, `@UpdatedAt`, `@DeletedAt`
能安全定义 `createdAt`, `updatedAt` 和 `deletedAt` 属性的注解:
```typescript
  @CreatedAt
  creationDate: Date;

  @UpdatedAt
  updatedOn: Date;

  @DeletedAt
  deletionDate: Date;
```

| 装饰器       | 描述                                                                  |
| ------------ | --------------------------------------------------------------------- |
| `@CreatedAt` | 设置 `timestamps=true` 和 `createdAt='creationDate'`                  |
| `@UpdatedAt` | 设置 `timestamps=true` 和 `updatedAt='updatedOn'`                     |
| `@DeletedAt` | 设置 `timestamps=true`, `paranoid=true` 和 `deletedAt='deletionDate'` |

### `@Column`
`@Column` 注解能在不传入任何参数下使用，不过传入参数来让 js 类型可以被自行推断是相当有必要的(查看 [类型推断](#type-inference) 细节).
```typescript
  @Column
  name: string;
```
如果类型不能或者不应该被推断，请使用
```typescript
import {DataType} from 'sequelize-typescript';

  @Column(DataType.TEXT)
  name: string;
```
或者，对于更详细的描述，可以使用对象来描述
(所有的 sequelize 的 [模型属性选项](https://sequelize.org/v5/manual/models-definition.html#configuration) 都可以使用):
```typescript
  @Column({
    type: DataType.FLOAT,
    comment: 'Some value',
    ...
  })
  value: number;
```
#### Column API

| 装饰器                               | 描述                                                                                                              |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| `@Column`                            | 尝试从其 js 类型去定义 [数据类型](https://sequelize.org/v5/manual/models-definition.html#data-types) from js type |
| `@Column(dataType: DataType)`        | 手动指定 [数据类型](https://sequelize.org/v5/manual/models-definition.html#data-types)                            |
| `@Column(options: AttributeOptions)` | 按 [模型属性选项](https://sequelize.org/v5/manual/models-definition.html#configuration) 设置                      |

#### *事半功倍*
如果你喜欢使用装饰器: *sequelize-typescript* 还提供了更多可供使用. 下面这些装饰器能和 `@Column` 注解一起使用，来让一些属性更容易使用：

| 装饰器                            | 描述                                       |
| --------------------------------- | ------------------------------------------ |
| `@AllowNull(allowNull?: boolean)` | 设置 `attribute.allowNull` (默认为 `true`) |
| `@AutoIncrement`                  | 设置 `attribute.autoIncrement=true`        |
| `@Unique`                         | 设置 `attribute.unique=true`               |
| `@Default(value: any)`            | 设置 `attribute.defaultValue` 为指定值     |
| `@PrimaryKey`                     | 设置 `attribute.primaryKey=true`           |
| `@Comment(value: string)`         | 设置 `attribute.comment` 为指定值          |
| 校验器注解                        | 了解 [模型校验](#model-validation)         |

### 类型推断
以下类型能从 javascript 类型上自行推断，其他类型需要需要去手动指定。

| 数据库数据类型 | Sequelize 数据类型 |
| -------------- | ------------------ |
| `string`       | `STRING`           |
| `boolean`      | `BOOLEAN`          |
| `number`       | `INTEGER`          |
| `Date`         | `DATE`             |
| `Buffer`       | `BLOB`             |

### 访问器
get/set 访问器 也能正常使用
```typescript
@Table
class Person extends Model<Person> {

  @Column
  get name(): string {
    return 'My name is ' + this.getDataValue('name');
  }

  set name(value: string) {
    this.setDataValue('name', value);
  }
}
```

## 使用
除了一些微小的变化， *sequelize-typescript* 使用起来像原汁原味的 sequelize。
(查阅 sequelize [文档](https://docs.sequelizejs.com/manual/tutorial/models-usage.html))
### 配置
为了使定义的模型可用，你必须从 `sequelize-typescript` 中配置一个 `Sequelize` 实例 (!).
```typescript
import {Sequelize} from 'sequelize-typescript';

const sequelize =  new Sequelize({
  database: 'some_db',
  dialect: 'sqlite',
  username: 'root',
  password: '',
  storage: ':memory:',
  models: [__dirname + '/models'], // or [Player, Team],
});
```
在你使用你的模型之前，你必须告诉 sequelize 去那里找到这些模型。所以要么将 `models` 写到 sequelize 配置中，要么通过调用 `sequelize.addModels([Person])` 或 `sequelize.addModels([__dirname + '/models'])` 来添加：

```typescript
sequelize.addModels([Person]);
sequelize.addModels(['path/to/models']);
```
### globs 匹配
```typescript
import {Sequelize} from 'sequelize-typescript';

const sequelize =  new Sequelize({
  ...
  models: [__dirname + '/**/*.model.ts']
});
// or
sequelize.addModels([__dirname + '/**/*.model.ts']);
```

#### 模型路径解析
一个模型通过其文件名与文件匹配。 例如：
```typescript
// File User.ts matches the following exported model.
export class User extends Model<User> {}
```
这个是通过把文件名与所有导出的模型所比较来完成的，可以通过指定 `modelMatch` 函数来自定义模型和文件的匹配。

比方说，如果你的模型被命名为`user.model.ts`，而你的类叫做`User`，你可以通过使用以下的函数来匹配这两者

```typescript
import {Sequelize} from 'sequelize-typescript';

const sequelize =  new Sequelize({
  models: [__dirname + '/models/**/*.model.ts']
  modelMatch: (filename, member) => {
    return filename.substring(0, filename.indexOf('.model')) === member.toLowerCase();
  },
});
```

对于每一个匹配 `*.model.ts` 规则的文件，`modelMatch` 函数会将这个文件导出的所有成员都匹配一遍，例如下面这个文件：

```TypeScript
//user.model.ts
import {Table, Column, Model} from 'sequelize-typescript';

export const UserN = 'Not a model';
export const NUser = 'Not a model';

@Table
export class User extends Model<User> {

  @Column
  nickname: string;
}
```

`modelMatch` 函数会匹配全部三个导出的成员
```
user.model UserN -> false
user.model NUser -> false
user.model User  -> true (User 会作为模型被添加)
```

还有一种匹配模型的方式，就是去使用 `export default` 来导出

```TypeScript
export default class User extends Model<User> {}
```

> ⚠️ 当使用路径来添加模型时，一定要记住这些模型会在runtime时被加载，这意味着在开发时和执行时路径有可能会不同。
> 比方说，使用`.ts`扩展名只能和 [ts-node](https://github.com/TypeStrong/ts-node) 一起使用。

### build 和 create
实例化再插入在 sequelize 上是一种将数据保存到数据库的老办法
```typescript
const person = Person.build({name: 'bob', age: 99});
person.save();

Person.create({name: 'bob', age: 99});
```
但 *sequelize-typescript* 提供了一种使用 `new` 关键字创建实例的方法:
```typescript
const person = new Person({name: 'bob', age: 99});
person.save();
```

### find 和 update
查询和更新数据也如同使用原生 sequelize 一样，查阅 sequelize 
[文档](https://docs.sequelizejs.com/manual/tutorial/models-usage.html) 来获取更多细节.
```typescript
Person
 .findOne()
 .then(person => {

     person.age = 100;
     return person.save();
 });

Person
 .update({
   name: 'bobby'
 }, {where: {id: 1}})
 .then(() => {

 });
```

## 模型关联
关联关系可以由模型中的 `@HasMany`, `@HasOne`, `@BelongsTo`, `@BelongsToMany` 和 `@ForeignKey` 注解直接阐明。

### 一对多
```typescript
@Table
class Player extends Model<Player> {

  @Column
  name: string;

  @Column
  num: number;

  @ForeignKey(() => Team)
  @Column
  teamId: number;

  @BelongsTo(() => Team)
  team: Team;
}

@Table
class Team extends Model<Team> {

  @Column
  name: string;

  @HasMany(() => Player)
  players: Player[];
}
```
就这样, *sequelize-typescript* 把其他所有的事情都帮你做了。 所以当你用 `find` 来检索这个 `team`时，
```typescript

Team
 .findOne({include: [Player]})
 .then(team => {

     team.players.forEach(player => console.log(`Player ${player.name}`));
 })
```
`player` 也同样会被解析 (当给 `find` 传入 `include: Player` 这个选项时)

### 多对多
```typescript
@Table
class Book extends Model<Book> {
  @BelongsToMany(() => Author, () => BookAuthor)
  authors: Author[];
}

@Table
class Author extends Model<Author> {

  @BelongsToMany(() => Book, () => BookAuthor)
  books: Book[];
}

@Table
class BookAuthor extends Model<BookAuthor> {

  @ForeignKey(() => Book)
  @Column
  bookId: number;

  @ForeignKey(() => Author)
  @Column
  authorId: number;
}
```
#### 安全的*跨表实例*访问
为了安全的访问*跨表实例*（例如上方代码中的`BookAuthor`），关联关系需要手动的设置。
`Author`模型就可以这样来设置：

```ts
  @BelongsToMany(() => Book, () => BookAuthor)
  books: Array<Book & {BookAuthor: BookAuthor}>;
```

### 一对一
对于 一对一 的关系，使用`@HasOne(...)`(关联的外键会被创建于另一个模型) 和 `@BelongsTo(...)`(关联的外键会被创建于本模型)

### `@ForeignKey`, `@BelongsTo`, `@HasMany`, `@HasOne`, `@BelongsToMany` API

| 装饰器                                                                                                                        | 描述                                                                                                                                                                                                           |
| ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@ForeignKey(relatedModelGetter: () => typeof Model)`                                                                         | 将相关类的属性标记为 `foreignKey`                                                                                                                                                                              |
| `@BelongsTo(relatedModelGetter: () => typeof Model)`                                                                          | sets `SourceModel.belongsTo(RelatedModel, ...)` while `as` is key of annotated property and `foreignKey` is resolved from source class                                                                         |
| `@BelongsTo(relatedModelGetter: () => typeof Model, foreignKey: string)`                                                      | sets `SourceModel.belongsTo(RelatedModel, ...)` while `as` is key of annotated property and `foreignKey` is explicitly specified value                                                                         |
| `@BelongsTo(relatedModelGetter: () => typeof Model, options: AssociationOptionsBelongsTo)`                                    | sets `SourceModel.belongsTo(RelatedModel, ...)` while `as` is key of annotated property and `options` are additional association options                                                                       |
| `@HasMany(relatedModelGetter: () => typeof Model)`                                                                            | sets `SourceModel.hasMany(RelatedModel, ...)` while `as` is key of annotated property and `foreignKey` is resolved from target related class                                                                   |
| `@HasMany(relatedModelGetter: () => typeof Model, foreignKey: string)`                                                        | sets `SourceModel.hasMany(RelatedModel, ...)` while `as` is key of annotated property and `foreignKey` is explicitly specified value                                                                           |
| `@HasMany(relatedModelGetter: () => typeof Model, options: AssociationOptionsHasMany)`                                        | sets `SourceModel.hasMany(RelatedModel, ...)` while `as` is key of annotated property and `options` are additional association options                                                                         |
| `@HasOne(relatedModelGetter: () => typeof Model)`                                                                             | sets `SourceModel.hasOne(RelatedModel, ...)` while `as` is key of annotated property and `foreignKey` is resolved from target related class                                                                    |
| `@HasOne(relatedModelGetter: () => typeof Model, foreignKey: string)`                                                         | sets `SourceModel.hasOne(RelatedModel, ...)` while `as` is key of annotated property and `foreignKey` is explicitly specified value                                                                            |
| `@HasOne(relatedModelGetter: () => typeof Model, options: AssociationOptionsHasOne)`                                          | sets `SourceModel.hasOne(RelatedModel, ...)` while `as` is key of annotated property and `options` are additional association options                                                                          |
| `@BelongsToMany(relatedModelGetter: () => typeof Model, through: (() => typeof Model))`                                       | sets `SourceModel.belongsToMany(RelatedModel, {through: ThroughModel, ...})` while `as` is key of annotated property and `foreignKey`/`otherKey` is resolved from through class                                |
| `@BelongsToMany(relatedModelGetter: () => typeof Model, through: (() => typeof Model), foreignKey: string)`                   | sets `SourceModel.belongsToMany(RelatedModel, {through: ThroughModel, ...})` while `as` is key of annotated property, `foreignKey` is explicitly specified value and `otherKey` is resolved from through class |
| `@BelongsToMany(relatedModelGetter: () => typeof Model, through: (() => typeof Model), foreignKey: string, otherKey: string)` | sets `SourceModel.belongsToMany(RelatedModel, {through: ThroughModel, ...})` while `as` is key of annotated property and `foreignKey`/`otherKey` are explicitly specified values                               |
| `@BelongsToMany(relatedModelGetter: () => typeof Model, through: string, foreignKey: string, otherKey: string)`               | sets `SourceModel.belongsToMany(RelatedModel, {through: throughString, ...})` while `as` is key of annotated property and `foreignKey`/`otherKey` are explicitly specified values                              |
| `@BelongsToMany(relatedModelGetter: () => typeof Model, options: AssociationOptionsBelongsToMany)`                            | sets `SourceModel.belongsToMany(RelatedModel, {through: throughString, ...})` while `as` is key of annotated property and `options` are additional association values, including `foreignKey` and `otherKey`.  |


请注意当使用 `AssociationOptions` 时，某些属性会在关联关系建立时被 `reflection metadata` 或 传入的属性所覆盖。
For example, `as` will always be the annotated property's name, and `through` will be the explicitly stated value.

### 同一模型的多种关联
*sequelize-typescript* 通过识别响应的类的引用来解析外键。
所以说如果你像下方代码一样来建立多重关联：
```typescript
@Table
class Book extends Model<Book> {

  @ForeignKey(() => Person)
  @Column
  authorId: number;

  @BelongsTo(() => Person)
  author: Person;

  @ForeignKey(() => Person)
  @Column
  proofreaderId: number;

  @BelongsTo(() => Person)
  proofreader: Person;
}

@Table
class Person extends Model<Person> {

  @HasMany(() => Book)
  writtenBooks: Book[];

  @HasMany(() => Book)
  proofedBooks: Book[];
}
```
*sequelize-typescript* 不知道去使用哪一个外键去建立对应关联，所以你需要像下方代码一样去明确外键：
```typescript

  // in class "Books":
  @BelongsTo(() => Person, 'authorId')
  author: Person;

  @BelongsTo(() => Person, 'proofreaderId')
  proofreader: Person;

  // in class "Person":
  @HasMany(() => Book, 'authorId')
  writtenBooks: Book[];

  @HasMany(() => Book, 'proofreaderId')
  proofedBooks: Book[];
```

### 安全的使用自动生成函数 Type safe usage of auto generated functions
当模型关联创建时，sequelize 会在对应的模型上绑定一些方法。所以在`modelA` 和 `modelB` 间创建一对多的关系时，`modelA` 的实例将会存在 `getModelBs`, `setModelBs`, `addModelB`, `removeModelB`, `hasModelB` 这些方法。这些同样可以在 *sequelize-typescript* 中使用，但在你访问 `getModelB`, `setModelB` 或者
`addModelB` 时，Typescript 没办法正常的识别他们。为了在使用 TypeScript 时更加友好，*sequelize-typescript* 中的 `Model.prototype` 具有 `$set`, `$get`, `$add` 这几个函数。
```typescript
@Table
class ModelA extends Model<ModelA> {

  @HasMany(() => ModelB)
  bs: ModelB[];
}

@Table
class ModelB extends Model<ModelB> {

  @BelongsTo(() => ModelA)
  a: ModelA;
}
```
如果要使用这些方法，请将该关联对应的键值作为第一个参数传入:
```typescript
const modelA = new ModelA();

modelA.$set('bs', [ /* instance */]).then( /* ... */);
modelA.$add('b', /* instance */).then( /* ... */);
modelA.$get('bs').then( /* ... */);
modelA.$count('bs').then( /* ... */);
modelA.$has('bs').then( /* ... */);
modelA.$remove('bs', /* instance */ ).then( /* ... */);
modelA.$create('bs', /* value */ ).then( /* ... */);
```

## 索引

### `@Index`
`@Index` 注解能在不传入任何参数的情况下使用。
```typescript
@Table
class Person extends Model<Person> {
  @Index // 用默认名称定义一个索引
  @Column
  name: string;

  @Index // 定义另一个索引
  @Column
  birthday: Date;
}
```

通过传递对象来指定索引的相关设置 (查看 [索引定义选项](https://sequelize.org/v5/manual/models-definition.html#indexes) 文档):
```typescript
@Table
class Person extends Model<Person> {
  @Index('my-index') // 给 name 和 birthday 定义多列索引
  @Column
  name: string;

  @Index('my-index') // 把 birthday 字段添加到 my-index 索引
  @Column
  birthday: Date;

  @Index({
    // index options
    name: 'job-index',
    parser: 'my-parser',
    type: 'UNIQUE',
    unique: true,
    where: { isEmployee: true },
    concurrently: true,
    using: 'BTREE',
    operator: 'text_pattern_ops',
    prefix: 'test-',
    // index field options
    length: 10,
    order: 'ASC',
    collate: 'NOCASE',
  })
  @Column
  jobTitle: string;

  @Column
  isEmployee: boolean;
}
```

#### Index API

| Decorator                                | Description                                                                                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `@Index`                                 | 把被装饰的字段名作为新索引添加到 `options.indexes`                                               |
| `@Index(name: string)`                   | 添加新的索引或将字段添加到指定名称的现有索引中                                                   |
| `@Table(options: IndexDecoratorOptions)` | 通过 [配置](https://sequelize.org/v5/manual/models-definition.html#indexes) 来设置索引和索引字段 |

### `createIndexDecorator()`
`createIndexDecorator()` 函数能用来创建一个自定义装饰器，这个装饰器支持通过指定的属性来设置索引，传入装饰器的参数将被用来配置索引字段。
```typescript
const SomeIndex = createIndexDecorator();
const JobIndex = createIndexDecorator({
  // index options
  name: 'job-index',
  parser: 'my-parser',
  type: 'UNIQUE',
  unique: true,
  where: { isEmployee: true },
  concurrently: true,
  using: 'BTREE',
  operator: 'text_pattern_ops',
  prefix: 'test-',
});

@Table
class Person extends Model<Person> {
  @SomeIndex // Add name to SomeIndex
  @Column
  name: string;

  @SomeIndex // Add birthday to SomeIndex
  @Column
  birthday: Date;

  @JobIndex({
    // index field options
    length: 10,
    order: 'ASC',
    collate: 'NOCASE',
  })
  @Column
  jobTitle: string;

  @Column
  isEmployee: boolean;
}
```

## Repository 模式
repository 模式可以把一些类似 `find`, `create` 等的静态方法从模型定义中分离出来，并且赋予了模型可以和多个 sequelize 实例一起使用的能力。

### 怎样启动 repository 模式?
通过设置 `repositoryMode` 字段来开启 repository 模式:
```typescript
const sequelize = new Sequelize({
  repositoryMode: true,
  ...,
});
```
### 如何使用 repository 模式?
通过操作 repository 来进行创建实例和查询的操作：
```typescript
const userRepository = sequelize.getRepository(User);

const luke = await userRepository.create({name: 'Luke Skywalker'});
const luke = await userRepository.findOne({where: {name: 'luke'}});
```
### 怎样在 repository 模式下使用关联?
目前需要在使用 repository 时通过配置 `include` 参数来进行检索或创建相关数据

```typescript
const userRepository = sequelize.getRepository(User);
const addressRepository = sequelize.getRepository(Address);

userRepository.find({include: [addressRepository]});
userRepository.create({name: 'Bear'}, {include: [addressRepository]});
```
> ⚠️ 未来会有变动：开发者应该去引用模型类而不是 repository

### repository 模式的限制
当同时使用 `@Scope` 注解和 repository 模式时，嵌套作用域和 `include` 通常将不起作用
```typescript
@Scopes(() => ({
  // includes
  withAddress: {
    include: [() => Address],
  },
  // nested scopes
  withAddressIncludingLatLng: {
    include: [() => Address.scope('withLatLng')],
  }
}))
@Table
class User extends Model<User> {}
```
> ⚠️ 未来会有变动: 单一的 `include` 将被实现

## 模型校验
校验项可以在 `@Column` 注解中设置，不过如果你更偏爱单独的装饰器来校验的话，你可以通过添加校验装饰器来实现：
比方说 `validate.isEmail=true` 写作 `@IsEmail`、`validate.equals='value'` 写作 `@Equals('value')` 以此类推。
需要注意的是，一个期望布尔值的校验器会被编译为一个不接收参数的注解。

查阅 sequelize [文档](https://sequelize.org/v5/manual/models-definition.html#validations)
来了解所有的校验规则。

### 异常处理

下面的校验器不能被简单的从一个 sequelize 校验器转换成修饰器：


| 校验器                          | 注解                                                                                                                       |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `validate.len=[number, number]` | `@Length({max?: number, min?: number})`                                                                                    |
| `validate[customName: string]`  | 自定义校验器可使用 `@Is(...)` 注解: `@Is('custom', (value) => { /* ... */})` 或 `@Is(function custom(value) { /* ... */})` |
### Example
```typescript
const HEX_REGEX = /^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$/;

@Table
export class Shoe extends Model<Shoe> {

  @IsUUID(4)
  @PrimaryKey
  @Column
  id: string;

  @Equals('lala')
  @Column
  readonly key: string;

  @Contains('Special')
  @Column
  special: string;

  @Length({min: 3, max: 15})
  @Column
  brand: string;

  @IsUrl
  @Column
  brandUrl: string;

  @Is('HexColor', (value) => {
    if (!HEX_REGEX.test(value)) {
      throw new Error(`"${value}" is not a hex color value.`);
    }
  })
  @Column
  primaryColor: string;

  @Is(function hexColor(value: string): void {
    if (!HEX_REGEX.test(value)) {
      throw new Error(`"${value}" is not a hex color value.`);
    }
  })
  @Column
  secondaryColor: string;

  @Is(HEX_REGEX)
  @Column
  tertiaryColor: string;

  @IsDate
  @IsBefore('2017-02-27')
  @Column
  producedAt: Date;
}
```

## 作用域
作用域同样能使用装饰器来定义。作用域配置同原生 sequelize (查阅 sequelize [文档](https://docs.sequelizejs.com/manual/tutorial/scopes.html) 来了解作用域用法)

### `@DefaultScope` 和 `@Scopes`
```typescript
@DefaultScope(() => ({
  attributes: ['id', 'primaryColor', 'secondaryColor', 'producedAt']
}))
@Scopes(() => ({
  full: {
    include: [Manufacturer]
  },
  yellow: {
    where: {primaryColor: 'yellow'}
  }
}))
@Table
export class ShoeWithScopes extends Model<ShoeWithScopes> {

  @Column
  readonly secretKey: string;

  @Column
  primaryColor: string;

  @Column
  secondaryColor: string;

  @Column
  producedAt: Date;

  @ForeignKey(() => Manufacturer)
  @Column
  manufacturerId: number;

  @BelongsTo(() => Manufacturer)
  manufacturer: Manufacturer;
}
```

## Hooks - 钩子
Hooks(也称为生命周期事件)是在执行 sequelize 中的调用之前和之后调用的函数，所有模型上钩子都被支持。 查阅 [相关的单元测试](https://github.com/RobinBuschmann/sequelize-typescript/blob/master/test/models/Hook.ts) 来获得总结。

每一个钩子都必须为一个 `static` 方法，多个钩子可以被关联到一个方法，同时你也可以给一个钩子定义多个方法。

方法的函数名不能和钩子的函数名相同（比方说， `@BeforeCreate` 钩子不能被命名为 `beforeCreate`，因为这个名称的方法被 Sequelize 预定义过了）

```typescript
@Table
export class Person extends Model<Person> {
  @Column
  name: string;

  @BeforeUpdate
  @BeforeCreate
  static makeUpperCase(instance: Person) {
    // this will be called when an instance is created or updated
    instance.name = instance.name.toLocaleUpperCase();
  }

  @BeforeCreate
  static addUnicorn(instance: Person) {
    // this will also be called when an instance is created
    instance.name += ' 🦄';
  }
}
```

## 为何使用 `() => Model`?
既然 `@ForeignKey(Model)` 可读性更高, 那么为什么要使用 `@ForeignKey(() => Model)` 呢? 当发生循环依赖时，`Model` 会为 `undefined` (通常 node 会为你解决)。当他被传递给 `@ForeignKey`时，我们通过使用一个返回了实际模型的函数来避免这个问题的发生。

## 风格推荐和限制

###  每个模型对应一个 Sequelize 实例 (除 repository 模式)
除非你使用 [repository 模式](#repository-模式), 你不能向连接配置不同的 Sequelize 实例添加一个相同模型。所以，一个模型仅适用于一个 Sequelize 连接。

### 每个文件存放一个模型类

这不仅仅是一个架构设计的好习惯，并且有利于命令的执行。由于 TypeScript 根据 `emitDecoratorMetadata` 这个编译选项创建了 `__metadata("design:type", SomeModel)`，在某些情况下， `SomeModel` 有可能没有被定义(不是 undefined!)，并且会出一个 `ReferenceError`。当把 `SomeModel` 放进一个单独的文件中，就相当于 `__metadata("design:type", SomeModel_1.SomeModel)`，这样的话就不会报错。

### 文件压缩
如果你有压缩代码的需求，为了 `@Table` 注解能正常使用，你需要设置在 `@DefineOptions` 中设置 `tableName` 和 `modelName`。sequelize-typescript 会使用 `tableName` 和 `modelName` 作为类的默认名称。当代码被压缩后，类名将不再是最初定义的那个了。（就像 `class User` 会变成 `class b`）


# 贡献

如果你想对这个开源项目贡献出自己的一份力量，你可以：
- 提交 issue 或者 参与其他 issue 的讨论
- Fork 这个项目来提交 PR
- 丰富 [Sequelize 的类型声明](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/sequelize).
- 提出任何有建设性的建议

为了提交一个PR，请:
- 创建一个新的分支.
- 完整的跑一遍测试流程 (`npm install && npm run build && npm run cover`) 并且确保你的提交能顺利通过测试。
- 通过 commit 中的 message、详细的 PR 说明，必要时的代码注释等等来记录你的工作

如果你想丰富 Sequelize 的类型声明，请去 [类型定义仓库](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/sequelize)，向 [sequelize](https://github.com/sequelize/sequelize) 提交一个 PR 也是一个好方法，那样子 Sequlize 就能维护自己的类型声明，不过这样也许比更新微软的类型声明库要更难。TypeScript 团队正在尝试去慢慢的鼓励 npm 包维护者去维护自己的类型声明，但 TypeScript 还是会有专门的人员去维护 DT repo，审核PR，来保证高质量。

# 改进文档

如果你发现文档中有哪些翻译的晦涩，或者不合理的地方，欢迎通过 issue 或者 PR 的方式来帮助改进

**请牢记， `sequelize-typescript` 不提供 `sequelize` 的类型声明** - 这是不相关联的事情
`sequelize-typescript` 参数中的许多类型都引用或扩展都是 sequelize 已经拥有的内容。