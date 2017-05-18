# Azure Cosmos DB (DocumentDB API) TypeScript interface

This Node.js module provides a TypeScript-based wrapper around the Node.js APIs for Microsoft's awesome SQL-queried schema-free NoSQL database in the Azure cloud, ~~DocumentDB~~ Cosmos DB.

Refer to the DocumentDB API documentation here: https://docs.microsoft.com/en-us/azure/documentdb/documentdb-sdk-node

**No TypeScript required** &mdash; you can use this module with plain JavaScript too (ES3, ES5, or ES6 aka ES2015, ES7 and whatever comes after), and enjoy enhanced Intellisense in an editor that supports TypeScript 2 definition files, such as VS Code.

> **NOTE:** The author of this module is _not_ affiliated with Microsoft, Azure, or Cosmos DB / Document DB.

### Goals
This module was written with the following goals in mind:

- Streamline common DocumentDB use cases;
- Enable a better developer experience with accurate Intellisense;
- Reduce clutter by grouping methods into classes and combining some of their functionality;
- Use idiomatic TypeScript 2 (es6 for Node JS) internally and externally;
- Enable asynchronous programming with `async/await` and/or Promises (native Node JS).

### Change Log

**v1.0.5:**

* _Update:_ DocumentDB Node.js API v1.12.0 for Cosmos DB, with support for RU/min billing and ConsistentPrefix consistency level.

**v1.0.4:**

* Minor bug fix

**v1.0.3:**

* _New:_ Added support for Typescript 2.3's `for await (... of ...)` syntax (without breaking support for ES < 6 and TS < 2.3). See example below.

**v1.0.0:**

* _Important:_ This version now requires TypeScript 2.1+.
* _New:_ Added an `existsAsync` method to `Collection` that uses a `select count(1) from c where...` query to determine more efficiently if any documents exist that match given ID or properties.
* _New:_ Added `path` properties to `Database` and `Collection`, which can be used with the underlying Node.js API (through `Client.documentClient`) if needed.
* _New:_ Most methods now accept an `options` parameter to forward feed and/or request options to the underlying Node.js API (e.g. for `enableCrossPartitionQuery`).
* _Improved:_ Where possible, document IDs are now used to locate document resources instead of mandatory `_self` links. This allows for a new overload of the `deleteDocumentAsync` method that just takes an ID, and removes the need for a query in `findDocumentAsync` if an ID is passed in (either as a property or as a single parameter). Also, `storeDocumentAsync` with `StoreMode.UpdateOnly` no longer requires a `_self` property, an `id` property will do.
* _Improved:_ More accurate types for objects passed to and/or returned from `Collection` methods. E.g. query results generated by `queryDocuments` no longer automatically include document properties such as `id` and `_self`, because queries may not actually return full documents anyway (or a document at all, e.g. for `select value...` queries). This is a breaking change since the TypeScript compiler may no longer find these properties on result objects, even for `select *` queries. The `findDocumentAsync` and `queryDocuments` methods now accept a type parameter to specify a result type explicitly.
* _Changed:_ Getting `Client.documentClient` now throws an exception if the client connection has not been opened yet, or has been closed. Use `isOpen()` to check if the connection is currently open.
* _Fixed:_ Operations are now queued properly in `DocumentStream`, e.g. calling `.read()` twice in succession (synchronously) actually returns promises for two different results.
* _Fixed:_ Added `strictNullChecks` and `noImplicitAny` to the TypeScript configuration for compatibility with projects that have these options enabled.
* _Fixed:_ Added TypeScript as a development dependency to `package.json`.

**Note:**

At this point parts of the DocumentDB API feature set are still missing. If your app needs stored procedures, or users and permissions, for example, then please add to this code (preferably as new classes). Pull requests are greatly appreciated!

Tests are sorely needed as well. Perhaps some of the tests can be ported over from DocumentDB itself.

## Installation
Use `npm` to install this module:

```
npm install documentdb-typescript
```

Then import this module into your JavaScript or TypeScript code:

```javascript
const DB = require("documentdb-typescript");
const client = new DB.Client(/*...*/);

// OR: ...
import * as DB from "documentdb-typescript";
const client = new DB.Client(/*...*/);

// OR: ...
import { Client /*,...*/} from "documentdb-typescript";
const client = new Client(/*...*/);
```

