# Kansas API Documentation

*Token based rate limited API to go!*

## <a name='TOC'>Table of Contents</a>

  1. [Configuration](#configuration)
  1. [Database Connection](#connect)
  1. [Policies](#policies)
  1. [Creating Tokens](#creating-tokens)
  1. [Consuming Tokens](#consuming-tokens)
  1. [Managing Tokens](#managing-tokens)
  1. [Populating](#populating)

## <a name='configuration'>Configuration</a>

  Kansas can be configured at instantiation and using the `setup()` method.

  ```js
  var kansas = require('kansas');

  var api = kansas({/* options */});

  // this will work too
  api.setup({/* options */});
  ```

  ### Available Options

  * **prefix** `string` *Optional* A string to prefix all keys on Redis.
  * **redis** `Object` *Optional* Redis connection options
    * **port** `string|number=` *Optional* Define redis port, default `6379`.
    * **host** `string=` *Optional* The host to connect to, default is `"localhost"`.
    * **pass**  `string=` *Optional* Password to connect with.
    * **redisOptions** `Object=` *Optional* More advanced options to pass directly to *redis-client*.


  ### Example

  ```js
  var kansas = require('kansas');

  var api = kansas();

  api.setup({
      port: 7777,
      host: 'some.redis.host.com',
      password: 'secret',
  });
  ```

  **[[⬆]](#TOC)**

## <a name='connect'>Connecting to Database</a>

  > `api.create()` No arguments.
  >
  > *Returns* `Promise` A Promise.

  Before you start a connection Kansas will refuse to perform any task. Connecting to redis is plain Promise returning method, `connect()`.

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

  Policies are stored in memory. After many iterations it became apparent that this is the most convenient way to work with policies. You define and maintain them inside your application, they are shared between multiple cores and exist in the memory saving on needless database reads. After all, typically an application is not expected to have more than 10 policies.

  ### Creating policies

  > `api.policy.create(options)`
  >
  >    * **options** `Object` A dictionary with the following options.
  >       * **name** `string` The policy's name, uniquely identifying it.
>       * **maxTokens** `number` Maximum number of tokens each owner is allowed to have.
>       * **limit** `number` The maximum number of API calls (units) per period.
  >
  > *Returns* `Object` The Policy Item.



  To create a policy you need to define three parameters:

  ```js
  api.policy.create({
    name: 'free',
    maxTokens: 3, // max tokens per owner
    limit: 100 // max usage units per period
  });
  ```

### Reading a Policy

  > `api.policy.get(policyName)`
  >
  >    * **policyName** `string` The policy name to fetch.
  >
  > *Returns* `Object` The Policy Item.

  ```js
  var policyItem = api.policy.get('free');
  ```

### Checking a Policy exists

  > `api.policy.has(policyName)`
  >
  >    * **policyName** `string` The policy name to fetch.
  >
  > *Returns* `boolean` Yes or no.

  ```js
  var policyExists = api.policy.has('free');
  ```


### Change an Owner's policy

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


## <a name='creating-tokens'>Creating Tokens</a>

  > `api.create(options)`
  >
  >    * **options** `Object` A dictionary with the following key/value pairs:
  >      * **ownerId** `string` The owner's id.
  >      * **policyName** `string` The new policy.
  >
  > *Returns* `Promise(tokenItem)` A promise returning the *tokenItem* an *Object*.

  Creates a tokens and populates usage keys and indexes.


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

## <a name='consuming-tokens'>Consuming Tokens</a>

  > `api.consume(token, optUnits)`
  >
  >    * **token** `string` The token to consume.
  >    * **optUnits=** `number` *Optional* Optionally define how many units to consume.
  > *Returns* `Promise(remaining)` A promise returning the remaining units a *number*.

  Will consume a unit and return the remaining units.

  ```js
  api.consume('token')
  .then(function(remaining) {
    console.log('Units remaining:', remaining);
  }).catch(function(err) {
    console.error('No more units!', err);
  });
  ```
  **[[⬆]](#TOC)**
