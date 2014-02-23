# The Redis Store Schema

Redis is the main data store for Kansas. Indexes, where mentioned are created using [the plain *String*][redis string] key/valye store type, with the Index as key and the store's unitue id as value.

## Applications Store

* **Type** Hash
* **Path** [prefix]:application:[id]
* **Indexes** `userId`

#### Properties

* **Type** `string` **id** Unique id.
* **Type** `string` **hostname** Bare hostname without protocol or path.
* **Type** `string` **policyId** The policy id.
* **Type** `string` **ownerId** Arbitrary string that identifies the owner.
* **Type** `string` **createdOn** Date in ISO 8601 Extended Format.

## Tokens Store

* **Type** Hash
* **Path** [prefix]:token:[token]
* **Indexes** `applicationId`, 

#### Properties

* **Type** `string` **token** Unique id.
* **Type** `string` **applicationId** Application relation.
* **Type** `string` **hostname** Bare hostname without protocol or path.
* **Type** `string` **policyId** The policy id.
* **Type** `string` **ownerId** Arbitrary string that identifies the owner.
* **Type** `string` **createdOn** Date in ISO 8601 Extended Format.


[redis string]: http://redis.io/commands#string
