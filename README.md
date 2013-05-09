[![Build Status](https://travis-ci.org/biggora/caminte.png?branch=master)](https://travis-ci.org/biggora/caminte)
## About CaminteJS

CaminteJS is cross-db ORM for nodejs, providing common interface to access
most popular database formats.

#### CaminteJS adapters:
    mongoose, mysql, sqlite3, riak, postgres, couchdb, mongodb, redis, neo4j, firebird, nano


## Installation

First install [node.js](http://nodejs.org/). Then:

    $ npm install caminte

## Overview

### Connecting to DB

First, we need to define a connection.

```javascript
    var caminte = require('caminte'),
    Schema = caminte.Schema,
    db = {
         driver     : "mysql",
         host       : "localhost",
         port       : "3306",
         username   : "test",
         password   : "test",
         database   : "test"
    };

    var schema = new Schema(db.driver, db);
```

### Defining a Model

Models are defined through the `Schema` interface.

```javascript
// define models
var Post = schema.define('Post', {
    title:     { type: String, length: 255 },
    content:   { type: Schema.Text },
    date:      { type: Date,    default: Date.now },
    published: { type: Boolean, default: false, index: true }
});

// simplier way to describe model
var User = schema.define('User', {
    name:         String,
    bio:          Schema.Text,
    approved:     Boolean,
    joinedAt:     Date,
    age:          Number
});
```

### Accessing a Model

```javascript
// models also accessible in schema:
schema.models.User;
schema.models.Post;
```

### Setup Relationships

```javascript
User.hasMany(Post,   {as: 'posts',  foreignKey: 'userId'});
// creates instance methods:
// user.posts(conds)
// user.posts.build(data) // like new Post({userId: user.id});
// user.posts.create(data) // build and save

Post.belongsTo(User, {as: 'author', foreignKey: 'userId'});
// creates instance methods:
// post.author(callback) -- getter when called with function
// post.author() -- sync getter when called without params
// post.author(user) -- setter when called with object

// work with models:
var user = new User;
user.save(function (err) {
    var post = user.posts.build({title: 'Hello world'});
    post.save(console.log);
});
```

### Setup Validations

```javascript
User.validatesPresenceOf('name', 'email')
User.validatesLengthOf('password', {min: 5, message: {min: 'Password is too short'}});
User.validatesInclusionOf('gender', {in: ['male', 'female']});
User.validatesExclusionOf('domain', {in: ['www', 'billing', 'admin']});
User.validatesNumericalityOf('age', {int: true});
User.validatesUniquenessOf('email', {message: 'email is not unique'});

user.isValid(function (valid) {
    if (!valid) {
        user.errors // hash of errors {attr: [errmessage, errmessage, ...], attr: ...}
    }
})
```

### Common API methods

#### Just instantiate model

```javascript
   var post = new Post();
```

#### create(callback)

Save model (of course async)

```javascript
Post.create(function(err){
   // your code here
});
// same as new Post({userId: user.id});
user.posts.build
// save as Post.create({userId: user.id}, function(err){
   // your code here
});
user.posts.create(function(err){
   // your code here
});
```

#### all(params, callback)

```javascript
// all posts
Post.all(function(err, posts){
   // your code here
});
// all posts by user
Post.all({where: {userId: user.id}, order: 'id', limit: 10, skip: 20}, function(err, posts){
   // your code here
});
// the same as prev
user.posts(function(err, posts){
   // your code here
})
```

#### findOne(params, callback)

Get one latest post

```javascript
Post.findOne({where: {published: true}, order: 'date DESC'}, function(err, post){
   // your code here
});
```

#### findById(id, callback)

Find instance by id

```javascript
User.findById(1, function(err, post){
   // your code here
})
```

#### count(params, callback)

Count instances

```javascript
// count posts by user
User.count({where: {userId: user.id}}, function(err, count){
   // your code here
});
```

#### destroy(callback)

Destroy instance

```javascript
user.destroy(function(err){
   // your code here
});
```

#### destroyAll(callback)

Destroy all instances

```javascript
User.destroyAll(function(err){
   // your code here
});
```

### Queries

#### API methods

* [where](#where)
* [gt](#gt)
* [gte](#gte)
* [lt](#lt)
* [lte](#lte)
* [ne](#ne)
* [in](#in)
* [nin](#nin)
* [regex](#regex)
* [like](#like)
* [nlike](#nlike)

#### Example Queries
```javascript
var user = User.find();
user.where('active', 1);
user.order('id DESC');
user.run({}, function(err, users) {
   // your code here
});
```

#### #where
<a name="where"></a>
```javascript
var Query = User.find();
Query.where('userId', user.id);
Query.run({}, function(err, count){
   // your code here
});
// the same as prev
User.find({where: {userId: user.id}}, function(err, count){
   // your code here
});
```

#### #gt

Specifies a greater than expression.

<a name="gt"></a>
```javascript
User.gt('userId', 100);
// the same as prev
User.find({
      where: {
         userId: {
              gt : 100
         }
      }
    }}, function(err, count){
   // your code here
});
```

#### #gte

Specifies a greater than or equal to expression.

<a name="gte"></a>
```javascript
User.gte('userId', 100);
// the same as prev
User.find({
      where: {
         userId: {
              gte : 100
         }
      }
    }}, function(err, count){
   // your code here
});
```

### Define any Custom Method

```javascript
User.prototype.getNameAndAge = function () {
    return this.name + ', ' + this.age;
};
```

### Middleware (callbacks)

The following callbacks supported:

    - afterInitialize
    - beforeCreate
    - afterCreate
    - beforeSave
    - afterSave
    - beforeUpdate
    - afterUpdate
    - beforeDestroy
    - afterDestroy
    - beforeValidation
    - afterValidation


```javascript
User.afterUpdate = function (next) {
    this.updated_ts = new Date();
    this.save();
    next();
};
```

Each callback is class method of the model, it should accept single argument: `next`, this is callback which
should be called after end of the hook. Except `afterInitialize` because this method is syncronous (called after `new Model`).


### Automigrate
required only for mysql NOTE: it will drop User and Post tables

```javascript
schema.automigrate();
```

## Object lifecycle:

```javascript
var user = new User;
// afterInitialize
user.save(callback);
// beforeValidation
// afterValidation
// beforeSave
// beforeCreate
// afterCreate
// afterSave
// callback
user.updateAttribute('email', 'email@example.com', callback);
// beforeValidation
// afterValidation
// beforeUpdate
// afterUpdate
// callback
user.destroy(callback);
// beforeDestroy
// afterDestroy
// callback
User.create(data, callback);
// beforeValidate
// afterValidate
// beforeCreate
// afterCreate
// callback
```

Read the tests for usage examples: ./test/common_test.js
Validations: ./test/validations_test.js


## Your own database adapter

To use custom adapter, pass it's package name as first argument to `Schema` constructor:

    mySchema = new Schema('couch-db-adapter', {host:.., port:...});

Make sure, your adapter can be required (just put it into ./node_modules):

    require('couch-db-adapter');

## Running tests

To run all tests (requires all databases):

    npm test

If you run this line, of course it will fall, because it requres different databases to be up and running,
but you can use js-memory-engine out of box! Specify ONLY env var:

    ONLY=memory nodeunit test/common_test.js

of course, if you have redis running, you can run

    ONLY=redis nodeunit test/common_test.js

## Package structure

Now all common logic described in `./lib/*.js`, and database-specific stuff in `./lib/adapters/*.js`. It's super-tiny, right?

## Contributing

If you have found a bug please write unit test, and make sure all other tests still pass before pushing code to repo.


## Recommend extensions

- [TrinteJS](http://www.trintejs.com/) - Javascrpt MVC Framework for Node.JS
- [express-useragent](https://github.com/biggora/express-useragent) - Exposing user-agent for NodeJS

## License

(The MIT License)

Copyright (c) 2011 by Anatoliy Chakkaev <mail [åt] anatoliy [døt] in>

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


## Resources

- Follow [@biggora](https://twitter.com/#!/biggora) on Twitter for updates.
- Report issues on the [github issues](https://github.com/biggora/caminte/issues) page.