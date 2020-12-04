# sequelize-typescript 中文文档
提供给 sequelize (v5) 使用的装饰器和一些其他功能.

### 施工中 🚧

 - [安装](#安装)
 - [升级到 `sequelize-typescript@1`](#升级到`sequelize-typescript@1`)
 - [模型定义](#模型定义)
   - [`@Table` API](#table-api)
   - [`@Column` API](#column-api)
 - [使用](#使用)
   - [配置](#配置)
   - [globs](#globs)
   - [模型路径解析](#模型路径解析)
 - [模型关联](#模型关联)
   - [一对多](#一对多)
   - [多对多](#多对多)
   - [一对一](#一对一)
   - [`@ForeignKey`, `@BelongsTo`, `@HasMany`, `@HasOne`, `@BelongsToMany` API](#foreignkey-belongsto-hasmany-hasone-belongstomany-api)
   - [生成 getter 和 setter](#type-safe-usage-of-auto-generated-functions)
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
 - [Hooks](#hooks)
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
`sequelize-typescript@1` only works with `sequelize@5>=`.
For `sequelize@4` & `sequelize@3` use `sequelize-typescript@0.6`.

### 破坏性更新
`sequelize@5` 的破坏性更新同样会影响 `sequelize-typescript@1`.
查看 [升级到 V5](https://github.com/sequelize/sequelize/blob/master/docs/upgrade-to-v5.md) 详情.

#### 官方 TypeScript 支持
sequelize-typescript 目前使用 sequlize 官方的 TypeScript 定义
(点击 [查看](https://github.com/sequelize/sequelize/blob/master/docs/upgrade-to-v5.md#typescript-support)).
请注意以下事项：
- 绝大部分之前版本 sequelize-typescript 所提供的 interface 被官方所替代
- 不再使用`@types/sequelize`
- `@types/bluebird` 不再是一个显示依赖
- 官方类型声明没 sequelize-typescript 之前版本的严格

#### Sequelize 选项
- `SequelizeConfig` 重命名为 `SequelizeOptions`
- `modelPaths` 重命名为 `models`

#### 作用域 选项
`@Scopes` 和 `@DefaultScope` 装饰器目前接收匿名函数作为选项
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
`@Column` 注解能在不传入任何参数下使用。 传入参数来让 js 类型可以被自行推断是相当有必要的(查看 [类型推断](#type-inference) 细节).
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
### globs
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
`player` 也同样会被解析 (当给 `find` 传入 `include: Player` 这个选项)

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
对于 一对一 的关系，使用`@HasOne(...)`(关联的外键存在另一个模型) 和 `@BelongsTo(...)`(关联的外键存在本模型)

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

Note that when using AssociationOptions, certain properties will be overwritten when the association is built, based on reflection metadata or explicit attribute parameters. For example, `as` will always be the annotated property's name, and `through` will be the explicitly stated value.

### 同一模型的多种关联
*sequelize-typescript* resolves the foreign keys by identifying the corresponding class references.
So if you define a model with multiple relations like
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
*sequelize-typescript* cannot know which foreign key to use for which relation. So you have to add the foreign keys
explicitly:
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

### Type safe usage of auto generated functions
With the creation of a relation, sequelize generates some method on the corresponding
models. So when you create a 1:n relation between `ModelA` and `ModelB`, an instance of `ModelA` will
have the functions `getModelBs`, `setModelBs`, `addModelB`, `removeModelB`, `hasModelB`. These functions still exist with *sequelize-typescript*.
But TypeScript wont recognize them and will complain if you try to access `getModelB`, `setModelB` or
`addModelB`. To make TypeScript happy, the `Model.prototype` of *sequelize-typescript* has `$set`, `$get`, `$add`
functions.
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
To use them, pass the property key of the respective relation as the first parameter:
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

## Indexes

### `@Index`
The `@Index` annotation can be used without passing any parameters.
```typescript
@Table
class Person extends Model<Person> {
  @Index // Define an index with default name
  @Column
  name: string;

  @Index // Define another index
  @Column
  birthday: Date;
}
```

To specify index and index field options, use
an object literal (see [indexes define option](https://sequelize.org/v5/manual/models-definition.html#indexes)):
```typescript
@Table
class Person extends Model<Person> {
  @Index('my-index') // Define a multi-field index on name and birthday
  @Column
  name: string;

  @Index('my-index') // Add birthday as the second field to my-index
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

| Decorator                                | Description                                                                                               |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `@Index`                                 | adds new index on decorated field to `options.indexes`                                                    |
| `@Index(name: string)`                   | adds new index or adds the field to an existing index with specified name                                 |
| `@Table(options: IndexDecoratorOptions)` | sets both index and index field [options](https://sequelize.org/v5/manual/models-definition.html#indexes) |

### `createIndexDecorator()`
The `createIndexDecorator()` function can be used to create a decorator for an index with options specified with an object literal supplied as the argument. Fields are added to the index by decorating properties.
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

## Repository mode
The repository mode makes it possible to separate static operations like `find`, `create`, ... from model definitions.
It also empowers models so that they can be used with multiple sequelize instances.

### How to enable repository mode?
Enable repository mode by setting `repositoryMode` flag:
```typescript
const sequelize = new Sequelize({
  repositoryMode: true,
  ...,
});
```
### How to use repository mode?
Retrieve repository to create instances or perform search operations:
```typescript
const userRepository = sequelize.getRepository(User);

const luke = await userRepository.create({name: 'Luke Skywalker'});
const luke = await userRepository.findOne({where: {name: 'luke'}});
```
### How to use associations with repository mode?
For now one need to use the repositories within the include options in order to retrieve or create related data:
```typescript
const userRepository = sequelize.getRepository(User);
const addressRepository = sequelize.getRepository(Address);

userRepository.find({include: [addressRepository]});
userRepository.create({name: 'Bear'}, {include: [addressRepository]});
```
> ⚠️ This will change in the future: One will be able to refer the model classes instead of the repositories.

### Limitations of repository mode
Nested scopes and includes in general won't work when using `@Scope` annotation together with repository mode like:
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
> ⚠️ This will change in the future: Simple includes will be implemented.

## Model validation
Validation options can be set through the `@Column` annotation, but if you prefer to use separate decorators for
validation instead, you can do so by simply adding the validate options *as* decorators:
So that `validate.isEmail=true` becomes `@IsEmail`, `validate.equals='value'` becomes `@Equals('value')`
and so on. Please notice that a validator that expects a boolean is translated to an annotation without a parameter.

See sequelize [docs](https://sequelize.org/v5/manual/models-definition.html#validations)
for all validators.

### Exceptions
The following validators cannot simply be translated from sequelize validator to an annotation:

| Validator                       | Annotation                                                                                                                                                                  |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `validate.len=[number, number]` | `@Length({max?: number, min?: number})`                                                                                                                                     |
| `validate[customName: string]`  | For custom validators also use the `@Is(...)` annotation: Either `@Is('custom', (value) => { /* ... */})` or with named function `@Is(function custom(value) { /* ... */})` |

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

## Scopes
Scopes can be defined with annotations as well. The scope options are identical to native
sequelize (See sequelize [docs](https://docs.sequelizejs.com/manual/tutorial/scopes.html) for more details)

### `@DefaultScope` and `@Scopes`
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

## Hooks
Hooks can be attached to your models. All Model-level hooks are supported. See [the related unit tests](test/models/Hook.ts) for a summary.

Each hook must be a `static` method. Multiple hooks can be attached to a single method, and you can define multiple methods for a given hook.

The name of the method cannot be the same as the name of the hook (for example, a `@BeforeCreate` hook method cannot be named `beforeCreate`). That’s because Sequelize has pre-defined methods with those names.

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

## Why `() => Model`?
`@ForeignKey(Model)` is much easier to read, so why is `@ForeignKey(() => Model)` so important? When it
comes to circular-dependencies (which are in general solved by node for you) `Model` can be `undefined`
when it gets passed to @ForeignKey. With the usage of a function, which returns the actual model, we prevent
this issue.

## Recommendations and limitations

### One Sequelize instance per model (without repository mode)
Unless you are using the [repository mode](#repository-mode), you won't be able to add one and the same model to multiple
Sequelize instances with differently configured connections. So that one model will only work for one connection.

### One model class per file
This is not only good practice regarding design, but also matters for the order
of execution. Since Typescript creates a `__metadata("design:type", SomeModel)` call due to `emitDecoratorMetadata`
compile option, in some cases `SomeModel` is probably not defined(not undefined!) and would throw a `ReferenceError`.
When putting `SomeModel` in a separate file, it would look like `__metadata("design:type", SomeModel_1.SomeModel)`,
which does not throw an error.

### Minification
If you need to minify your code, you need to set `tableName` and `modelName`
in the `DefineOptions` for `@Table` annotation. sequelize-typescript
uses the class name as default name for `tableName` and `modelName`.
When the code is minified the class name will no longer be the originally
defined one (So that `class User` will become `class b` for example).


# Contributing

To contribute you can:
- Open issues and participate in discussion of other issues.
- Fork the project to open up PR's.
- Update the [types of Sequelize](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/sequelize).
- Anything else constructively helpful.

In order to open a pull request please:
- Create a new branch.
- Run tests locally (`npm install && npm run build && npm run cover`) and ensure your commits don't break the tests.
- Document your work well with commit messages, a good PR description, comments in code when necessary, etc.

In order to update the types for sequelize please go to [the Definitely Typed repo](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/types/sequelize), it would also be a good
idea to open a PR into [sequelize](https://github.com/sequelize/sequelize) so that Sequelize can maintain its own types, but that
might be harder than getting updated types into microsoft's repo. The Typescript team is slowly trying to encourage
npm package maintainers to maintain their own typings, but Microsoft still has dedicated and good people maintaining the DT repo,
accepting PR's and keeping quality high.

**Keep in mind `sequelize-typescript` does not provide typings for `sequelize`** - these are seperate things.
A lot of the types in `sequelize-typescript` augment, refer to, or extend what sequelize already has.