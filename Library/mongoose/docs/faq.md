# 常见问题 {#faq}

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  Why don't my changes to arrays get saved when I update an element directly?

  ```js
  doc.array[3] = 'changed';
  doc.save();
  ```

  <strong class="answer">答：</strong> Mongoose doesn't create getters/setters for array indexes; without them mongoose never gets notified of the change and so doesn't know to persist the new value. The work-around is to use MongooseArray#set available in Mongoose >= 3.2.0.

  ```js
  // 3.2.0
  doc.array.set(3, 'changed');
  doc.save();

  // if running a version less than 3.2.0, you must mark the array modified before saving.
  doc.array[3] = 'changed';
  doc.markModified('array');
  doc.save();
  ```

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  I declared a schema property as unique but I can still save duplicates. What gives?

  <strong class="answer">答：</strong> Mongoose doesn't handle unique on its own: { name: { type: String, unique: true } } is just a shorthand for creating a MongoDB unique index on name. For example, if MongoDB doesn't already have a unique index on name, the below code will not error despite the fact that unique is true.

  ```js
  var schema = new mongoose.Schema({
    name: { type: String, unique: true }
  });
  var Model = db.model('Test', schema);

  Model.create([{ name: 'Val' }, { name: 'Val' }], function(err) {
    console.log(err); // No error, unless index was already built
  });
  ```

  However, if you wait for the index to build using the Model.on('index') event, attempts to save duplicates will correctly error.

  ```js
  var schema = new mongoose.Schema({
    name: { type: String, unique: true }
  });
  var Model = db.model('Test', schema);

  Model.on('index', function(err) { // <-- Wait for model's indexes to finish
    assert.ifError(err);
    Model.create([{ name: 'Val' }, { name: 'Val' }], function(err) {
      console.log(err);
    });
  });

  // Promise based alternative. `init()` returns a promise that resolves
  // when the indexes have finished building successfully. The `init()`
  // function is idempotent, so don't worry about triggering an index rebuild.
  Model.init().then(function() {
    assert.ifError(err);
    Model.create([{ name: 'Val' }, { name: 'Val' }], function(err) {
      console.log(err);
    });
  });
  ```

  MongoDB persists indexes, so you only need to rebuild indexes if you're starting with a fresh database or you ran db.dropDatabase(). In a production environment, you should [create your indexes using the MongoDB shell])(https://docs.mongodb.com/manual/reference/method/db.collection.createIndex/) rather than relying on mongoose to do it for you. The unique option for schemas is convenient for development and documentation, but mongoose is not an index management solution.

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  When I have a nested property in a schema, mongoose adds empty objects by default. Why?

  ```js
  var schema = new mongoose.Schema({
    nested: {
      prop: String
    }
  });
  var Model = db.model('Test', schema);

  // The below prints `{ _id: /* ... */, nested: {} }`, mongoose assigns
  // `nested` to an empty object `{}` by default.
  console.log(new Model());
  ```

  <strong class="answer">答：</strong> This is a performance optimization. These empty objects are not saved to the database, nor are they in the result toObject(), nor do they show up in JSON.stringify() output unless you turn off the minimize option.

  The reason for this behavior is that Mongoose's change detection and getters/setters are based on Object.defineProperty(). In order to support change detection on nested properties without incurring the overhead of running Object.defineProperty() every time a document is created, mongoose defines properties on the Model prototype when the model is compiled. Because mongoose needs to define getters and setters for nested.prop, nested must always be defined as an object on a mongoose document, even if nested is undefined on the underlying POJO.

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  I'm using an arrow function for a virtual, getter/setter, or method and the value of this is wrong.

  <strong class="answer">答：</strong> Arrow functions handle the this keyword much differently than conventional functions. Mongoose getters/setters depend on this to give you access to the document that you're writing to, but this functionality does not work with arrow functions. Do not use arrow functions for mongoose getters/setters unless do not intend to access the document in the getter/setter.

  ```js
  // Do **NOT** use arrow functions as shown below unless you're certain
  // that's what you want. If you're reading this FAQ, odds are you should
  // just be using a conventional function.
  var schema = new mongoose.Schema({
    propWithGetter: {
      type: String,
      get: v => {
        // Will **not** be the doc, do **not** use arrow functions for getters/setters
        console.log(this);
        return v;
      }
    }
  });

  // `this` will **not** be the doc, do **not** use arrow functions for methods
  schem<strong class="answer">答：</strong>method.arrowMethod = () => this;
  schem<strong class="answer">答：</strong>virtual('virtualWithArrow').get(() => {
    // `this` will **not** be the doc, do **not** use arrow functions for virtuals
    console.log(this);
  });
  ```

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  I have an embedded property named type like this:

  ```js
  const holdingSchema = new Schema({
    // You might expect `asset` to be an object that has 2 properties,
    // but unfortunately `type` is special in mongoose so mongoose
    // interprets this schema to mean that `asset` is a string
    asset: {
      type: String,
      ticker: String
    }
  });
  ```

  But mongoose gives me a CastError telling me that it can't cast an object to a string when I try to save a Holding with an asset object. Why is this?

  ```js
  Holding.create({ asset: { type: 'stock', ticker: 'MDB' } }).catch(error => {
    // Cast to String failed for value "{ type: 'stock', ticker: 'MDB' }" at path "asset"
    console.error(error);
  });
  ```

  <strong class="answer">答：</strong> The type property is special in mongoose, so when you say type: String, mongoose interprets it as a type declaration. In the above schema, mongoose thinks asset is a string, not an object. Do this instead:

  ```js
  const holdingSchema = new Schema({
    // This is how you tell mongoose you mean `asset` is an object with
    // a string property `type`, as opposed to telling mongoose that `asset`
    // is a string.
    asset: {
      type: { type: String },
      ticker: String
    }
  });
  ```

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  Why don't in-place modifications to date objects (e.g. date.setMonth(1);) get saved?

  ```js
  doc.createdAt.setDate(2011, 5, 1);
  doc.save(); // createdAt changes won't get saved!
  ```

  <strong class="answer">答：</strong> Mongoose currently doesn't watch for in-place updates to date objects. If you have need for this feature, feel free to discuss on this GitHub issue. There are several workarounds:

  ```js
  doc.createdAt.setDate(2011, 5, 1);
  doc.markModified('createdAt');
  doc.save(); // Works
  doc.createdAt = new Date(2011, 5, 1).setHours(4);
  doc.save(); // Works
  ```

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  I'm populating a nested property under an array like the below code:

  ```js
  new Schema({ arr: [{ child: { ref: 'OtherModel', type: Schem<strong class="answer">答：</strong>Types.ObjectId } }] });
  ```

  .populate({ path: 'arr.child', options: { sort: 'name' } }) won't sort by arr.child.name?

  <strong class="answer">答：</strong> See this GitHub issue. It's a known issue but one that's exceptionally difficult to fix.

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  All function calls on my models hang, what am I doing wrong?

  <strong class="answer">答：</strong> By default, mongoose will buffer your function calls until it can connect to MongoDB. Read the buffering section of the connection docs for more information.

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  How can I enable debugging?

  <strong class="answer">答：</strong> Set the debug option to true:

  ```js
  mongoose.set('debug', true)
  ```

  All executed collection methods will log output of their arguments to your console.

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  My save() callback never executes. What am I doing wrong?

  <strong class="answer">答：</strong> All collection actions (insert, remove, queries, etc.) are queued until the connection opens. It is likely that an error occurred while attempting to connect. Try adding an error handler to your connection.

  ```js
  // if connecting on the default mongoose connection
  mongoose.connect(..);
  mongoose.connection.on('error', handleError);

  // if connecting on a separate connection
  var conn = mongoose.createConnection(..);
  conn.on('error', handleError);
  ```

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  Should I create/destroy a new connection for each database operation?

  <strong class="answer">答：</strong> No. Open your connection when your application starts up and leave it open until the application shuts down.

* <strong class="question"><span class='q-icon'><i class="fa fa-question" aria-hidden="true"></i></span>问：</strong>
  Why do I get "OverwriteModelError: Cannot overwrite .. model once compiled" when I use nodemon / a testing framework?

  <strong class="answer">答：</strong> mongoose.model('ModelName', schema) requires 'ModelName' to be unique, so you can access the model by using mongoose.model('ModelName'). If you put mongoose.model('ModelName', schema); in a mocha beforeEach() hook, this code will attempt to create a new model named 'ModelName' before every test, and so you will get an error. Make sure you only create a new model with a given name once. If you need to create multiple models with the same name, create a new connection and bind the model to the connection.

  ```js
  var mongoose = require('mongoose');
  var connection = mongoose.createConnection(..);

  // use mongoose.Schema
  var kittySchema = mongoose.Schema({ name: String });

  // use connection.model
  var Kitten = connection.model('Kitten', kittySchema);
  ```

#### Something to add?

If you'd like to contribute to this page, please visit it on github and use the Edit button to send a pull request.
