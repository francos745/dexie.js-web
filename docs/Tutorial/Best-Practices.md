---
layout: docs
title: 'Best Practices'
---

### 1. Understand Promises

Make sure you learn at least the basic practices of A+ promises before diving too deep into Dexie.

Here's a little test. Please review the code below and then ask yourself if you understood what it was doing...

```javascript
function doSomething() {
    // Important: Understand why we use 'return' here and what we actually return!
    return db.friends.where('name').startsWith('A').first().then(function (aFriend) {
        return aFriend.id; // Important: Understand what 'return' means here!
    }).then (function (aFriendsId) {
        // Important: Understand what it means to return another Promise here:
        return fetch ('https://blablabla/friends/' + aFriendsId);
    });
}

// Important: Undestand how you would call and consume the doSomething() function.
```

Was there any of the "Understand" parts that you didn't understand? Then you would benefit from learning a little about promises. This will be useful knowledge whatever lib you'll be using. Some links to dive into:

* [http://www.html5rocks.com/en/tutorials/es6/promises/](http://www.html5rocks.com/en/tutorials/es6/promises/)
* [https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)

### 2. Be wise when catching promises!

Short Version:

```
Top-level code (such as on event handlers) is where you should catch promises.
Everywhere else, you should return promises to your callers rather than
catching them, unless:
   ...you really handle the error (not just log it)
   ...you just want to log it, but then also re-throw it. 
```

Long Version:

It's bad practice to do this everywhere:

```javascript
function somePromiseReturningFunc() {
    return db.friends.add({
        name: 'foo',
        age: 40
    }).catch (function (err) {
        console.log(err);
    });
}
```

It's much better to do just this:

```javascript
function somePromiseReturningFunc() {
    return db.friends.add({
        name: 'foo',
        age: 40
    });
    // Don't catch! The caller wouldn't find out that an error occurred if you do!
    // We are returning a Promise, aren't we? Let caller catch it instead!
}
```

If you catch a promise, your resulting promise will be considered successful. It's like doing try..catch in a function where it should be done from the caller, or caller's caller instead. Your flow would continue even after the error has occured.

In transaction scopes, it is even more important to NOT catch promises because if you do, transaction will commit! Catching a promise should mean you have a way to handle the error gracefully. If you don't have that, don't catch it!

```javascript
function myDataOperations() {
    return db.transaction('rw', db.friends, db.pets, function(){
        return db.friends.add({name: 'foo'}).then(function(id){
            return db.pets.add({name: 'bar', daddy: id});
        }).then (function() {
            return db.pets.where('name').startsWith('b').toArray();
        }).then (function (pets) {
            ....
        }); // Don't catch! Transaction SHOULD abort if error occur, shouldn't it?

    }); // Don't catch! Let the caller catch us instead! I mean we are returning a promise, aren't we?!
}
```

But on an event handler or other root-level scope, always catch! Why?! Because you are the last one to catch it since you are NOT returning Promise! You have no caller that expects a promise and you are the sole responsible of catching and informing the user about any error. If you don't catch it anywhere, an error will end-up in the standard [unhandledrejection](https://dexie.org/docs/Promise/unhandledrejection-event.html) event.

```javascript
somePromiseReturningFunc().catch(function (err) {
    $('#appErrorLabel').text(err);
    console.error(err.stack || err);
});
```

Sometimes you really WANT to handle an explicit error because you know it can happen and you have a way to work around it.

```javascript
function getHillary() {
  return db.friends
    .where('[firstName+lastName]')
    .equals(['Hillary', 'Clinton'])
    .toArray()
    .catch('DataError', function (err) {
      // May fail in IE/Edge because it lacks support for compound keys.
      // Use a fallback method:
      return db.friends
        .where('firstName')
        .equals('Hillary')
        .and(function (friend) {
          return friend.lastName == 'Clinton';
        });
    });
}
```
In the above exampe, we are handling the error because we know it may happen and we have a way to solve that.

What about if you want to log stuff for debugging purpose? Just remember to rethrow the error if you do.

```javascript
function myFunc() {
    return Promise.resolve().then(function(){
        return db.friends.add({name: 'foo'});
    }).catch(function (err) {
        console.error("Failed to add foo!: " + err);
        throw err; // Re-throw the error to abort flow!
    }).then(function(id){
        return db.pets.add({name: 'bar', daddy: id});
    }).catch(function (err) {
        console.error("Failed to add bar!: " + err);
        throw err; // Re-throw the error!
    }).then (function() {
        ...
    });
};
```

### 3. Avoid using other async APIs inside transactions

IndexedDB will commit a transaction as soon as it isn't used within a tick. This means that you MUST NOT call any other async API (at least not wait for it to finish) within a transaction scope. If you do, you will get a TransactionInactiveError thrown at you as soon as you try to use the transaction after having waited for the other async API.

In case you really need to call a short-lived async-API, Dexie can actually keep your transaction alive for you if you use [Dexie.waitFor()](https://dexie.org/docs/Dexie/Dexie.waitFor()).

### 4. Use the global Promise within transactions!

Make sure to use the **global** promise (window.Promise) within transactions. You may use a Promise polyfill for old browsers like IE10/IE11, but just make sure to put it on window.Promise in your page bootstrap.

#### OK:
```javascript
import PromisePolyfill from 'promise-polyfill-of-your-choice'; 

// To add to window
if (!window.Promise) {
  window.Promise = PromisePolyfill;
}
...

db.transaction(..., ()=>{
    Promise.all()
    Promise.race()
    new Promise((resolve, reject) => { ... })
})
```

#### NOT OK:
```javascript
import Promise form 'promise-polyfill-of-your-choice'; 

db.transaction(..., ()=>{
    Promise.all()
    Promise.race()
    new Promise((resolve, reject) => { ... })
})
```

In the case you write a library (not an app) and you want your library to work on old browsers without requiring a Promise polyfill, it is still safe to use Dexie.Promise:

#### STILL ALSO OK:
```javascript
db.transaction(..., ()=>{
    Dexie.Promise.all()
    Dexie.Promise.race()
    new Dexie.Promise((resolve, reject) => { ... })
})
```

### 5. Use transaction() scopes whenever you plan to make more than one operation
Whenever you are going to do more than a single operation on your database in a sequence, use a transaction. This will not only encapsulate your changes into an atomic operation, but also optimize your code! Internally, non-transactional operations also use a transaction but it is only used in the single operation, so if you surround your code within a transaction, you will perform less costly operations in total.

Using transactions gives you the following benefits:

* Robustness: If any error occur, transaction will be rolled back!
* Simpler code: You may do all operations sequencially - they get queued on the transaction.
* One single line to catch them all - exceptions, errors wherever they occur.
* You can just fire off database operations without handling returned promises. The transaction block will catch any error explicitely.
* Faster execution
* Remember that a browser can close down at any moment. Think about what would happen if the user closes the browser somewhere between your operations. Would that lead to an invalid state? If so, use a transaction - that will make all operations abort if browser is closed between operations.

Here is how you enter a transaction block:

```javascript
db.transaction("rw", db.friends, db.pets, function() {
    db.friends.add({name: "Måns", isCloseFriend: 1}); // unhandled promise = ok!
    db.friends.add({name: "Nils", isCloseFriend: 1}); // unhandled promise = ok!
    db.friends.add({name: "Jon", isCloseFriend: 1});  // unhandled promise = ok!
    db.pets.add({name: "Josephina", kind: "dog"});    // unhandled promise = ok!
    // If any of the promises above fails, transaction will abort and it's promise
    // reject.

    // Since we are in a transaction, we can query the table right away and
    // still get the results of the write operations above.
    var promise = db.friends.where("isCloseFriend").equals(1).toArray();

    // Make the transaction resolve with the last promise result
    return promise;

}).then(function (closeFriends) {

    // Transaction complete.
    console.log("My close friends: " + JSON.stringify(closeFriends));

}).catch(function (error) {

    // Log or display the error.
    console.error(error);
    // Notice that when using a transaction, it's enough to catch
    // the transaction Promise instead of each db operation promise.
});
```

Notes:
* `friends` and `pets` are objectStores registered using [Version.stores()](/docs/Version/Version.stores()) method.
* `"rw"` should be replaced with `"r"` if you are just going to read from database.
* Also errors occurring in nested callbacks in the block will be catched by the catch() method.


### 6. Rethrow errors if transaction should be aborted

Saying this again.

When you catch database operations explicitely for logging purpose, transaction will not abort unless you rethrow the error or return the rejected Promise.

```javascript
db.transaction("rw", db.friends, function () {
    return Promise.all([
        db.friends.add ({name: "Måns", isCloseFriend: 1})
          .catch(function (error) {
              console.error("Couldnt add Måns to the database");
              // If not rethrowing here, error will be regarded as "handled"
              // and transaction would not abort.
              throw error;
          }),
        db.friends.add ({name: "Nils", isCloseFriend: 1})
   ]);
}).then(function () {
    // transaction committed
}).catch(function (error) {
    // transaction failed
});
```
If not rethrowing the error, Nils would be successfully added and transaction would commit since the error is regarded as handled when you catch the database operation.

An alternate way of rethrowing the error is to replace `throw error;` with `return Promise.reject(error)`.

### 7. (Optionally:) Declare Classes

When you declare your object stores (`db.version(1).stores({...})`), you only specify nescessary indexes, not all properties. A good practice is to have a more detailed class declaration for your persistant classes. It will help the IDE with autocomplete making life easier for you while coding. Also, it is very good to have a reference somewhere of what properties are actually used on your objects.

There are two different methods available to accomplish this. Use whichever you prefer:

1. [mapToClass()](/docs/Table/Table.mapToClass()) - map an existing class to an objectStore
2. [defineClass()](/docs/Table/Table.defineClass()) - let Dexie declare a class for you

Whichever method you use, your database will return real instances of your mapped class, so that the expression

```javascript
(obj instanceof Class)
```
will return true, and you may use any methods declared via

```javascript
Class.prototype.method = function(){}
```
.

##### Method 1: Use [mapToClass()](/docs/Table/Table.mapToClass()) (map existing class)

```javascript
var db = new Dexie("MyAppDB");

db.version(1).stores({
    folders: "++id,&path",
    files: "++id,filename,extension,folderId"
});

// Folder class
function Folder(path, description) {
    this.path = path;
    this.description = description; 
}
Folder.prototype.save = function () {
    return db.folders.put(this);
}

/// File class
function File(filename, extention, parentFolderId) {
    this.filename = filename;
    this.extention = extention;
    this.folderId = parentFolderId;
}

File.prototype.save = function () {
    return db.files.put(this);
}

db.folders.mapToClass(Folder);
db.files.mapToClass(File);
```

##### Method 2: Use [defineClass()](/docs/Table/Table.defineClass())

```javascript
var db = new Dexie("MyAppDB");

db.version(1).stores({
    folders: "++id,&path",
    files: "++id,filename,extension,folderId"
});

var Folder = db.folders.defineClass({
    id: Number,
    path: String,
    description: String
});

Folder.prototype.save = function () {
    return db.folders.put(this);
}

var File = db.files.defineClass({
    id: Number,
    filename: String,
    extension: String,
    folderId: Number,
    tags: [String]
});

File.prototype.save = function () {
    return db.files.put(this);
}

```

### [Back to Tutorial](/docs/Tutorial)
