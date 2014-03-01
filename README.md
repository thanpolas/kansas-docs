# Kansas API Documentation

*Token based rate limited API to go!*

## <a name='TOC'>Table of Contents</a>

  1. [Configuration](#configuration)
  1. [Database Connection](#connect)
  1. [Policies](#policies)
    1. [Creating Policies](#policies-creating)
    1. [Reading Policies](#policies-read)
    1. [Checking Policies](#policies-has)
    1. [Change Owner's Policy](#policies-change)
    1. [The Policy Item](#policies-item)
  1. [Tokens](#tokens)
    1. [Creating Tokens](#creating-tokens)
    1. [Fetching Tokens](#get-tokens)
    1. [Deleting Tokens](#del-tokens)
    1. [Consuming Tokens](#consuming-tokens)
    1. [Fetch By Owner Id](#getByOwnerId-tokens)
    1. [The Token Item](#tokens-item)
  1. [Kansas Middleware](#middleware)
  1. [Kansas Events](#events)
  1. [Kansas Errors](#errors)
    1. [Kansas BaseError](#errors-BaseError)
    1. [Kansas Database](#errors-Database)
    1. [Kansas Validation](#errors-Validation)
    1. [Kansas TokenNotExists](#errors-TokenNotExists)
    1. [Kansas UsageLimit](#errors-UsageLimit)
    1. [How to work with Errors](#errors-howtowork)
  1. [Database Maintenance](#maintenance)
    1. [Prepopulating Usage Keys](#maintenance-prepopulate)
    1. [Nuke the Database](#maintenance-nuke)

## <a name='configuration'>Configuration</a>

Kansas can be configured at instantiation or by using the `setup()` method.

```js
var kansas = require('kansas');

var api = kansas({/* options */});

// this will work too
api.setup({/* options */});
```

> **NOTICE** Logging options are an exception and can **only** be set during instantiation

### Available Options

* **prefix** `string` *Optional* A string to prefix all keys on Redis.
* **logging** `boolean` *Optional* Set to `false` Mute all logging, default `true`.
* **console** `boolean` *Optional* Set to `false` to not log to console, default `true`.
* **logg** `logg` *Optional* Kansas uses the [logg](https://github.com/dpup/node-logg) package for logging, use this option to inject your own instance of *logg*.
* **redis** `Object` *Optional* Redis connection options
  * **port** `string|number=` *Optional* Define redis port, default `6379`.
  * **host** `string=` *Optional* The host to connect to, default is `"localhost"`.
  * **pass**  `string=` *Optional* Password to connect with.
  * **redisOptions** `Object=` *Optional* More advanced options to pass directly to *redis-client*.


### Example

```js
var kansas = require('kansas');

var api = kansas({
    // don't log to console
    console: false,
});

api.setup({
    port: 7777,
    host: 'some.redis.host.com',
    password: 'secret',
});
```

**[[⬆]](#TOC)**

## <a name='connect'>Connecting to Redis</a>

> ### api.connect() No arguments.
>
> *Returns* `Promise` A Promise.

Before you start a connection Kansas will refuse to perform any task. Connecting to Redis is a plain Promise returning method, `connect()`.

```js
var kansas = require('kansas');

var api = kansas();

api.connect().then(function() {
  console.log('Connected!');
}).catch(function(err){
  console.error('Opts, something went wrong:', err);
});
```
**[[⬆]](#TOC)**


## <a name='policies'>Policies</a>

  Policies are stored in memory. After many iterations it became apparent that this is the most efficient way to work with policies. You define and maintain them inside your application, they are shared between multiple cores and exist in the memory saving on needless database reads. After all, typically an application is not expected to have more than 10 policies.

### <a name='policies-creating'>Creating policies</a>

> ### api.policy.create(options)
>
>    * **options** `Object` A dictionary with the following options.
>       * **name** `string` The policy's name, uniquely identifying it.
>       * **maxTokens** `number` Maximum number of tokens each owner is allowed to have.
>       * **limit** `number` The maximum number of API calls (units) per period.
>
> *Returns* `Object` [The Policy Item](#policies-item).

To create a policy you need to define three parameters:

```js
api.policy.create({
  name: 'free',
  maxTokens: 3, // max tokens per owner
  limit: 100 // max usage units per period
});
```

**[[⬆]](#TOC)**


### <a name='policies-read'>Reading a Policy</a>

> ### api.policy.get(policyName)
>
>    * **policyName** `string` The policy name to fetch.
>
> *Returns* `Object` [The Policy Item](#policies-item).

```js
var policyItem = api.policy.get('free');
```

**[[⬆]](#TOC)**

### <a name='policies-has'>Checking a Policy exists</a>

> ### api.policy.has(policyName)
>
>    * **policyName** `string` The policy name to fetch.
>
> *Returns* `boolean` Yes or no.

```js
var policyExists = api.policy.has('free');
```

**[[⬆]](#TOC)**


### <a name='policies-change'>Change an Owner's policy</a>

> ### api.policy.change(options)
>
>    * **options** `Object` A dictionary with the following key/value pairs:
>      * **ownerId** `string` The owner's id.
>      * **policyName** `string` The new policy.
>
> *Returns* `Promise` A promise.

You'll typically use this when an owner upgrades or downgrades to a new plan. Beware, the usage units will reset to the Policy's limits.

```js
api.policy.change({
  ownerId: 'a-unique-id'
  policyName: 'basic',
}).then(function() {
  console.log('It Worked!');
}).catch(function(err) {
  console.error('Uhhhhh duh?', err);
});
```


**[[⬆]](#TOC)**

### <a name='policies-item'>The Policy Item</a>

When a policy item is returned from Kansas, this is the structure it will have:

```js
var policyItem = {
  /** @type {string} The policy's name */
  name: 'free',
  /** @type {number} Maximum tokens per owner */
  maxTokens: 3,
  /** @type {number} Maximum usage units per period */
  limit: 'free',
  /** @type {kansas.model.PeriodBucket.Period} Period's enum */
  period: 'month',
};
```

**[[⬆]](#TOC)**

## <a name='tokens'>Tokens</a>

### <a name='creating-tokens'>Creating Tokens</a>

> ### api.create(options)
>
> `api.create` is an alias to `api.set`
>
>    * **options** `Object` A dictionary with the following key/value pairs:
>      * **ownerId** `string` The owner's id.
>      * **policyName** `string` The new policy.
>
> *Returns* `Promise(tokenItem)` A promise returning the [*tokenItem* an *Object*](#tokens-item).

Creates a tokens and populates usage keys and indexes.

```js
api.create({
  ownerId: 'a-unique-id'
  policyName: 'basic',
}).then(function(tokenItem) {
  console.log('It Worked!', tokenItem);
  console.log('Here is the token:', tokenItem.token);
}).catch(function(err) {
  console.error('OH NO!', err);
});
```

> This method has [Before / After Middleware](#middleware).


**[[⬆]](#TOC)**

### <a name='get-tokens'>Fetching Tokens</a>

> ### api.get(token)
>
>    * **token** `string` The token's string.
> *Returns* `Promise(tokenItem)` A promise returning the [*tokenItem* an *Object*](#tokens-item).

Will fetch a token.


```js
api.get('unique-token-id')
.then(function(tokenItem) {
  console.log('It Worked!', tokenItem);
  console.log('Here is the token:', tokenItem.token);
}).catch(function(err) {
  console.error('OH NO!', err);
});
```
> This method has [Before / After Middleware](#middleware).

**[[⬆]](#TOC)**

### <a name='del-tokens'>Deleting Tokens</a>

> ### api.del(token)
>
>    * **token** `string` The token's string.
> *Returns* `Promise()` A promise.

Will delete a token.

```js
api.del('unique-token-id')
.then(function() {
  console.log('Token Deleted!');
}).catch(function(err) {
  console.error('OH NO!', err);
});
```
> This method has [Before / After Middleware](#middleware).

**[[⬆]](#TOC)**

### <a name='consuming-tokens'>Consuming Tokens</a>

> ### api.consume(token, optUnits)
>
>    * **token** `string` The token to consume.
>    * **optUnits=** `number` *Optional* Optionally define how many units to consume.
> *Returns* `Promise(remaining)` A promise returning the remaining units a *number*.

Will consume a unit and return the remaining units.

```js
api.consume('token').then(function(remaining) {
  console.log('Units remaining:', remaining);
}).catch(function(err) {
  console.error('No more units!', err);
});

// consume multiple units
api.consume('token', 5).then(function(remaining) {
  console.log('Units remaining:', remaining);
}).catch(function(err) {
  console.error('No more units!', err);
});

```
> This method has [Before / After Middleware](#middleware).

**[[⬆]](#TOC)**


### <a name='getByOwnerId-tokens'>Fetch Tokens By Owner Id</a>

> ### api.getByOwnerId(ownerId)
>
>    * **ownerId** `string` A string uniquely identifying an owner.
>
> *Returns* `Promise(Array.<Object>)` A promise with an array of [tokenItems](#tokens-item).

Will fetch all tokens based on Owner Id.

```js
api.getByOwnerId('hip')
.then(function(tokens) {
  console.log('Total Tokens:', tokens.length);
}).catch(function(err) {
  console.error('OH NO!', err);
});
```

> This method has [Before / After Middleware](#middleware).

**[[⬆]](#TOC)**

### <a name='tokens-item'>The Token Item</a>

When a Token item is returned from Kansas, this is the structure it will have:

```js
var tokenItem = {
  /** @type {string} The token */
  token: 'random-32-char-string',
  /** @type {string} The Policy's name this token belogs to */
  policyName: 'free',
  /** @type {number} The usage limit per period */
  limit: 10;
  /** @type {kansas.model.PeriodBucket.Period} The period */
  period: 'month';
  /** @type {string} Any string uniquely identifying the owner */
  ownerId: 'hip';
  /** @type {string} An ISO 8601 formated date */
  createdOn: '2014-03-01T18:12:15.711Z';
};
```

**[[⬆]](#TOC)**



## <a name='middleware'>Kansas Middleware</a>

Kansas uses the [Middlewarify package](https://github.com/thanpolas/middlewarify) to apply the middleware pattern to its methods. More specifically the [Before/After type of middleware](https://github.com/thanpolas/middlewarify#using-the-before--after-middleware-type) has been applied to the following methods:

  * api.set()
  * api.get()
  * api.del()
  * api.consume()
  * api.getByOwnerId()
  * api.policy.change()

Each of these methods has the Before/After methods so you can add corresponding middleware. All middleware get the same arguments, all *After* middleware receive an extra argument which is the result of the main middleware resolution.

```js
api.set.before(function(params) {
  console.log(params.ownerId); // prints 'hip'
});

api.set.after(function(params, tokenItem) {
  console.log(tokenItem.token); // the generated token
});

api.create({
  ownerId: 'hip',
  policyName: 'free',
});
```

To pass control asynchronously from a middleware you need to return a Promise:

```js
api.set.before(function(params) {
  return new Promise(function(resolve, reject) {
    performAsyncFunction(function(err) {
        if (err) { return reject(err); }

        // pass control
        resolve();
    });
  });
});
```

**[[⬆]](#TOC)**

## <a name='events'>Kansas Events</a>

You can hook for emitted events using the `on()` method. Kansas will emmit the following type of events:

* **message** A logging message occurred.
  * **logObj** `Object` The logging item object.
* **create** A new Token was created
  * **tokenItem** `Object` The created [Token Item](#tokens-item).
* **delete**
  * **tokenItem** `Object` The deleted [Token Item](#tokens-item).
* **consume** A token was consumed.
  * **token** `string` The token's unique id.
  * **consumed** `number` Units consumed.
  * **remaining** `number` How many units remaining.
* **policyChange** An owner policy change has occurred.
  * **changeObj** `Object` The Change object as passed, contains `ownerId` and `newPolicyName`.
  * **policy** `Object` The new [Policy Item](#policies-item).
* **maxTokens** An owner reached the maximum number of allowed tokens.
  * **params** `Object` The params as passed, contain: `ownerId` and `policyName`.
  * **maxTokens** `number` The maximum number of allowed tokens.

How to hook on events

```js
var api = kansas();

api.on('consume', function(token, consumed, remaining) {
  console.log('Token:', token, ' Consumed:',
    consumed, 'Remaining units:', remaining);
});

```

## <a name='errors'>Kansas Errors</a>

Kansas *signs* all error Objects so you have a better chance of determining what went wrong. All error Constructors are exposed through the `api.error` namespace.

### <a name='errors-BaseError'>api.error.BaseError</a>

All Kansas errors inherit from this Class. BaseError provides these properties:

* **srcError** `Error=` The original raw error instance may be stored in this property, i.e. errors produced by underlying libraries like the Redis Client.
* **name** `string` The name identifying the error, default is `Kansas Error`.

**[[⬆]](#TOC)**

### <a name='errors-Database'>api.error.Database</a>

Errors related to the Redis Database. Provides the following properties

* **srcError** `Error=` The original raw error instance may be stored in this property, i.e. errors produced by underlying libraries like the Redis Client.
* **name** `string` "Database Error"
* **type** `kansas.error.Database.Type` an enum containing these values:
  * `api.error.Database.Type.UNKNOWN` **unknown**
  * `api.error.Database.Type.REDIS` **redis** Redis originated error.
  * `api.error.Database.Type.REDIS_CONNECTION` **redisConnection** Redis connection error.

**[[⬆]](#TOC)**

### <a name='errors-Validation'>api.error.Validation</a>

Validation errors. Provides the following properties

* **name** `string` "Validation Error"  

**[[⬆]](#TOC)**

### <a name='errors-TokenNotExists'>api.error.TokenNotExists</a>

Token does not exist error. Provides the following properties

* **name** `string` "Token Not Exists Error"  

**[[⬆]](#TOC)**

### <a name='errors-UsageLimit'>api.error.UsageLimit</a>

Usage limits errors. Provides the following properties

* **name** `string` "Usage limit exceeded"  

**[[⬆]](#TOC)**

### <a name='errors-howtowork'>How to work with Errors</a>

Use Javascript's `instanceof` to determine the type of error you received.

```js
var kansasError = api.error;

api.consume('unique-token-id')
  .then(onsuccess)
  .catch(function(err) {
    if (err instanceof kansasError.TokenNotExists) {
      console.log('This token does not exist');
    }
    if (err instanceof kansasError.UsageLimit) {
      console.log('Usage limit reached!');
    }
  });
```

**[[⬆]](#TOC)**

## <a name='maintenance'>Database Maintenance</a>

The following Database maintenance tasks are available under the `db` namespace.

**[[⬆]](#TOC)**

### <a name='maintenance-prepopulate'>Prepopulate Usage Keys</a>

  **TL;DR** You need to run the `kansas.db.prepopulate()` method once per month to populate the next month's usage keys.

  Consuming units is the most repeatedly invoked method of Kansas, thus certain performance restrictions apply. In order to have blazing fast speeds the *Consume* operation has been reduced to a single database call, `DECR`.

  In order for this to happen Kansas creates new keys with the limit as value. i.e. a Kansas usage key can be:

  ```
  kansas:usage:2014-02-01:token-unique-id = 10
  ```

  That way Kansas can calculate the key to perform the `DECR` op given the `token` and the current date.

  Each time a token is created Kansas will populate the usage keys for the current period (month) and the next period (month). By design there is no built-in way to prepopulate the usage keys with each passing month. This operation can potentially be very costly and you should have the responsibility and control of when and where it's run.

  > ### api.db.prepopulate()
  >
  > *Returns* `Promise()` A promise.

  Run once each month to populate the usage keys for the  next month.

  ```js
  api.db.prepopulate().then(function() {
    console.log('Done!');
  }).catch(function(err) {
    console.error('An error occurred', err);
  });

  ```
  **[[⬆]](#TOC)**

### <a name='maintenance-nuke'>Nuke the Database</a>

  **WARNING** Irreversible purging of all data **WARNING**

  That said, here's how to nuke the db for your testing purposes:

  > ### api.db.nuke(confirm, confirmPrefix)
  >
  >    * **confirm** `string` A confirmation string.
  >    * **confirmPrefix** `string` Confirm the defined prefix.
  >
  > *Returns* `Promise()` A promise.

  The confirmation string is `Yes purge all records irreversably`

  ```js
  var api = kansas({prefix: 'test'});

  api.db.nuke('Yes purge all records irreversably', 'test').then(function() {
    console.log('Poof Gone!');
  }).catch(function(err) {
    console.error('An error occurred', err);
  });

  ```
  **[[⬆]](#TOC)**
