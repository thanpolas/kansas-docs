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
    1. [Consuming Tokens](#consuming-tokens)
    1. [The Token Item](#tokens-item)
  1. [Managing Tokens](#managing-tokens)
  1. [Populating](#populating)
  1. [Database Maintenance](#maintenance)
    1. [Prepopulating Usage Keys](#maintenance-prepopulate)
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

  > `api.connect()` No arguments.
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

  > `api.policy.create(options)`
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

### <a name='policies-read'>Reading a Policy</a>

  > `api.policy.get(policyName)`
  >
  >    * **policyName** `string` The policy name to fetch.
  >
  > *Returns* `Object` [The Policy Item](#policies-item).

  ```js
  var policyItem = api.policy.get('free');
  ```

### <a name='policies-has'>Checking a Policy exists</a>

  > `api.policy.has(policyName)`
  >
  >    * **policyName** `string` The policy name to fetch.
  >
  > *Returns* `boolean` Yes or no.

  ```js
  var policyExists = api.policy.has('free');
  ```


### <a name='policies-change'>Change an Owner's policy</a>

  > `api.policy.change(options)`
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

## <a name='tokens'>Tokens</a>

### <a name='creating-tokens'>Creating Tokens</a>

  > `api.create(options)`
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
  **[[⬆]](#TOC)**

### <a name='get-tokens'>Fetching Tokens</a>

  > `api.get(token)`
  >
  >    * **token** `string` The token's string.
  > *Returns* `Promise(tokenItem)` A promise returning the [*tokenItem* an *Object*](#tokens-item).

  Will fetch a token.


  ```js
  api.policy.change({
    ownerId: 'a-unique-id'
    policyName: 'basic',
  }).then(function(tokenItem) {
    console.log('It Worked!', tokenItem);
    console.log('Here is the token:', tokenItem.token);
  }).catch(function(err) {
    console.error('OH NO!', err);
  });
  ```
  **[[⬆]](#TOC)**

### <a name='consuming-tokens'>Consuming Tokens</a>

  > `api.consume(token, optUnits)`
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
    /** @type {string} An [ISO 8601 formated date ](http://en.wikipedia.org/wiki/ISO_8601) */
    createdOn: '2014-03-01T18:12:15.711Z';
  };
  ```

  **[[⬆]](#TOC)**

## <a name='maintenance'>Database Maintenance</a>

The following Database maintenance tasks are available under the `db` namespace.

### <a name='maintenance-prepopulate'>Prepopulate Usage Keys</a>
  > `api.consume(token, optUnits)`
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
  **[[⬆]](#TOC)**
