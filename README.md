# Much ado about Indexing

This is a presentation about MongoDB Indexing. The basic content is found below. The [presentation](http://revealme.herokuapp.com/yanatan16/mongodb-indexing-presentation/master/presentation.html#/) is hosted in github but served via a [small app](https://github.com/yanatan16/revealme) I built.

_by [Jon Eisen](http://joneisen.me)_

## Overview

Indexes are awesome because:
- take a linear search and make it logarithmic (i.e. search through 8 items in 3 steps instead of 8).
- eliminate in-memory sorts.
- reduce hard drive lookups and page faulting.

## Basic Commands

- `db.collection.ensureIndex(indexed_fields, [options]);` Create an index.
- `db.collection.getIndexes();` See indexes.
- `db.collection.dropIndex(indexed_fields);` Remove an index.
- `db.collection.find(find_fields).explain();` See what indexes a query uses.
- `db.collection.find(fields).hint(index_or_name);` Force a query to use a certain index

## Basics on indexes

- `ensureIndex({_id: 1}, {unique: true})` is always (mostly) true
- A query can only use one index (exception: $or queries)
- Order of keys matters for range and sort queries.

## Indexing basic queries - equality

```javascript
db.subunits.find({ id: 'ZMB' }).explain();
{
  "cursor" : "BasicCursor", // no index!
	"n" : 1,
	"nscanned" : 170, // Searched 170 objects to find 1
}
db.subunits.ensureIndex({ id: 1 }, { unique: true });
db.subunits.find({ id: 'ZMB' }).explain();
{
  "cursor" : "BtreeCursor id_1", // yay index worked!
	"n" : 1,
	"nscanned" : 1,
}
```

## Indexing basic queries - multi-key

```javascript
db.subunits.find({ type: 'Polygon', rkey: 40 });
db.subunits.ensureIndex({ type: 1, rkey: 1 }, { name: 'my_custom_name' });

// Works with a prefix subset
db.subunits.find({ type:'Polygon' }) // indexed

db.subunits.find({ rkey: 40 }) // not-indexed!
db.subunits.find({ rkey: 40 }).hint('my_custom_name'); // indexed!
```

- Prefix indexes are optimal secondary indexes.
- Subset (non-prefix) are non-optimalsecondary indexes that must be hinted.

## Indexing a complex query - equality, range, sort

```javascript
db.subunits.find({ type: 'Polygon', rkey: { $gt: 40 }}).sort({ 'properties.name': -1 }).explain();
{
  "cursor" : "BtreeCursor type_1_rkey_1", // used the previous index
	"scanAndOrder" : true, // OH NO! in-memory sort!
}
db.subunits.ensureIndex({ type: 1, rkey: 1, 'properties.name': -1 }); // conjectured index...
db.subunits.find({ type: 'Polygon', rkey: { $gt: 40 }}).sort({ 'properties.name': -1 }).explain();
{
  "cursor" : "BtreeCursor type_1_rkey_1", // wait what? why didn't it use our index?
	"scanAndOrder" : true, // still in-memory sorting
}
```

## Really indexing a complex query

Index *Equality*, then *Sorts* (in order with correct _directions_), then *Range queries*.

```javascript
db.subunits.ensureIndex({ type: 1, 'properties.name': -1, rkey: 1}); // the Correct index
db.subunits.find({ type: 'Polygon', rkey: { $gt: 40 }}).sort({ 'properties.name': -1 }).explain();
{
  "cursor" : "BtreeCursor type_1_rkey_1", // what happened?
  "scanAndOrder" : true, // still in-memory sorting
}
```

- Every time we use `.explain()`, the _query optimizer_ finds the best (fastest) index for that query.
- There is a trade-off of index lookup speed versus in-memory sort speed.
- In this case lookup speed won, but not always. Use `.hint(index_keys).explain()` to compare.

## Sparse Indexes

- Like regular secondary indexes, but they don't include references to documents without that field.
- Use with `{ unique: true }` to force uniqueness on only non-null values.

```javascript
db.places.ensureIndex({ rkey: 1 }, { sparse: true });
db.places.find({ 'coordinates.0': { $lt: 500 } }).count() // == 2
db.places.find({ 'coordinates.0': { $lt: 500 } }).sort({rkey: 1}).count() // == 1
```

_Beware_ of sorting with a sparse index as it can filter your returning dataset.

## Hash Indexing

Mostly useful as a shard key. Somewhat useful when querying for object equality:

```javascript
db.collection.find({ myobject: { data: "data.data.data.....", other: [ 1, 2, 3, 4, 5, 6, 7 ] } });
db.collection.ensureIndex({ myobject: 'hashed' });
```

mongo will now hash the object instead of comparing based on the object. _Note_ does not support range or sort queries.

## Other Indexing Options

- TTL a collection: Must be used on a date field.

    db.collection.ensureIndex({ timestamp: 1 }, { expireAfterSeconds: 500 });

- Drop duplicate documents:

    db.collection.ensureIndex({ type: 1, user: 1 }, { unique: true, dropDups: true });

- Don't kill your database:

    db.collection.ensureIndex({ sessId: 1 }, { background: true });

## Keeping Indexes in memory

- Indexes are second to the [working set](http://docs.mongodb.org/manual/reference/glossary/#term-working-set) to be kept in memory
- "Working set" is basically the amount of data AND indexes that will be active/in use by your system.
- It is much much much much better to keep your indexes in memory.
- It is not always necessary to keep _all_ of your index in memory, only the most used.

## Optimizing Queries

- [Profiling](http://docs.mongodb.org/manual/reference/command/profile/): `db.runCommand({ profile: 1 });`
- Log files - mongoDB will log all long queries (set by `slowms` parameter in profiler)
    - Remember slow queries may not be badly indexed, but instead held up by locks
- [dex](https://github.com/mongolab/dex) - The mongoDB indexing tool: `dex -f my/mongodb.log -n "mydb.mycoll" mongodb://myHost:12345/mydb`

## Log Optimizing

```
Fri Aug 17 16:21:43 [conn2472] update qa.products query: { UK.META.PRODUCT_GROUP_LABEL: { $exists: true } } update: { $unset: { UK.META.PRODUCT_GROUP_LABEL: 1.0 } } 133ms
Fri Aug 17 16:21:43 [initandlisten] connection accepted from 192.168.8.101:51638 #2481
Fri Aug 17 16:21:43 [conn2481] end connection 192.168.8.101:51638
Fri Aug 17 16:21:43 [conn2472] update qa.products query: { UK.META.PRODUCT_GROUP: { $exists: true } } update: { $unset: { UK.META.PRODUCT_GROUP: 1.0 } } 116ms
```

The mongoDB log records long operations, such as > 100ms. There's no index on `UK.META.PRODUCT_GROUP_LABEL`, but we can add it.

```javascript
db.products.ensureIndex({ 'UK.META.PRODUCT_GROUP_LABEL': 1 });
```

# Questions

I'm Jon Eisen.

[http://joneisen.me](http://joneisen.me)

[twitter://@jm_eisen](http://twitter.com/jm_eisen)

[git://github.com/yanatan16](http://github.com/yanatan16)

Presentation: [https://github.com/yanatan16/mongodb-indexing-presentation](https://github.com/yanatan16/mongodb-indexing-presentation)

Hosted via [revealme](https://github.com/yanatan16/revealme) using [reveal.js](http://lab.hakim.se/reveal-js/).
