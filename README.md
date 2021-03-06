SugarPouch
==========

SugarPouch is a PouchDB plugin that offers a simpler, more streamlined API for common PouchDB operations.  It's syntactic sugar, but in many cases it's also a utility belt. If PouchDB is jQuery for databases, then think of SugarPouch as your Underscore.

```js
var pouch = new PouchDB('myDB');

pouch.putMany([
  {name: 'charmander'},
  {name: 'pikachu'}
  {name: 'squirtle'}
]).then(function () {
  return pouch.createIndex('name');
}).then(function () {
  return pouch.findByName('pikachu'); // pikachu, I choose you!
}).then(function (docs) {
  // you got pikachu!
});
```

**Warning!  This is vaporware.  I'm doing doc-driven development, i.e. I'm writing the docs before any working code.  This warning will be removed when the plugin is ready.**

API
---

All operations are on a PouchDB database `pouch`, e.g.

```js
var pouch = new PouchDB('myDB');
```

**Indexes & querying**

* createIndex
* findBy*
* countBy*
* max*
* min*

**Built-in query functions**

* findById
* maxId
* minId

**Convenience functions**

* putMany
* upsert
* dbSize
* dbVersion

**Utility functions**

* createBlob
* supportedAdapters

### Indexes & querying

#### createIndex(indexSpec [, options] [, callback])

Create a new index using the given specification, or do nothing if the index already exists.  Indexes are defined based on the field in the document you'd like to index.  Some examples:

```js
pouch.createIndex('name'); // index on doc.name

pouch.createIndex(['firstName', 'lastName']); // index on [doc.firstName, doc.lastName]

pouch.createIndex('employee.spouse.name') // index on doc.employee.spouse.name if it exists
```

##### Options

* `skip_nulls`: defaults to false.  True if you want to skip any fields that are null, which will save space on disk if there are only a few documents that have that field.  Example:

```js
// only index if the doc has a field called 'favoritePokemon'
pouch.createIndex('favoritePokemon', {skip_nulls: true});
```

#### findBy*([criteria] [, options] [, callback])

Okay, now that you've created an index on `'name'`, your `pouch` will suddenly have a method called `findByName()`.  Cool beans!

If you index on `'employee.spouse.name'`, you'll have a method called `findByNameOfSpouseOfEmployee()`.  Names are automatically CamelCased for your convenience.

If you index on multiple fields like `['firstName', 'lastName']`, you'll have a brand-new method called `findByFirstNameAndLastName()`.

Examples:

```js
pouch.findByName('sally'); // find docs where doc.name === 'sally'
pouch.findByName('>=', 's'); // find docs where doc.name >= 's'
pouch.findByName('>=', 's', 'and', '<', 'z'); // find docs where 's' <= doc.name < 'z'
pouch.findByName('in', 'sally', 'jimbo', 'lucy'); // find where doc.name is one of those 3
```

The available comparison operators are `'='`, `'<'`, `'>'`, `'<='`, `'>='`.  If you mess up and put `'=='` or `'==='` then it's cool, we know what you mean.

The `options` you pass in are the same as for `query()`.  So you can always pass in options like `{descending: true}`, `{limit: 10}`, etc.

#### countBy*(criteria [, options] [, callback])

Guess what?  Now that you've created an index on `'name'`, you also have a method called `countByName()`.  It's the same as `findByName()`, except it returns the total count of documents that match your criteria.
```js
pouch.countByName('sally'); // how many people are named sally?
```

If you're calling a remote CouchDB, then map/reduce is used under the hood, so the counting is distributed and only the count itself is sent over the wire.

#### max*(criteria [, options] [, callback])
#### min*(criteria [, options] [, callback])

The fun doesn't stop there: you also now have `max*()` and `min*()` methods.  So let's assume you've created an index on `age`:

```js
pouch.createIndex('age'); // ain't nothin' but a number
```

So now you get this:

```js
pouch.maxAge(); // returns the doc with the max age
pouch.minAge(); // returns the doc with the min age
```

### Built-in query functions

#### findById(criteria [, options] [, callback])
#### maxId(criteria [, options] [, callback])
#### minId(criteria [, options] [, callback])

Since `_id` is the primary index for a document, you also get these functions for free.  They work exactly the same as the `findBy*()`, `max*()`, etc. methods.  Go have fun.

If you want a `countById()` method, though, you'll have to create an explicit index on `id` first, since we need to make an explicit map/reduce function.  Sorry, I don't make the rules; I just enforce 'em.

```js
pouch.createIndex('_id').then(function () {
  return pouch.countById('>=', 'foo');
}).then(function (count) {
  console.log('got count: ' + count);
}).catch(function (err) { /* ... */ });
```

### Convenience functions

#### putMany(documents [, options] [, callback])

The same as `bulkDocs`, but it takes a list of documents and returns that same list of documents with the `_rev` added (and the `_id` added, if you didn't include an `_id`).  Within the list, an error object is returned if a document conflict occurred.

#### upsert(docId, updateFunction [, callback])

Updates or inserts a document with the given `docId`, using the given `updateFunction`.  An update function take a document as input, and returns that same document with any changes you would like to apply.  The database is repeatedly called until we get the freshest version of that document, at which point the update is applied.

The function can return `undefined` or `false` instead of a document object, in which case the update will not be applied.  (This is a performance improvement and is not strictly necessary.)

```js
pouch.upsert('myDocId', function (doc) {
  if (doc.favoritePokemon) {
    if (doc.favoritePokemon.indexOf('pikachu') === -1) {
      // you don't like Pikachu?  You're crazy, you love Pikachu
      doc.favoritePokemon.push('pikachu');
      return doc;
    } else {
      // you already like Pikachu? awesome, no update necessary
      return false;
    }
  } else {
    // you don't even have a list of favorite pokemon? screw you, you're deleted
    return {_deleted: true};
  }
});
```

If the document doesn't exist yet, the function will be passed an empty object `{}`. You don't have to worry about putting an `_id` or a `_rev` on the document; we'll handle that automatically.

#### dbSize([callback])

Returns the total count of non-deleted documents in the database.

#### dbVersion([callback])

Returns the current version of the database (a.k.a. the `update_seq`).  Starts at 0, and increments every time a document is added or modified.

### Utility functions

#### createBlob(data [, type] [, callback])

Cross-browser shim for the `new Blob()` function.  Useful for working with attachments.

#### supportedAdapters([callback])

Returns the list of local adapters that this browser supports, e.g. `['idb']` on Firefox, `['websql']` on Safari, `['idb', 'websql']` on Chrome, and `['idb', 'websql', 'localstorage']` on Chrome if you have the localstorage plugin installed.

#### toIndexableString(obj)

Converts any object into an indexable string that still maintains CouchDB collation ordering. What this basically means is that any object you would normally emit as a key in map/reduce can be used as a document ID, thus avoiding building up a second index and speeding up your queries. This is the same function used by PouchDB under the hood for map/reduce.
