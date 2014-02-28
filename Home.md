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
  > *Returns* A Promise.

  Before you start a connection Kansas will refuse to perform any task. Connecting to redis is plain Promise returning method, `connect()`.

  ```js
  var kansas = require('kansas');

  var api = kansas();

  api.connect().then(function() {
    console.log('Connected!');
  }).catch(function(err){
    console.log('Opts, something went wrong:', err);
  });
  ```
  **[[⬆]](#TOC)**


## <a name='policies'>Policies</a>

  Policies are stored in memory. After many iterations it became apparent that this is the most convenient way to work with policies. You define and maintain them inside your application, they are shared between multiple cores and exist in the memory saving on needless database reads. After all, typically an application is not expected to have more than 10 policies.

  ### Creating policies

  To create a policy you need to define three parameters:

  ```js
  api.policy.create({
    name: 'free',
    maxTokens: 3, // max tokens per owner
    limit: 100 // max usage units per period
  });
  ```

  ###

  **[[⬆]](#TOC)**


## <a name='creating-tokens'>Creating Tokens</a>

  To create a token you need an `ownerId` and a `policyName`.


  ```js
  api.create({
    policyName: 'free',
    ownerId: 'a-unique-id',
  });
  ```
  **[[⬆]](#TOC)**