## Usage

This module exports only the following symbols:
- `Client` class: contains methods for dealing with the connection to ~~DocumentDB~~ Cosmos DB account.
- `Database` class: represents a database.
- `Collection` class: represents a collection, and contains methods for dealing with documents.
- `DocumentStream` class: contains methods for reading query results (used as a return type only).
- `StoreMode` enum: lists different modes of storing documents in a collection.

*Where is Document?* &mdash;
There is no 'Document' class because documents are really just plain JavaScript objects (of type `any`), which may or may not have some extra properties (such as `_self`) depending on where they come from, and are very hard to pin down as such. Also, the results of a query may or may not be full documents, which makes it impossible to predict exact return types. Adding another abstraction layer (à la the .NET API with its set- / getPropertyValue methods) doesn't seem like the right thing to do in JavaScript code.

## `Client`
Start here if you need to work with multiple databases, or set advanced connection options.

```typescript
// Example: open a connection and find all databases
async function main(url, masterKey) {
    var client = new Client(url, masterKey);

    // enable logging of all operations to the console
    client.enableConsoleLog = true;

    // dump the account information
    console.log(await client.getAccountInfoAsync());

    // open the connection and print a list of IDs
    await client.openAsync();
    var dbs = await client.listDatabasesAsync();
    console.log(dbs.map(db => db.id));

    // unnecessary unless you expect new clients
    // to reopen the connection:
    client.close();
}
```

The original DocumentClient from the `documentdb` module is kept in the `documentClient` property, after the `openAsync` method is called:

A static property `Client.concurrencyLimit` (number) controls how many requests may be outstanding *globally* at any time; this defaults to 25. Further requests are held internally (without timing out) until a pending request completes. You may want to increase this number if you are performing a high volume of low-cost operations such as deletions.

## `Database`
Start here if you need to list all collections in a database, or delete it from the account. Nothing else here.

```typescript
// Example: get a list of collections
async function main(url, masterKey) {
    var client = new Client(url, masterKey);
    var db = new Database("sample", client);

    // ... or this, which creates a new client
    // but reuses the connection:
    var db2 = new Database("sample", url, masterKey);

    // create the database if necessary
    await db.openOrCreateAsync();

    // ... or not at all (fails if not found)
    await db.openAsync();

    // print a list of collection IDs
    var colls = await db.listCollectionsAsync();
    console.log(colls.map(c => c.id));

    // delete the database
    await db.deleteAsync();
}
```

## `Collection`
This is where most of the functionality lives. Finding and/or creating a collection, optionally along with the database is easy:

```typescript
// Example: create and delete a collection
async function main(url, masterKey) {
    var client = new Client(url, masterKey);
    var db = new Database("sample", client);

    // these are all the same:
    var coll = new Collection("test", db);
    var coll2 = new Collection("test", "sample", client);
    var coll3 = new Collection("test", "sample", url, masterKey);
    
    // create everything if necessary
    await coll.openOrCreateDatabaseAsync();

    // ... or just the collection
    await coll.openOrCreateAsync();

    // ... or nothing (fails if not found)
    await coll.openAsync();

    // delete the collection
    await coll.deleteAsync();
}
```

The `Collection` instance has methods for setting and getting provisioned throughput levels:

```typescript
// Example: set and get throughput information 
async function main(url, masterKey) {
    var client = new Client(url, masterKey);
    var coll = new Collection("test", "sample", client);
    await coll.openOrCreateDatabaseAsync();

    // set the offer throughput
    await coll.setOfferInfoAsync(500);
    
    // dump the new offer information
    console.log(await coll.getOfferInfoAsync());
}
```

## Storing documents
This module abstracts away most of the work involved in creating, updating, and upserting documents in a collection.

