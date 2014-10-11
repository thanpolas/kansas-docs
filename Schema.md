# The Redis Store Schema

Redis is the main data store for Kansas.

## Tokens Keys

* **Type** [Hash][redis hash]
* **Path** [prefix]:kansas:token:[token]
* **Indexes** `ownerId`
* **Value** `Object` (a hash)

```
test:kansas:token:dsakj3nfbsDeTFK1s12rpLKsMcyTE4sW
```

## Usage Keys

* **Type** [Strings][redis string]
* **Path** [prefix]:kansas:usage:[yyyy-mm-dd]:[token]
* **Indexes** none.
* **Value** `number` A number representing usage units left.

```
test:kansas:usage:2014-03-01:dsakj3nfbsDeTFK1s12rpLKsMcyTE4sW
```

## Count Keys

* **Type** [Strings][redis string]
* **Path** [prefix]:kansas:usage:[yyyy-mm-dd]:count:[token]
* **Indexes** none.
* **Value** `number` A number representing units used.

```
test:kansas:usage:2014-03-01:count:dsakj3nfbsDeTFK1s12rpLKsMcyTE4sW
```

## Indexes

Indexes are created atomically using [the plain *String*][redis string] key/value store type, with the Index as key and the store's unique id as value.

* **Type** [Set][redis set]
* **Path** [prefix]:kansas:index:[index]
* **Value** `string` The token

```
test:kansas:index:token:owner-unique-id
```

[redis string]: http://redis.io/commands#string
[redis set]: http://redis.io/commands#set
[redis hash]: http://redis.io/commands#hash
