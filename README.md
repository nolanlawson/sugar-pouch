SugarPouch
==========

SugarPouch is a PouchDB plugin that offers a simpler, more streamlined API for some common PouchDB operations.  It's syntactic sugar, but in many cases it's also a utility belt. If PouchDB is jQuery, think of SugarPouch as underscore.

**Warning!  This is vaporware.  I'm doing doc-driven development, so I'm writing the docs before I write any working code.  Do not be tricked by this README into thinking that this is a working plugin!  This warning will be removed when it's ready.**
API
---

* putMany
* getMany
* upsert
* createIndex
* findByXXX
* countByXXX
* count

All operations are on a PouchDB database, e.g.

```js
var pouch = new PouchDB('mydb');
```

#### pouch.putMany(documents [, options] [, callback])

The same as `bulkDocs`, but it takes a list of documents and returns that same list of documents with the `_rev` added (and the `_id` added, if you didn't include an `_id`).  Within the list, an error object is returned if a document conflict occurred.

#### pouch.getMany([, options] [, callback])

The same as `allDocs`, but directly returns a list of documents (i.e. `include_docs` is always `true`).

#### pouch.upsert(docId, updateFunction [, callback])

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

You don't have to worry about putting an `_id` field or a `_rev` field on the document; we'll handle that automatically.


#### pouch.createIndex(indexSpec [, options] [, callback])

Create a new index (i.e. a design doc with a view) using the given specification, or do nothing if the index already exists.  Indexes are defined based on the field in the document you'd like to index.  Some examples:

```js
pouch.createIndex('name'); // index on doc.name
pouch.createIndex(['firstName', 'lastName']); // index on [doc.firstName, doc.lastName]
pouch.createIndex('employee.spouse.name') // index on doc.employee.spouse.name
```

##### Options

* `skip_nulls`: defaults to false.  True if you want to skip any fields that are null; will save space on database where only a few documents actually have that field.  Example:

```js
// only index if the doc has a field called 'favoritePokemon'
pouch.createIndex('favoritePokemon', {skip_nulls: true});
```

#### pouch.findByXXX()

Okay, now that you've created an index on `'name'`, your `pouch` will suddenly have a method called `findByName()`.  Imagine that!

If you index on `'employee.spouse.name'`, you'll have a method called `findByNameOfSpouseOfEmployee()`.  Names are automatically CamelCased for your convenience.

If you index on multiple fields like `['firstName', 'lastName']`, you'll have a brand-new method called `findByFirstNameAndLastName()`. You'll also get a bonus method `findByFirstName()`, because PouchDB just works like that, and it's awesome.


Examples:

```js
pouch.findByName('sally'); // find docs where doc.name === 'sally'
pouch.findByName('>=', 's'); // find docs where doc.name >= 's'
pouch.findByName('>=', 's', 'and', '<', 'z'); // find docs where 's' <= doc.name < 'z'
pouch.findByName('within', 'sally', 'jimbo', 'lucy'); // find where doc.name is one of those 3
```

The `options` you pass in are the same as for `query()`.  So you can always pass in options like `{descending: true}`, `{limit: 10}`, etc.

#### pouch.countByXXX()

Guess what?  Now that you've created an index on `'name'`, you also have a method called `countByName()`.  It's the same as `findByName()`, except it returns the total count of documents that match your criteria.

```js
pouch.countByName('sally'); // how many people are named sally?
```

#### pouch.count()

Returns the total count of non-deleted documents in the database.