```typescript
// Example: store and delete a document
async function main(url, masterKey) {
    var client = new Client(url, masterKey);
    client.enableConsoleLog = true;
    var coll = new Collection("test", "sample", client);
    await coll.openOrCreateDatabaseAsync();

    // create a document (fails if ID exists),
    // returns document with meta properties
    var doc = { id: "abc", foo: "bar" };
    doc = await coll.storeDocumentAsync(doc, StoreMode.CreateOnly);

    // update a document (fails if not found)
    doc.foo = "baz";
    doc = await coll.storeDocumentAsync(doc, StoreMode.UpdateOnly);

    // update a document if not changed in DB,
    // using _etag property (which must exist)
    doc.foo = "bla";
    doc = await coll.storeDocumentAsync(doc, StoreMode.UpdateOnlyIfNoChange);
    
    // upsert a document (in parallel, without errors)
    var doc2 = { id: "abc", foo: "bar" };
    var doc3 = { id: "abc", foo: "baz" };
    await Promise.all([
        coll.storeDocumentAsync(doc, StoreMode.Upsert),
        coll.storeDocumentAsync(doc)  // same
    ]);

    // delete the document (fails if not found)
    await coll.deleteDocumentAsync(doc);

    // ... or delete by ID (fails if not found)
    await coll.deleteDocumentAsync("abc");
}
```

## Finding documents
There are a number of ways to find a document or a set of documents in a collection.

```typescript
// Example: find document(s)
async function main(url, masterKey) {
    var coll = await new Collection("test", "sample", url, masterKey)
        .openOrCreateDatabaseAsync();

    // check if a document with given ID exists
    // (uses "count(1)" aggregate in a query)
    var exists = coll.existsAsync("abc");

    // check if a document with given properties exists
    // (exact match, also uses "count(1)" aggregate)
    var customerExists = coll.existsAsync({
        isCustomer: true,
        customerID: "1234"
    })

    // retrieve a document by ID (fails if not found)
    var doc = await coll.findDocumentAsync("abc");

    // retrieve a document with given properties
    // (exact match, fails if not found, takes
    // newest if multiple documents match)
    try {
        var user = await coll.findDocumentAsync({
            isAccount: true,
            isInactive: false,
            email: "foo@example.com"
        });
        console.log(`Found ${user.email}: ${user.id}`);
    }
    catch (err) {
        console.log("User not found");
    }

    // find a set of documents (see below)
    var stream = coll.queryDocuments();  // <= all
    var stream2 = coll.queryDocuments("select * from c");  // same
    var stream3 = coll.queryDocuments({
        query: "select * from c where c.foo = @foo",
        parameters: [
            { name: "@foo", value: "bar" }
        ]
    });
}
```

## Iterating over query results
The `queryDocuments` method on the `Collection` class is one of the few methods that does *not* return a Promise. Instead, it returns a `DocumentStream` instance which can be used to iterate over the results or load them all in one go.

The `DocumentStream` class exposes a number of methods (e.g. `forEach` and `mapAsync`)

If you do not wish to use the Typescript/ES6 `await` keyword, you can use the `Promise` object returned by `next`, `read`, `forEach`, or `mapAsync` instead.

```typescript
// Example: iterate over query results
async function main(url, masterKey) {
    var coll = await new Collection("test", "sample", url, masterKey)
        .openOrCreateDatabaseAsync();

    // load all documents into an array
    var allDocs = await coll.queryDocuments().toArray();

    // the hard way: read all results in a loop (with type hint)
    type FooResult = { foo: string };
    var stream = coll.queryDocuments<FooResult>("select c.foo from c");
    while (true) {
        var { done, value } = await stream.next();
        if (done) break;
        console.log(value.foo);
    }

    // explicitly reset the stream to the beginning if needed:
    await stream.resetAsync();

    // the **new** way (Typescript 2.3): for await ... of
    for await (const doc of stream) {
        console.log(doc.foo);
    }

    // ... or use the forEach method
    // (can be awaited, too)
    await stream.reset().forEach(doc => {
        console.log(doc.foo);
    });

    // ... or map all results to another array
    var ids = await stream.mapAsync(doc => doc.id);
    console.log(ids);

    // use `top 1` to get only the newest time stamp
    var newest = await coll.queryDocuments(
        "select top 1 c._ts from c order by c._ts desc")
        .read();
    if (!newest)
        console.log("No documents");
    else
        console.log("Last change " +
            (Date.now() / 1000 - newest._ts) +
            "s ago");

    // ... or without `await`:
    coll.queryDocuments(
        "select top 1 c._ts from c order by c._ts desc")
        .read()
        .then(newest => {
            if (!newest) console.log("No documents");
            else console.log("Last change" /* + ... */);
        });
}
```
