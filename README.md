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
    1. [Creating Tokens](#creating-tokens) Create a token item.
    1. [Fetching Tokens](#get-tokens) Get a token item.
    1. [Deleting Tokens](#del-tokens) Delete a token item.
    1. [Consuming Tokens](#consuming-tokens) Decrease units from a defined limit.
    1. [Counting Tokens](#count-tokens) Add units starting from 1.
    1. [Fetch By Owner Id](#getByOwnerId-tokens) Get all token items for the provided owner id.
    1. [Get Token Usage](#getUsage) Get the usage items of a token for the current period.
    1. [Get Usage by Owner](#getUsageByOwnerId) Get all the token items including usage items for the current period.
    1. [The Token Item](#tokens-item) A break out of the token item data object.
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

exports.kansas = kansas({/* options */});

// this will work too
exports.kansas.setup({/* options */});
```

> **NOTICE** Logging options cannot be set using the `setup()` method. They can **only** be set during instantiation.

### Available Options

* **prefix** `string` *Optional* A string to prefix all keys on Redis.
* **logging** `boolean` *Optional* Set to `false` Mute all logging, default `true`.
* **console** `boolean` *Optional* Set to `false` to not log to console, default `true`.
* **logg** `logg` *Optional* Kansas uses the [logg](https://github.com/dpup/node-logg) package for logging, use this option to inject your own instance of *logg*.
* **logLevel** `number` A value between 0 to 1000 with highest being more important. Kansas uses the following types of logging presented with their corresponding levels:
  * `error` 1000
  * `warn`   800
  * `info`   600
  * `fine`   400
  * `finer`  200
  * `finest` 100
* **redis** `Object` *Optional* Redis connection options
  * **port** `string|number=` *Optional* Define redis port, default `6379`.
  * **host** `string=` *Optional* The host to connect to, default is `"localhost"`.
  * **pass**  `string=` *Optional* Password to connect with.
  * **redisOptions** `Object=` *Optional* More advanced options to pass directly to *redis-client*.


### Example

```js
var kansas = require('kansas');

exports.kansas = kansas({
    // don't log to console
    console: false,
});

exports.kansas.setup({
    port: 7777,
    host: 'some.redis.host.com',
    password: 'secret',
});
```

**[[⬆]](#TOC)**

## <a name='connect'>Connecting to Redis</a>

> ### kansas.connect() No arguments.
>
> *Returns* `Promise` A Promise.

Before you start a connection Kansas will refuse to perform any task. Connecting to Redis is a plain Promise returning method, `connect()`.

```js
var kansas = require('kansas');

exports.kansas = kansas();

exports.kansas.connect()
  .then(function() {
    console.log('Connected!');
  }).catch(function(err){
    console.error('Opts, something went wrong:', err);
  });
```
**[[⬆]](#TOC)**


## <a name='policies'>Policies</a>

  Policies are stored in memory. After many iterations it became apparent that this is the most efficient way to work with policies. You define and maintain them inside your application, they are shared between multiple cores and stored in memory, saving on needless database reads.

### <a name='policies-creating'>Creating policies</a>

> ### kansas.policy.create(options)
>
>    * **options** `Object` A dictionary with the following options.
>       * **name** `string` The policy's name, uniquely identifying it.
>       * **maxTokens** `number` Maximum number of tokens each owner is allowed to have.
>       * **limit** `number=` The maximum number of API calls (units) per period.
>       * **count** `boolean=` Make this a Count Policy.
>
> *Returns* `Object` [The Policy Item](#policies-item).

There are two types of policies that you can create, the Consume Policy and the Count Policy.

#### The Consume Policy Type

The consuming policies have a finite number of total API calls that can be made within the given period. Each API call consumes one unit from this pool. When all units have been consumed Kansas will deny access to the resource.

To create a Consume Policy you need to define the following parameters:

```js
kansas.policy.create({
  name: 'free',
  maxTokens: 3, // max tokens per owner
  limit: 100 // max usage units per period
});
```

#### The Count Policy Type

The counting policies have no limit to usage, their only purpose is to count how many API calls where performed. Each new period starts the API usage counter from 0 (zero).

To create a counting policy you just need to turn the `count` switch on and omit the `limit` attribute:

```js
kansas.policy.create({
  name: 'enterprise',
  maxTokens: 10,
  count: true,
});
```

**[[⬆]](#TOC)**


### <a name='policies-read'>Reading a Policy</a>

> ### kansas.policy.get(policyName)
>
>    * **policyName** `string` The policy name to fetch.
>
> *Returns* `Object` [The Policy Item](#policies-item).

```js
var policyItem = kansas.policy.get('free');
```

**[[⬆]](#TOC)**

### <a name='policies-has'>Checking a Policy exists</a>

> ### kansas.policy.has(policyName)
>
>    * **policyName** `string` The policy name to fetch.
>
> *Returns* `boolean` Yes or no.

```js
var policyExists = kansas.policy.has('free');
```

**[[⬆]](#TOC)**


### <a name='policies-change'>Change an Owner's policy</a>

> ### kansas.policy.change(options)
>
>    * **options** `Object` A dictionary with the following key/value pairs:
>      * **ownerId** `string` The owner's id.
>      * **policyName** `string` The new policy.
>
> *Returns* `Promise` A promise.

You'll typically use this when an owner upgrades or downgrades to a new plan. Beware, the usage units will reset to the Policy's limits.

```js
kansas.policy.change({
  ownerId: 'a-unique-id'
  policyName: 'basic',
})
  .then(function() {
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
  /** @type {number=} Maximum usage units per period */
  limit: 10,
  /** @type {boolean=} Make this a Count Policy */
  count: false,
  /** @type {kansas.model.PeriodBucket.Period} Period's enum */
  period: 'month',
};
```

**[[⬆]](#TOC)**

## <a name='tokens'>Tokens</a>

### <a name='creating-tokens'>Creating Tokens</a>

> ### kansas.create(options)
>
> `kansas.create` is an alias to `kansas.set`
>
>    * **options** `Object` A dictionary with the following key/value pairs:
>      * **ownerId** `string` The owner's id.
>      * **policyName** `string` The new policy.
>      * **token** `string=` *Optional* Define a custom token name.
>
> *Returns* `Promise(tokenItem)` A promise returning the [*tokenItem* an *Object*](#tokens-item).

Creates a token and populates usage keys and indexes.

```js
kansas.create({
  ownerId: 'a-unique-id'
  policyName: 'basic',
})
  .then(function(tokenItem) {
    console.log('It Worked!', tokenItem);
    console.log('Here is the token:', tokenItem.token);
  }).catch(function(err) {
    console.error('OH NO!', err);
  });
```

> This method has [Before / After Middleware](#middleware).


**[[⬆]](#TOC)**

### <a name='get-tokens'>Fetching Tokens</a>

> ### kansas.get(token)
>
>    * **token** `string` The token's string.
> *Returns* `Promise(tokenItem)` A promise returning the [*tokenItem* an *Object*](#tokens-item).

Will fetch a token.


```js
kansas.get('unique-token-id')
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

> ### kansas.del(token)
>
>    * **token** `string` The token's string.
> *Returns* `Promise()` A promise.

Will delete a token.

```js
kansas.del('unique-token-id')
  .then(function() {
    console.log('Token Deleted!');
  }).catch(function(err) {
    console.error('OH NO!', err);
  });
```
> This method has [Before / After Middleware](#middleware).

**[[⬆]](#TOC)**

### <a name='count-tokens'>Adding up Tokens</a>

> ### kansas.count(token, optUnits)
>
>    * **token** `string` The token to consume.
>    * **optUnits==** `number=` *Optional* Optionally define how many units to consume.
> *Returns* `Promise(used)` A promise returning the used units a *number*.

Will add up a consume unit and return how many units were used.

**[[⬆]](#TOC)**

### <a name='consuming-tokens'>Consuming Tokens</a>

> ### kansas.consume(token, optUnits)
>
>    * **token** `string` The token to consume.
>    * **optUnits==** `number=` *Optional* Optionally define how many units to consume.
> *Returns* `Promise(remaining)` A promise returning the remaining units a *number*.

Will consume a unit and return the remaining units.

```js
kansas.consume('token')
  .then(function(remaining) {
    console.log('Units remaining:', remaining);
  }).catch(function(err) {
    console.error('No more units!', err);
  });

// consume multiple units
kansas.consume('token', 5)
  .then(function(remaining) {
    console.log('Units remaining:', remaining);
  }).catch(function(err) {
    console.error('No more units!', err);
  });
```

> This method has [Before / After Middleware](#middleware).

**[[⬆]](#TOC)**


### <a name='getByOwnerId-tokens'>Fetch Tokens By Owner Id</a>

> ### kansas.getByOwnerId(ownerId)
>
>    * **ownerId** `string` A string uniquely identifying an owner.
>
> *Returns* `Promise(Array.<Object>)` A promise with an array of [tokenItems](#tokens-item).

Will fetch all tokens based on Owner Id.

```js
kansas.getByOwnerId('hip')
  .then(function(tokens) {
    console.log('Total Tokens:', tokens.length);
  }).catch(function(err) {
    console.error('OH NO!', err);
  });
```

> This method has [Before / After Middleware](#middleware).

**[[⬆]](#TOC)**

### <a name='getUsage'>Get Token Usage</a>

> ### kansas.getUsage(tokenId, optTokenItem)
>
>    * **tokenId** `string` The token id to query.
>    * **optTokenItem** `Object=` Optionally provide the token item if it is available.
>
> *Returns* `Promise(number)` A promise with a number representing the units consumed for the given token.

Get the usage items of a token for the current period.

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
  /** @type {boolean} Indicates the token is a 'count' type */
  count: false;
  /** @type {kansas.model.PeriodBucket.Period} The period */
  period: 'month';
  /** @type {string} Any string uniquely identifying the owner */
  ownerId: 'hip';
  /** @type {string} An ISO 8601 formated date */
  createdOn: '2014-03-01T18:12:15.711Z';

  /** @type {number}  How many units were consumed in this perdiod */
  // This attribute is only available if the item is of type 'count'
  consumed: 10,

  /** @type {number}  How many units remain in this perdiod */
  // This attribute is only available if the item is of type 'limit'
  remaining: 10,
};
```

**[[⬆]](#TOC)**



## <a name='middleware'>Kansas Middleware</a>

Kansas uses the [Middlewarify package](https://github.com/thanpolas/middlewarify) to apply the middleware pattern to its methods. More specifically the [Before/After type of middleware](https://github.com/thanpolas/middlewarify#using-the-before--after-middleware-type) has been applied to the following methods:

  * kansas.set()
  * kansas.get()
  * kansas.del()
  * kansas.consume()
  * kansas.getByOwnerId()
  * kansas.policy.change()

Each of these methods has the Before/After methods so you can add corresponding middleware. All middleware get the same arguments, all *After* middleware receive an extra argument which is the result of the main middleware resolution.

```js
kansas.set.before(function(params) {
  console.log(params.ownerId); // prints 'hip'
});

kansas.set.after(function(params, tokenItem) {
  console.log(tokenItem.token); // the generated token
});

kansas.create({
  ownerId: 'hip',
  policyName: 'free',
});
```

To pass control asynchronously from a middleware you need to return a Promise:

```js
kansas.set.before(function(params) {
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
kansas.on('consume', function(token, consumed, remaining) {
  console.log('Token:', token, ' Consumed:',
    consumed, 'Remaining units:', remaining);
});

```

## <a name='errors'>Kansas Errors</a>

Kansas *signs* all error Objects so you have a better chance of determining what went wrong. All error Constructors are exposed through the `kansas.error` namespace.

### <a name='errors-BaseError'>kansas.error.BaseError</a>

All Kansas errors inherit from this Class. BaseError provides these properties:

* **srcError** `Error=` The original raw error instance may be stored in this property, i.e. errors produced by underlying libraries like the Redis Client.
* **name** `string` The name identifying the error, default is `Kansas Error`.

**[[⬆]](#TOC)**

### <a name='errors-Database'>kansas.error.Database</a>

Errors related to the Redis Database. Provides the following properties

* **srcError** `Error=` The original raw error instance may be stored in this property, i.e. errors produced by underlying libraries like the Redis Client.
* **name** `string` "Database Error"
* **type** `kansas.error.Database.Type` an enum containing these values:
  * `kansas.error.Database.Type.UNKNOWN` **unknown**
  * `kansas.error.Database.Type.REDIS` **redis** Redis originated error.
  * `kansas.error.Database.Type.REDIS_CONNECTION` **redisConnection** Redis connection error.

**[[⬆]](#TOC)**

### <a name='errors-Validation'>kansas.error.Validation</a>

Validation errors. Provides the following properties

* **name** `string` "Validation Error"  

**[[⬆]](#TOC)**

### <a name='errors-TokenNotExists'>kansas.error.TokenNotExists</a>

Token does not exist error. Provides the following properties

* **name** `string` "Token Not Exists Error"  

**[[⬆]](#TOC)**

### <a name='errors-UsageLimit'>kansas.error.UsageLimit</a>

Usage limits errors. Provides the following properties

* **name** `string` "Usage limit exceeded"  

**[[⬆]](#TOC)**

### <a name='errors-howtowork'>How to work with Errors</a>

Use Javascript's `instanceof` to determine the type of error you received.

```js
kansas.consume('unique-token-id')
  .then(onsuccess)
  .catch(function(err) {
    if (err instanceof kansas.error.TokenNotExists) {
      console.log('This token does not exist');
    }
    if (err instanceof kansas.error.UsageLimit) {
      console.log('Usage limit reached!');
    }
  });
```

**[[⬆]](#TOC)**

## <a name='logger'>Kansas Logging Facilities</a>

Kansas uses the [node-logg package](https://github.com/dpup/node-logg) to perform logging. The following logging behavior can only be defined during instatiation:

* **logging** `boolean=` *Optional* Set to `false` to mute all logging, default `true`.
* **console** `boolean=` *Optional* Set to `false` to not log to console, default `true`.
* **logg** `logg=` *Optional* Kansas uses the [logg](https://github.com/dpup/node-logg) package for logging, use this option to inject your own instance of *logg*.
* **logLevel** `number=` *Optional* A value between 0 to 1000 with highest being more important. Kansas uses the following types of logging presented with their corresponding levels:
  * `error` 1000
  * `warn`   800
  * `info`   600
  * `fine`   400
  * `finer`  200
  * `finest` 100


```js
var logg = require('logg');
var kansas = require('kansas');

exports.kansas = kansas({
    logging: true,
    console: false,
    logg: logg,
    level: 100,
});
```

**[[⬆]](#TOC)**

### <a name='logger-events'>Logging Events</a>

All logging messages are emitted as events from kansas. The namespace to listen for is `message`:

```js
kansas.on('message', function(logObj) {
    console.log('Log:', logObj);
});
```

Here is a sample `logObj`:

```js
var logObj = {
  level: 100,
  name: 'cc.model.Token',
  rawArgs: [ 'createActual() :: Init...' ],
  date: 'Tue Apr 16 2013 18:29:52 GMT+0300 (EEST)',
  message: 'createActual() :: Init...',
};
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

  > ### kansas.db.prepopulate()
  >
  > *Returns* `Promise()` A promise.

  Run once each month to populate the usage keys for the  next month.

  > The `db.prepopulate()` method is safe to run multiple times, the effect will be to overwrite any existing next months records. They haven't been consumed so the operation has no effect to your accounting.

  ```js
  kansas.db.prepopulate().then(function() {
    console.log('Done!');
  }).catch(function(err) {
    console.error('An error occurred', err);
  });

  ```
  **[[⬆]](#TOC)**

### <a name='maintenance-nuke'>Nuke the Database</a>

  **WARNING** Irreversible purging of all data **WARNING**

  That said, here's how to nuke the db for your testing purposes:

  > ### kansas.db.nuke(confirm, confirmPrefix)
  >
  >    * **confirm** `string` A confirmation string.
  >    * **confirmPrefix** `string` Confirm the defined prefix.
  >
  > *Returns* `Promise()` A promise.

  The confirmation string is `Yes purge all records irreversably`

  ```js
  var kansas = kansas({prefix: 'test'});

  kansas.db.nuke('Yes purge all records irreversably', 'test').then(function() {
    console.log('Poof Gone!');
  }).catch(function(err) {
    console.error('An error occurred', err);
  });

  ```
  **[[⬆]](#TOC)**
