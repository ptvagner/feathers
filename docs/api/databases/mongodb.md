---
outline: deep
---

# MongoDB

<Badges>

[![npm version](https://img.shields.io/npm/v/@feathersjs/mongodb.svg?style=flat-square)](https://www.npmjs.com/package/@feathersjs/mongodb)
[![Changelog](https://img.shields.io/badge/changelog-.md-blue.svg?style=flat-square)](https://github.com/feathersjs/feathers/blob/dove/packages/mongodb/CHANGELOG.md)

</Badges>

Support for MongoDB is provided in Feathers via the `@feathersjs/mongodb` database adapter which uses the [MongoDB Client for Node.js](https://www.npmjs.com/package/mongodb). The adapter uses the [MongoDB Aggregation Framework](https://www.mongodb.com/docs/manual/aggregation/), internally, and enables using Feathers' friendly syntax with the full power of [aggregation operators](https://www.mongodb.com/docs/manual/meta/aggregation-quick-reference/). The adapter automatically uses the [MongoDB Query API](https://www.mongodb.com/docs/drivers/node/current/quick-reference/) when you need features like [Collation](https://www.mongodb.com/docs/drivers/node/current/fundamentals/collations/).

```bash
$ npm install --save @feathersjs/mongodb
```

<BlockQuote>

The MongoDB adapter implements the [common database adapter API](./common) and [querying syntax](./querying).

</BlockQuote>

## Setup

There are two typical setup steps for using `@feathersjs/mongodb` in an application:

- connect to the database and
- setup schemas for types and validation.

### Connect to the Database

Before using `@feathersjs/mongodb`, you'll need to create a connection to the database. This example connects to a MongoDB database similar to how the CLI-generated app connects. It uses `app.get('mongodb')` to read the connection string from `@feathersjs/configuration`. The connection string would be something similar to `mongodb://localhost:27017/my-app-dev` for local development or one provided by your database host.

Once the connection attempt has been started, the code uses `app.set('monodbClient', mongoClient)` to store the connection promise back into the config, which allows it to be looked up when initializing individual services.

```ts
import { MongoClient } from 'mongodb'
import type { Db } from 'mongodb'
import type { Application } from './declarations'

export const mongodb = (app: Application) => {
  const connection = app.get('mongodb')
  const database = new URL(connection).pathname.substring(1)
  const mongoClient = MongoClient.connect(connection).then((client) => client.db(database))
  app.set('mongodbClient', mongoClient)
}

declare module './declarations' {
  interface Configuration {
    mongodbClient: Promise<Db>
  }
}
```

### Setup the Schema & Types

To take full advantage of the new TypeScript features in Feathers v5, we can create schema for our service's data types. This example shows how to use `@feathersjs/typebox` to create schemas and types for data and query types. This is the same as generated by the CLI, but the resolvers have been removed for brevity.

```ts
import { Type, querySyntax } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'

// Main data model schema
export const messagesSchema = Type.Object(
  {
    _id: Type.String(),
    text: Type.String()
  },
  { $id: 'Messages', additionalProperties: false }
)
export type Messages = Static<typeof messagesSchema>

// Schema for creating new entries
export const messagesDataSchema = Type.Pick(messagesSchema, ['name'], {
  $id: 'MessagesData',
  additionalProperties: false
})
export type MessagesData = Static<typeof messagesDataSchema>

// Schema for allowed query properties
export const messagesQueryProperties = Type.Pick(messagesSchema, ['_id', 'name'], {
  additionalProperties: false
})
export const messagesQuerySchema = querySyntax(messagesQueryProperties)
export type MessagesQuery = Static<typeof messagesQuerySchema>
```

### Schemas vs MongoDB Validation

In Feathers v5 (Dove) we added support for Feathers Schema, which performs validation and provides TypeScript types. Recent versions of MongoDB include support for JSON Schema validation at the database server. Most applications will benefit from using Feathers Schema for the following reasons.

- Feathers Schema's TypeBox integration makes JSON Schema so much easier to read and write.
- You get TypeScript types for free once you've defined your validation rules, using `TypeBox` or `json-schema-to-ts`
- All configuration is done in code, reducing the time to prototype/setup/launch. With MongoDB's built-in validation, you essentially add another "DevOps" step before you can use the database.
- Support for JSON Schema draft 7. MongoDB's validation is based on version 4.
- Feathers Schema don't have to wait for a round-trip to the database to validate the data.
- Feathers Schema can be used in the browser or on the server.

MongoDB's built-in validation does have built-in support for `bsonType` to force data to be stored as a specific BSON type once it passes validation. There's nothing keeping you from using both solutions together. It's not a use case that's documented, here.

## API

### `service(options)`

Returns a new service instance initialized with the given options. The following example extends the `MongoDBService` class using the schema examples from earlier on this page. It then uses the `mongodbClient` from the app configuration and provides it to the `Model` option, which is passed to the new `MessagesService`.

```ts
import type { Params } from '@feathersjs/feathers'
import { MongoDBService } from '@feathersjs/mongodb'
import type { MongoDBAdapterParams, MongoDBAdapterOptions } from '@feathersjs/mongodb'

import type { Application } from '../../declarations'
import type { Messages, MessagesData, MessagesQuery } from './messages.schema'

export interface MessagesParams extends MongoDBAdapterParams<MessagesQuery> {}

export class MessagesService<ServiceParams extends Params = MessagesParams> extends MongoDBService<
  Messages,
  MessagesData,
  ServiceParams
> {}

export const messages = (app: Application) => {
  const options: MongoDBAdapterOptions = {
    paginate: app.get('paginate'),
    Model: app.get('mongodbClient').then((db) => db.collection('messages'))
  }
  app.use('messages', new MessagesService(options), {
    methods: ['find', 'get', 'create', 'update', 'patch', 'remove'],
    events: []
  })
}
```

Here's an overview of the `options` object:

**Options:**

- `Model {Promise<MongoDBCollection>}` (**required**) - The MongoDB collection instance
- `id {string}` (_optional_, default: `'_id'`) - The name of the id field property. By design, MongoDB will always add an `_id` property.
- `disableObjectify {boolean}` (_optional_, default `false`) - This will disable the objectify of the id field if you want to use normal strings
- `events {string[]}` (_optional_) - A list of [custom service events](/api/events.html#custom-events) sent by this service
- `paginate {Object}` (_optional_) - A [pagination object](/api/databases/common.html#pagination) containing a `default` and `max` page size
- `filters {Object}` (_optional_) - An object of additional filter parameters to allow (e..g `{ $customQueryOperator: true }`). See [Filters](/api/databases/querying.md#filters)
- `operators {string[]}` (_optional_) - A list of additional query parameters to allow (e..g `[ '$regex', '$geoNear' ]`) See [Operators](/api/databases/querying.md#operators)
- `multi {string[]|true}` (_optional_) - Allow `create` with arrays and `update` and `remove` with `id` `null` to change multiple items. Can be `true` for all methods or an array of allowed methods (e.g. `[ 'remove', 'create' ]`)
- `useEstimatedDocumentCount {boolean}` (_optional_, default `false`) - If `true` document counting will rely on `estimatedDocumentCount` instead of `countDocuments`

### `aggregateRaw(params)`

The `find` method has been split into separate utilities for converting params into different types of MongoDB requests. By default, requests are processed by this method and are run through the MongoDB Aggregation Pipeline. This method returns a raw MongoDB Cursor object, which can be used to perform custom pagination or in custom server scripts, if desired.

### `findRaw(params)`

The `find` method has been split into separate utilities for converting params into different types of MongoDB requests. When `params.mongodb` is used, the `findRaw` method is used to retrieve data using `params.mongodb` as the `FindOptions` object. This method returns a raw MongoDB Cursor object, which can be used to perform custom pagination or in custom server scripts, if desired.

### `makeFeathersPipeline(params)`

`makeFeathersPipeline` takes a set of Feathers params and converts them to a pipeline array, ready to pass to `collection.aggregate`. This utility comprises the bulk of the `aggregateRaw` functionality, but does not use `params.pipeline`.

### Custom Params

The `@feathersjs/mongodb` adapter utilizes two custom params which control adapter-specific features: `params.pipeline` and `params.mongodb`.

#### params.pipeline

This is a test

#### params.mongodb

When making a [service method](/api/services.md) call, `params` can contain an `mongodb` property (for example, `{upsert: true}`) which allows modifying the options used to run the MongoDB query.

The adapter will automatically switch to use the MongoClient's`collection.find` method when you use `params.mongodb`.

## Aggregation Pipeline

In Feathers v5 Dove, we added support for the full power of MongoDB's Aggregation Framework and blends it seamlessly with the familiar Feathers Query syntax. All `find` queries now use the Aggregation Framework, by default.

The Aggregation Framework is accessed through the mongoClient's `collection.aggregate` method, which accepts an array of "stages". Each stage contains an operator which describes an operation to apply to the previous step's data. Each stage applies the operation to the results of the previous step. It’s now possible to perform any of the [Aggregation Stages](https://www.mongodb.com/docs/upcoming/reference/operator/aggregation-pipeline/) like `$lookup` and `$unwind`, integration with the normal Feathers queries.

Here's how it works with the operators that match the Feathers Query syntax. Let's convert the following Feathers query:

```ts
const query = {
  text: { $regex: 'feathersjs', $options: 'igm' },
  $sort: { createdAt: -1 },
  $skip: 0,
  $limit: 10
}
```

The above query looks like this when converted to aggregation pipeline stages:

```ts
;[
  // returns the set of records containing the word "feathersjs"
  { $match: { text: { $regex: 'feathersjs', $options: 'igm' } } },
  // Sorts the results of the previous step by newest messages, first.
  { $sort: { createdAt: -1 } },
  // Skips the first 20 records of the previous step
  { $skip: 20 },
  // returns the next 10 records
  { $limit: 10 }
]
```

### Pipeline Queries

You can use the `params.pipeline` array to append additional stages to the query. This next example uses the `$lookup` operator together with the `$unwind` operator to populate a `user` attribute onto each message based on the message's `userId` property.

```ts
const result = await app.service('messages').find({
  query: { $sort: { name: 1 } },
  pipeline: [
    {
      $lookup: {
        from: 'users',
        localField: 'userId',
        foreignField: '_id',
        as: 'user'
      }
    },
    { $unwind: { path: '$user' } }
  ],
  paginate: false
})
```

### Aggregation Stages

In the example, above, the `query` is added to the pipeline, first. Then additional stages are added in the `pipeline` option:

- The `$lookup` stage creates an array called `user` which contains any matches in `message.userId`, so if `userId` were an array of ids, any matches would be in the `users` array. However, in this example, the `userId` is a single id, so...
- The `$unwind` stage turns the array into a single `user` object.

The above is like doing a join, but without the data transforming overhead like you'd get with an SQL JOIN. If you have properly applied index to your MongoDB collections, the operation will typically execute extremely fast for a reasonable amount of data.

A couple of other notable query stages:

- `$graphLookup` lets you recursively pull in a tree of data from a single collection.
- `$search` lets you do full-text search on fields

All stages of the pipeline happen directly on the MongoDB server.

Read through the full list of supported stages [in the MongoDB documentation](https://www.mongodb.com/docs/upcoming/reference/operator/aggregation-pipeline/).

### The `$feathers` Stage

The previous section showed how to append stages to a query using `params.pipeline`. Well, `params.pipeline` also supports a custom `$feathers` operator/stage which allows you to specify exactly where in the pipeline the Feathers Query gets injected.

### Example: Proxy Permissions

Imagine a scenario where you want to query the `pages` a user can edit by referencing a `permissions` collection to find out which pages the user can actually edit. Each record in the `permissions` record has a `userId` and a `pageId`. So we need to find and return only the pages to which the user has access by calling `GET /pages` from the client.

We could put the following query in a hook to pull the correct `pages` from the database in a single query THROUGH the permissions collection. Remember, the request is coming in on the `pages` service, but we're going to query for pages `through` the permissions collection. Assume we've already authenticated the user, so the user will be found at `context.params.user`.

```ts
// Assume this query on the client
const pages = await app.service('pages').find({ query: {} })

// And put this query in a hook to populate pages "through" the permissions collection
const result = await app.service('permissions').find({
  query: {},
  pipeline: [
    // query all permissions records which apply to the current user
    {
      $match: { userId: context.params.user._id }
    },
    // populate the pageId onto each `permission` record, as an array containing one page
    {
      $lookup: {
        from: 'pages',
        localField: 'pageId',
        foreignField: '_id',
        as: 'page'
      }
    },
    // convert the `page` array into an object, so now we have an array of permissions with permission.page on each.
    {
      $unwind: { path: '$page' }
    },
    // Add a permissionId to each page
    {
      $addFields: {
        'page.permissionId': '$_id'
      }
    },
    // discard the permission and only keep the populated `page`, and bring it top level in the array
    {
      $replaceRoot: { newRoot: '$page' }
    },
    // apply the feathers query stages to the aggregation pipeline.
    // now the query will apply to the pages, since we made the pages top level in the previous step.
    {
      $feathers: {}
    }
  ],
  paginate: false
})
```

Notice the `$feathers` stage in the above example. It will apply the query to that stage in the pipeline, which allows the query to apply to pages even though we had to make the query through the `permissions` service.

If we were to express the above query with JavaScript, the final result would the same as with the following example:

```ts
// perform a db query to get the permissions
const permissions = await context.app.service('permissions').find({
  query: {
    userId: context.params.user._id
  },
  paginate: false
})
// make a list of pageIds
const pageIds = permissions.map((permission) => permission.pageId)
// perform a db query to get the pages with matching `_id`
const pages = await context.app.service('pages').find({
  query: {
    _id: {
      $in: pageIds
    }
  },
  paginate: false
})
// key the permissions by pageId for easy lookup
const permissionsByPageId = permissions.reduce((byId, current) => {
  byId[current.pageId] = current
  return byId
}, {})
// Add the permissionId to each `page` record.
const pagesWithPermissionId = pages.map((page) => {
  page.permissionId = permissionByPageId[page._id]._id
  return page
})
// And now apply the original query, whatever the client may have sent, to the pages.
// It might require another database query
```

Both examples look a bit complex, but te one using aggregation stages will be much quicker because all stages run in the database server. It will also be quicker because it all happens in a single database query!

One more obstacle for using JavaScript this way is that if the user's query changed (from the front end), we would likely be required to edit multiple different parts of the JS logic in order to correctly display results. With the pipeline example, above, the query is very cleanly applied.

## Transactions

You can utilize [MongoDB Transactions](https://docs.mongodb.com/manual/core/transactions/) by passing a `session` with the `params.mongodb`:

```js
import { ObjectID } from 'mongodb'

export default async app => {
  app.use('/fooBarService', {
    async create(data) {
      // assumes you have access to the mongoClient via your app state
      let session = app.mongoClient.startSession()
      try {
        await session.withTransaction(async () => {
            let fooID = new ObjectID()
            let barID = new ObjectID()
            app.service('fooService').create(
              {
                ...data,
                _id: fooID,
                bar: barID,
              },
              { mongodb: { session } },
            )
            app.service('barService').create(
              {
                ...data,
                _id: barID
                foo: fooID
              },
              { mongodb: { session } },
            )
        })
      } finally {
        await session.endSession()
      }
    }
  })
}
```

## Collation

This adapter includes support for [collation and case insensitive indexes available in MongoDB v3.4](https://docs.mongodb.com/manual/release-notes/3.4/#collation-and-case-insensitive-indexes). Collation parameters may be passed using the special `collation` parameter to the `find()`, `remove()` and `patch()` methods.

### Example: Patch records with case-insensitive alphabetical ordering

The example below would patch all student records with grades of `'c'` or `'C'` and above (a natural language ordering). Without collations this would not be as simple, since the comparison `{ $gt: 'c' }` would not include uppercase grades of `'C'` because the code point of `'C'` is less than that of `'c'`.

```js
const patch = { shouldStudyMore: true };
const query = { grade: { $gte: 'c' } };
const collation = { locale: 'en', strength: 1 };
students.patch(null, patch, { query, collation }).then( ... );
```

### Example: Find records with a case-insensitive search

Similar to the above example, this would find students with a grade of `'c'` or greater, in a case-insensitive manner.

```js
const query = { grade: { $gte: 'c' } };
const collation = { locale: 'en', strength: 1 };
students.find({ query, collation }).then( ... );
```

For more information on MongoDB's collation feature, visit the [collation reference page](https://docs.mongodb.com/manual/reference/collation/).

## Querying

Additionally to the [common querying mechanism](https://docs.feathersjs.com/api/databases/querying.html) this adapter also supports [MongoDB's query syntax](https://docs.mongodb.com/v3.2/tutorial/query-documents/) and the `update` method also supports MongoDB [update operators](https://docs.mongodb.com/v3.2/reference/operator/update/).

> **Important:** External query values through HTTP URLs may have to be converted to the same type stored in MongoDB in a before [hook](https://docs.feathersjs.com/api/hooks.html) otherwise no matches will be found. Websocket requests will maintain the correct format if it is supported by JSON (ObjectIDs and dates still have to be converted).

For example, an `age` (which is a number) a hook like this can be used:

```js
const ObjectID = require('mongodb').ObjectID

app.service('users').hooks({
  before: {
    find(context) {
      const { query = {} } = context.params

      if (query.age !== undefined) {
        query.age = parseInt(query.age, 10)
      }

      context.params.query = query

      return Promise.resolve(context)
    }
  }
})
```

Which will allows queries like `/users?_id=507f1f77bcf86cd799439011&age=25`.

## Validate Data

There are two ways 

Since MongoDB uses special binary types for stored data, Feathers uses Schemas to validate MongoDB data and resolvers to convert types. Attributes on MongoDB services often require specitying two schema formats:

- Object-type formats for data pulled from the database and passed between services on the API server. The MongoDB driver for Node converts binary data into objects, like `ObjectId` and `Date`.
- String-type formats for data passed from the client.

You can convert values to their binary types to take advantage of performance enhancements in MongoDB. The following sections show how to use Schemas and Resolvers for each MongoDB binary format.

### Shared Validators

The following example shows how to create two custom validator/type utilities:

- An ObjectId validator which handles strings or objects. Place the following code inside `src/schema/shared.ts`.
- A Date validator

```ts
import { Type } from '@feathersjs/typebox'

export const ObjectId = () => Type.Union([
  Type.String({ format: 'objectid' }),
  Type.Object({}, { additionalProperties: true }),
])
export const NullableObjectId = () => Type.Union([ObjectId(), Type.Null()])

export const Date = () => Type.Union([
  Type.String({ format: 'date-time' }),
  Type.Date(),
])
```

Technically, `ObjectId` in the above example could be improved since it allows any object to pass validation. A query with a bogus id fail only when it reaches the database. Since objects can't be passed from the client, it's a situation which only occurs with our server code, so it will suffice.

### ObjectIds

With the [shared validators](#shared-validators) in place, you can specify an `ObjectId` type in your TypeBox Schemas:

```ts
import { Type } from '@feathersjs/typebox'
import { ObjectId } from '../../schemas/shared'

const userSchema = Type.Object(
  {
    _id: ObjectId(),
    orgIds: Type.Array( ObjectId() ),
  },
  { $id: 'User', additionalProperties: false }
)
```

### Dates

The standard format for transmitting dates between client and server is [ISO8601](https://www.rfc-editor.org/rfc/rfc3339#section-5.6), which is a string representation of a date.

While it's possible to use `$gt` (greater than) and `$lt` (less than) queries on string values, performance will be faster for dates stored as date objects. This means you'd use the same code as the previous example, followed by a resolver or a hook to convert the values in `context.data` and `context.query` into actual Dates.

With the [shared validators](#shared-validators) in place, you can use the custom `Date` type in your TypeBox Schemas:

```ts
import { Type } from '@feathersjs/typebox'
import { ObjectId, Date } from '../../schemas/shared'

const userSchema = Type.Object(
  {
    _id: ObjectId(),
    createdAt: Date(),
  },
  { $id: 'User', additionalProperties: false }
)
```

## Convert Data

It's possible to convert data by either customizing AJV or using resolvers.  Converting with AJV is currently the simplest solution, but it uses extended AJV features, outside of the JSON Schema standard.

### Convert With AJV

The FeathersJS CLI creates validators in the `src/schema/validators` file.  It's possible to enable the AJV validators to convert certain values using AJV keywords. This example works for both [TypeBox](/api/schema/typebox) and [JSON Schema](/api/schema/schema):

#### Create Converters

```ts
import type { AnySchemaObject } from 'ajv'
import { Ajv } from '@feathersjs/schema'
import { ObjectId } from 'mongodb'

// `convert` keyword.
const keywordConvert = {
  keyword: 'convert',
  type: 'string',
  compile(schemaVal: boolean, parentSchema: AnySchemaObject) {
    if (!schemaVal) return () => true

    // Convert date-time string to Date
    if (['date-time', 'date'].includes(parentSchema.format)) {
      return function (value: string, obj: any) {
        const { parentData, parentDataProperty } = obj
        parentData[parentDataProperty] = new Date(value)
        return true
      }
    }
    // Convert objectid string to ObjectId
    else if (parentSchema.format === 'objectid') {
      return function (value: string, obj: any) {
        const { parentData, parentDataProperty } = obj
        parentData[parentDataProperty] = new ObjectId(value)
        return true
      }
    }
    return () => true
  },
} as const

// `objectid` formatter
const formatObjectId = {
  type: 'string',
  validate: (id: string | ObjectId) => {
    if (ObjectId.isValid(id)) {
      if (String(new ObjectId(id)) === id) return true
      return false
    }
    return false
  },
} as const

export function addConverters(validator: Ajv) {
  validator.addKeyword(keywordConvert)
  validator.addFormat('objectid', formatObjectId)
}
```

#### Apply to AJV

You can then add converters to the generated validators by importing the `addConverters` utility in the `validators.ts` file:

```ts{3,30-32}
import { Ajv, addFormats } from '@feathersjs/schema'
import type { FormatsPluginOptions } from '@feathersjs/schema'
import { addConverters } from './converters'

const formats: FormatsPluginOptions = [
  'date-time',
  'time',
  'date',
  'email',
  'hostname',
  'ipv4',
  'ipv6',
  'uri',
  'uri-reference',
  'uuid',
  'uri-template',
  'json-pointer',
  'relative-json-pointer',
  'regex',
]

export const dataValidator = addFormats(new Ajv({}), formats)

export const queryValidator = addFormats(
  new Ajv({
    coerceTypes: true,
  }),
  formats,
)

addConverters(dataValidator)
addConverters(queryValidator)
```

#### Update Shared Validators

We can now update the [shared validators](#shared-validators) to also convert data.

```ts{5}
import { Type } from '@feathersjs/typebox'

export const ObjectId = () => Type.Union([
  Type.String({ format: 'objectid', convert: true }),
  Type.Object({}, { additionalProperties: true }),
])
export const NullableObjectId = () => Type.Union([ObjectId(), Type.Null()])

export const Date = () => Type.Union([
  Type.String({ format: 'date-time', convert: true }),
  Type.Date(),
])
```

Now when we use the shared validators (as shown in the [ObjectIds](#objectids) and [Dates](#dates) sections) AJV will also convert the data to the correct type.

### Convert with Resolvers

In place of [converting data with AJV](#convert-with-ajv), you can also use resolvers to convert data.

#### ObjectId Resolvers

MongoDB uses object ids as primary keys and for references to other documents. To a client they are represented as strings and to convert between strings and object ids, the following [property resolver](../schema/resolvers.md) helpers can be used.

#### resolveObjectId

`resolveObjectId` resolves a property as an object id. It can be used as a direct property resolver or called with the original value.

```ts
import { resolveObjectId } from '@feathersjs/mongodb'

export const messageDataResolver = resolve<Message, HookContext>({
  properties: {
    userId: resolveObjectId
  }
})

export const messageDataResolver = resolve<Message, HookContext>({
  properties: {
    userId: async (value, _message, context) => {
      // If the user is an admin, allow them to create messages for other users
      if (context.params.user.isAdmin && value !== undefined) {
        return resolveObjectId(value)
      }
      // Otherwise associate the record with the id of the authenticated user
      return context.params.user._id
    }
  }
})
```

#### resolveQueryObjectId

`resolveQueryObjectId` allows to query for object ids. It supports conversion from a string to an object id as well as conversion for values from the [$in, $nin and $ne query syntax](./querying.md).

```ts
import { resolveQueryObjectId } from '@feathersjs/mongodb'

export const messageQueryResolver = resolve<MessageQuery, HookContext>({
  properties: {
    userId: resolveQueryObjectId
  }
})
```

## Common Mistakes

Here are a couple of errors you might run into while using validators.

### unknown keyword: "convert"

You'll see an error like `"Error: strict mode: unknown keyword: "convert"` in a few scenarios:

- You fail to [Pass the Custom AJV Instance to every `schema`](#pass-the-custom-ajv-instance-to-schema). If you're using a custom AJV instance, be sure to provide it to **every** place where you call `schema()`.
- You try to use custom keywords in your schema without registering them, first.
- You make a typo in your schema. For example, it's common to forget to accidentally mis-document arrays and collapse the item `properties` up one level.

### unknown format "date-time"

You'll see an error like `Error: unknown format "date-time" ignored in schema at path "#/properties/createdAt"` in a few scenarios.

- You're attempting to use a formatter not built into AJV.
- You fail to [Pass the Custom AJV Instance to every `schema`](#pass-the-custom-ajv-instance-to-schema). If you're using a custom AJV instance, be sure to provide it to **every** place where you call `schema()`.

## Search

There are two ways to perform search queries with MongoDB: 

- Perform basic Regular Expression matches using the `$regex` filter.
- Perform full-text search using the `$search` filter.

### Basic Regex Search

You can perform basic search using regular expressions with the `$regex` operator.  Here's an example query.

```js
{
  text: { $regex: 'feathersjs', $options: 'igm' },
}
```

TODO: Show how to customize the query syntax to allow the `$regex` and `$options` operators.

### Full-Text Search

See the MongoDB documentation for instructions on performing full-text search using the `$search` operator:

- Perform [full-text queries on self-hosted MongoDB](https://www.mongodb.com/docs/manual/core/link-text-indexes/).
- Perform [full-text queries on MongoDB Atlas](https://www.mongodb.com/docs/atlas/atlas-search/) (MongoDB's first-party hosted database).
- Perform [full-text queries with the MongoDB Pipeline](https://www.mongodb.com/docs/manual/tutorial/text-search-in-aggregation/)
