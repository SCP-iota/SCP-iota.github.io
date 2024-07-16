---
layout: blog_post
category: software
title: Breaking JavaScript Objects
---

Remember SQL injection? While the days of extracting password hashes with carefully crafted usernames might be ([mostly](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-48788)) over, there are still some clever exploits that use trojan strings to cause unintended effects, and sometimes even the simplest of code can be vulnerable. If you ever thought your JavaScript code that uses objects is dictionaries is secure, think again.

Modern JavaScript has better ways to implement dictionaries anyway, like [`Map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map), but there's still plenty of code out there using regular objects for key-value storage - a relic of ES5 compatibility. To see why this can backfire, we need some context...

## Prototype Inheritance

Type inheritance in JavaScript is implemented by allowing objects to reference "prototypes" - other objects that the runtime can fall back on if the original object doesn't have an expected property. For example, instances of a `Cat` class may not have their own `eat` method, but they might have `Animal.prototype` as their prototype reference, which does have an `eat` method. When the runtime looks for the `eat` property of the `Cat` instance and doesn't find it, it looks for an `eat` property on the prototype.

## The Perils of Nonstandard Functionality

While changing the prototype of an object is discouraged for optimization reasons, the ECMAScript specification does standardize a [`setPrototypeOf`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf) method for that purpose. However, certain JavaScript engines (namely Chromium's V8 engine) implement a nonstandard `__proto__` special property on all (or almost all - see below) objects to reference the object's prototype. This property exists as a getter/setter pair on `Object.prototype`, which almost all objects inherit from.

Not only is this problematic because code might rely on that property and fail to run in a purely ECMAScript compliant runtime, it can also cause unusual bugs.

## Did You Really Name Your Kid `__proto__`?

Consider a website backend made with Node.js. The website has user-registered accounts with custom usernames, and although the user records themselves are stored in a database, some user data is cached as a dictionary in the running JavaScript code - something like this:

```js
// "Dictionary" to cache certain user data
const userCache = {}

class CachedUserData {
    constructor(lastLogin) {
        this.lastLogin = lastLogin
    }

    // Update `lastLogin` to now
    login() {
        this.lastLogin = new Date()
    }
}

// Called when a user logs in
function onUserLogin(username) {
    if(Object.hasOwn(userCache, username))
        userCache[username].login()
    else
        userCache[username] = new CachedUserData(new Date())
}
```

Notice the use of [`Object.hasOwn`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn) and *not* [`hasOwnProperty`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwnProperty) - that's to prevent a different bizarre issue:

```js
const obj = {
    x: 42,
    hasOwnProperty: null
}
console.log(obj.hasOwnProperty('x'))
// Uncaught TypeError: obj.hasOwnProperty is not a function
```

At least that won't happen to our website's backend, but what if we login a user named `__proto__`?

```js
onUserLogin('__proto__')
```

Oh - no error. Everything must be fine after all; but to make sure, let's see if the user now shows up in the cache...

```js
console.log(Object.keys(userCache))
// []
```

Where did it go? To the `__proto__` property, of course. The `CachedUserData` object became the prototype of `userCache`, and since `Object.keys` only lists *owned* properties and excludes inherited ones, the dictionary still appears empty. Strange, but it shouldn't cause issues for any other users, right?... Right?

```js
// Log when user caches change
function logCacheChange(username) {
    if(!Object.hasOwn(userCache, username))
        throw new Error(`User ${username} not cached`)

    console.log(`Cache for user ${username} updated: ${JSON.stringify(userCache[username])}`)
}

// Alter this to call the logger after updating the cache
function onUserLogin(username) {
    if(Object.hasOwn(userCache, username))
        userCache[username].login()
    else
        userCache[username] = new CachedUserData(new Date())

    logCacheChange(username)
}
```

Just adding a logger, nothing to see here. Now the infamous `__proto__` user logs in again, but this time, the server errors out.

```js
onUserLogin('__proto__')
// Uncaught Error: User __proto__ not cached
```

Wut? `onUserLogin` makes sure the user is cached before calling `logCachedChange`, so it seems like there should be no problems. However, since `__proto__` is an inherited property, it isn't considered to exist by `Object.hasOwn`, making `logCacheChange` think it has been called on an uncached user.

## Solutions?

When I point out this problem with object dictionaries, one of the first responses I get is usually that the strings should be sanitized - that `__proto__` should be rejected as a username before even trying to process it. While that would technically fix the bug, I don't think it's a good answer, for two reasons:

* `__proto__` is a nonstandard property, so, strictly speaking, codebases shouldn't have to know it exists. What if other JavaScript engines implement different nonstandard properties, or more are added in the future? Can you be sure you've sanitized them all?

* This is a dictionary, a very simple component of coding. It should be able to handle whatever key string is thrown at it. Saying that such a basic piece of code needs to sanitize its input and reject certain strings feels nearly as ridiculous as [that time "Jennifer Null" broke a database](https://www.bbc.com/future/article/20160325-the-names-that-break-computer-systems).

That said, if you really don't want to use a [`Map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map), you can create a null-prototype object to use as a dictionary with `Object.create(null)`, since it won't inherit anything from anywhere:

```js
const userCache = Object.create(null)
```

## Lessons Learned

Hopefully any faith that you had in the security of software is thouroughly destroyed. Ok - maybe that's a bit far - hopefully you'll always remember to check the edge cases before releasing to production. Otherwise, you might have to explain to unhappy users why your servers went down because of an misforunately named account.