# Concept

Models are the essence of Sequelize. A model is an abstraction that represents a table in your database. In Sequelize, it is a class that extends [Model](https://sequelize.org/api/v6/class/src/model.js~Model.html).

The model tells Sequelize several things about the entity it represents, such as the name of the table in the database and which columns it has (and their data types).

A model in Sequelize has a name. This name does not have to be the same name of the table it represents in the database. Usually, models have singular names (such as **User**) while tables have pluralized names (such as **Users**), although this is fully configurable.


# Model Definition

Models can be defined in two equivalent ways in Sequelize:

1. Calling [sequelize.define(modelName, attributes, options)](https://sequelize.org/api/v6/class/src/sequelize.js~Sequelize.html#instance-method-define).

2. Extending [Model](https://sequelize.org/api/v6/class/src/model.js~Model.html) and calling [init(attributes, options)](https://sequelize.org/api/v6/class/src/model.js~Model.html#static-method-init)


After a model is defined, it is available within **sequelize.models** by its model name.

To learn with an example, we will consider that we want to create a model to represent users, which have a **firstName** and a **lastName**. We want our model to be called **User**, and the table it represents is called **Users** in the database.

Both ways to define this moidel are shown below. After being defined, we can access our modle with **sequelize.models.User**.


## Using sequelize.define:

```js
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory');

const User = sequelize.define('User', {
  // Model attributes are defined here
  firstName: {
    type: DataTypes.STRING,
    allowNull: false
  },
  lastName: {
    type: DataTypes.STRING
    // allowNull defaults to true
  },
  {
    // Other model options go here
});


// sequelize.define also returns the model
console.log(User === sequelize.models.User); // true
```


## Extending Model

```js
const { Sequelize, DataTypes, Model } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory:');

class User extends Model {}

User.init({
  // Model attributes are defined here
  firstName: {
    type: DataTypes.STRING,
    allowNull: false
  },
  lastName: {
    type: DataTypes.STRING
    // allowNull defaults to true
  }, {
    // Other model options go here
    sequelize, // We need to pass the connection instance
    modelName: 'User' // We need to choose the model name
});
```

Internally, **sequelize.define** calls **Model.init**, so both appraoches are essentially equivalent.


### Caveat with Public Class Fields

Adding a [Public Class Field](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Public_class_fields) with the same name as one of the model's attribute is going to cause issues. Sequelize adds a getter & a setter for each attribute defined through **Model.init**. Adding a Public Class Field will shadow those getters and setters, blocking access to the model's actual data.

```js
// Invalid
class User extends Model {
  id; // this field will shadow sequelize's getter & setter. It should be removed.
  otherPublicField; // this field does not shadow anything. It is fine.
}

User.init({
  id: {
    type: DataTypes.INTEGER,
    autoIncrement: true,
    primaryKey: true
  }
}, { sequelize });

const user = new User({ id: 1 });
user.id; // undefined
```

```js
// Valid
class User extends Model {
  otherPublicFields;
}

User.init({
  id: {
    type: DataTypes.INTEGER,
    autoIncrement: true,
    primaryKey: true
  }
}, { sequelize });

const user = new user({ id: 1 });
user.id; // 1
```

In TypeScript you can add typing information without adding an actual public class field by using the **declare** keyword.

```js
// Valid
class User extends Model {
  declare id: number; // this is ok! The 'declare' keyword ensures this field will not be emitted by TypeScript.
}

User.init({
  id: {
    type: DataTypes.INTEGER,
    autoIncrement: true,
    primaryKey: true
  }
}, { sequelize });

const user = new User({ id: 1 });
user.id;  // 1
```


# Table name inference

Observe that, in both methods above, the table name (**Users**) was never explicitly defined. However, the model name was given (**User**).

By default, when the table name is not given, Sequelize automatically pluralizes the model name and uses that as the table name. The pluralization is done under the hood by a library called [inflection](https://www.npmjs.com/package/inflection), so that irregular plurals (such as **person -> people**) are computed correctly.


Of course, this behavior is easily configurable.


## Enforcing the table name to be equal to the model name

You can stop the auto-pluralization performed by Sequelize using the **freezeTableName: true** option. This way, Sequelize will infer the table name to be equal to the model name, without any modifications:

```js
sequelize.define('User', {
  // ... (attributes)
}, {
  freezeTableName: true
});
```

The example above will create a model named **User** pointing to a table also named **User**.

This behavior can also be defined globally for the sequelize instance, when it is created:

```js
const sequelize = new Sequelize('sqlite::memory:', {
  define: {
    freezeTableName: true
  }
});
```

This way, all tables will use the same name as the model name.


## Providing the table name directly

You can simply tell Sequelize the name of the table directly as well:

```js
sequelize.define('User', {
  // ... (attributes)
}, {
  tableName: 'Employees'
});\
```



# Model synchronization

When you define a model, you're telling Sequelize a few things about its table in the database. However, what if the table actually doesn't even exist in the database? What if it exists, but it has different columns, less columns, or any other difference?

This is where model synchroniztion comes in. A model can be synchronized with the database by calling **model.sync(options)**, an asynchronous function (that returns a Promise). With this call, Sequelize will automatically perform an SQL query to the database. Note that this changes only the table in the database, not the model in the JavaScript side.

* **User.sync()** - This creates the table if it doesn't exist (and does nothing if it already exists)

* **User.sync({ force: true })** - This creates the table, dropping it first if it already existed

* **User.sync({ alter: true })** - This checks what is the current state of the table in the database (which columns it has, what are their data types, etc), and then performs the necessary changes in the table to make it match the model.

Example:

```js
await User.sync({ force: true });
console.log("The table for the User model was just (re)created!");
```

## Synchronizing all models at once

You can use **sequelize.sync()** to automatically synchronize all models. Example:

```js
await sequelize.sync({ force: true });
console.log("All models were synchronized successfully.");
```

## Dropping tables

To drop the table related to a model:

```js
await User.drop();
console.log("User table dropped!");
```

To drop all tables:

```js
await sequelize.drop();
console.log("All tables dropped!");
```

## Database safety check

As shown above, the **sync** and **drop** operations are destructive. Sequelize accepts a **match** option as an additional safety check, which receives a RegExp:

```js
// This will run .sync() only if database name ends with '_test'
sequelize.sync({ force: true, match: /_test$/ });
```


## Synchronization in production

As shown above, **sync({ force: true })** and **sync({ alter: true })** can be destructive operations. Therefore, they are not recommended for production-level software. Instead, synchronization should be done with the advanced concept of Migrations, with the help of the Sequelize CLI.


# Timestamps

By default, Sequelize automatically adds the fields **createdAt** and **updatedAt** to every model, using the data type **DataTypes.DATE**. Those fields are automatically managed as well - whenever you use Sequelize to create or update something, those fields will be set correctly. The **createdAt** field will contain the timestamp representing the moment of creation, and the **updatedAt** will contain the timestamp of the latest update.

**Note:** This is done in the Sequelize level (i.e. not done with SQL triggers). This means that direct SQL queries (for example queries performed without Sequelize by any other means) will not cause these fields to be updated automatically.

This behavior can be disabled for a model with the **timestamps: false** option:

```ts
sequelize.define('User', {
  // ... (attributes)
}, {
  timestamps: false
});
```

It is also possible to enable only one of **createdAt**/**updatedAt**, and to provide a custom name for these columns:

```ts
class Foo extends Model {}
Foo.init({ /* attributes */ }, {
  sequelize,

  // don't forget to enable timestamps!
  timestamps: true,

  // I don't want createdAt
  createdAt: false,

  // I want updatedAt to actually be called updateTimestamp
  updatedAt: 'updateTimestamp'
});
```


# 