# The Redis Store Schema

Redis is the main data store for Kansas. 

## Tokens Store

* **Type** [Hash][redis hash]
* **Path** [prefix]:kansas:token:[token]
* **Indexes** `ownerId`

#### Properties

* `string` **token** Unique id.
* `string` **policyId** The policy id.
* `string` **ownerId** Arbitrary string that identifies the owner.
* `string` **createdOn** Date in ISO 8601 Extended Format.

## Policies Store

* **Type** [Hash][redis hash]
* **Path** [prefix]:kansas:policy:[id]
* **Indexes** none

#### Properties

* `string` **id** Unique id.
* `string` **name** Policy name.
* `integer` **maxTokens** Maximum number of tokens.
* `integer` **limit** Maximum requests limit per given period (now only month).
* `string` **createdOn** Date in ISO 8601 Extended Format.


## Usage Store

* **Type** [Hash][redis hash]
* **Path** [prefix]:kansas:usage:[yyyy-mm]:[token]
* **Indexes** none.

#### Properties

* `string` **token** Unique id.
* `string` **policyId** The policy id.
* `integer` **usage** Number of times used (gets increased on each request).
* `integer` **maxApps** Maximum number of apps.
* `integer` **maxTokensPerApp** Maximum tokens per app.
* `integer` **requestLimit** Maximum requests limit per given period (now only month).
* `string` **ownerId** Arbitrary string that identifies the owner.
* `string` **createdOn** Date in ISO 8601 Extended Format.

## Indexes

Indexes, where mentioned, are created atomically using [the plain *String*][redis string] key/value store type, with the Index as key and the store's unique id as value.

* **Type** [String][redis string]
* **Path** [prefix]:kansas:index:token:[indexId]

The `token` part is literraly the string *token* and represents that this is an index of the Tokens store. `indexId` is the value that was stored in the item.

For example for this token record:

```json
{
    "token": "abc",
    "ownerId": "1"

}
```

This key/value index record would be created:

```
kansas:index:token:1 --> "abc"
```

[redis string]: http://redis.io/commands#string
[redis hash]: http://redis.io/commands#hash

