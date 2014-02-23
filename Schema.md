# The Redis Store Schema

Redis is the main data store for Kansas. Indexes, where mentioned are created using [the plain *String*][redis string] key/valye store type, with the Index as key and the store's unitue id as value.

## Applications Store

* **Type** Hash
* **Path** [prefix]:application:[id]
* **Indexes** `userId`

#### Properties

* `string` **id** Unique id.
* `string` **hostname** Bare hostname without protocol or path.
* `string` **policyId** The policy id.
* `string` **ownerId** Arbitrary string that identifies the owner.
* `string` **createdOn** Date in ISO 8601 Extended Format.

## Tokens Store

* **Type** Hash
* **Path** [prefix]:token:[token]
* **Indexes** `applicationId`, 

#### Properties

* `string` **token** Unique id.
* `string` **applicationId** Application relation.
* `string` **hostname** Bare hostname without protocol or path.
* `string` **policyId** The policy id.
* `string` **ownerId** Arbitrary string that identifies the owner.
* `string` **createdOn** Date in ISO 8601 Extended Format.

[redis string]: http://redis.io/commands#string
