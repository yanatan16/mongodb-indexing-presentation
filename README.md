<!--

  A revealme-formatted presentation for MongoDB Indexing

  By Jon Eisen
  Given June 24 2013

title: Much ado about indexing
theme: sky
-->

# Much ado about Indexing

_by [Jon Eisen](http://joneisen.me)_

### [Presentation](http://revealme.herokuapp.com/yanatan16/mongodb-indexing-presentation)
### [Github](https://github.com/yanatan16/mongodb-indexing-presentation)

## Overview

Indexes are awesome because:

- take a linear search and make it logarithmic (i.e. search through 8 items in 3 steps instead of 8).
- eliminate in-memory sorts.
- reduce hard drive lookups and page faulting.

> Indexes make things faster.

# Preliminaries

## Commands

    > db.collection.ensureIndex(indexed_fields, [options]);
    > db.collection.getIndexes();
    > db.collection.dropIndex(indexed_fields);
    > db.collection.find(find_fields).explain();
    > db.collection.find(fields).hint(index_or_name);

## Points

- `ensureIndex({_id: 1}, {unique: true})` is always (mostly) true
- A query can only use one index (exception: $or queries)
- Order of keys matters for range and sort queries.
- `.explain()` not only gives information about the query, but also runs the query _optimizer_.
- Inserts and _some_ updates cause full index updates

> Always use _a priori_ knowledge of the data structure when creating your index.

## B-Trees

All indexes in Mongo are B-Trees. Understanding them will help you understand indexing.

> The B-tree is a generalization of a binary search tree in that a node can have more than two children.

See the [Wikipedia](https://en.wikipedia.org/wiki/B-tree) article.

# Indexing Basics

## Equality queries



    > db.subunits.find({ id: 'ZMB' }).explain();
    {
      "cursor" : "BasicCursor", // no index!
      "n" : 1,
      "nscanned" : 170, // Searched 170 objects to find 1
    }
    > db.subunits.ensureIndex({ id: 1 }, { unique: true });
    > db.subunits.find({ id: 'ZMB' }).explain();
    {
      "cursor" : "BtreeCursor id_1", // yay index worked!
      "n" : 1,
      "nscanned" : 1,
    }

## Multi-field queries



    > db.subunits.find({ type: 'Polygon', rkey: 40 });
    > db.subunits.ensureIndex({ type: 1, rkey: 1 },
      { name: 'my_custom_name' });

    // Works with a prefix subset
    > db.subunits.find({ type:'Polygon' }) // indexed

    > db.subunits.find({ rkey: 40 }) // not-indexed!
    > db.subunits.find({ rkey: 40 }).hint('my_custom_name'); // indexed!

- Prefix indexes are optimal secondary indexes.
- Subset (non-prefix) are non-optimal secondary indexes that must be hinted.
    - _Note_: The query optimizer will never choose this, but it _does_ work.

## Multi-key Index

> Multikey index: Any index with a field that is an array or is contained in an array.

Examples:

    { keywords: [ "fibonacci", "sequence", "has", "a", "closed", "form" ] }
    > db.coll.ensureIndex({ keywords: 1 });

This index will add a entry for each field in the array!

    { comments: [
      { user: "joe", text: "that's great!" },
      { user: "sam", text: "boo!" } ] }
    > db.coll.ensureIndex({ 'comments.user': 1 });

This index will a entry for each `comment.user` for each `comment` in `comments`!


# Complex Indexing

## Equality, Range, Sort

    > db.subunits.find({ type: 'Polygon', rkey: { $gt: 40 }})
      .sort({ 'properties.name': -1 }).explain();
    {
      "cursor" : "BtreeCursor type_1_rkey_1", // used the previous index
      "scanAndOrder" : true, // OH NO! in-memory sort!
    }
    > db.subunits.ensureIndex({ type: 1, rkey: 1, 'properties.name': -1 }); // conjectured index...
    > db.subunits.find({ type: 'Polygon', rkey: { $gt: 40 }})
      .sort({ 'properties.name': -1 }).explain();
    {
      "cursor" : "BtreeCursor type_1_rkey_1", // wait what? why didn't it use our index?
      "scanAndOrder" : true, // still in-memory sorting
    }

That index is not _optimal_.

## For reals this time

Index *Equality*, then *Sorts* (in order with correct _directions_), then *Range queries*.



    > db.subunits.ensureIndex({
        type: 1,
        'properties.name': -1,
        rkey: 1
      }); // the Correct index
    > db.subunits.find({ type: 'Polygon', rkey: { $gt: 40 }})
        .sort({ 'properties.name': -1 }).explain();
    {
      "cursor" : "BtreeCursor type_1_rkey_1", // what happened?
      "scanAndOrder" : true, // still in-memory sorting
    }

> There is a tradeoff between walking the BTree and sorting in memory.
> The query optimizer does not always get it right. Compare them yourself with `.hint()`

# Index Options

- Sparse
- Hash
- Geo
- TTL
- `dropDups`
- `background`

## Sparse Indexes

- Like regular secondary indexes, but they do not include references to documents without that field.
- Use with `{ unique: true }` to force uniqueness on only non-null values.


    > db.places.ensureIndex({ rkey: 1 }, { sparse: true });
    > db.places.find({ 'coordinates.0': { $lt: 500 } })
      .count()
    2
    > db.places.find({ 'coordinates.0': { $lt: 500 } })
      .sort({rkey: 1}).count() // == 1
    1

_Beware_ of sorting with a sparse index as it can filter your returning dataset.

## Hash Indexing

Mostly useful as a shard key. Somewhat useful when querying for equality on a big field:

    > db.collection.find({ bigfield: "lots of binary data......." });
    > db.collection.ensureIndex({ bigfield: 'hashed' });

mongo will now hash the binary data, object, or array instead of comparing based on the object.

> Does not support range or sort queries.

## Geospatial Indexes

Can either do "flat" (Euclidean plane) or "sphere" (Earthlike)

    db.places.ensureIndex( { "locs": "2dsphere" } );
    db.points.on.a.plane.ensureIndex { "locs": "2d" } );

Allows fast use of geospatial queries:

- `$geoWithin`
- `$geoIntersects`
- `$near`

_Note: Geospatial indexes cannot be used as shard key.

[Mongo docs](http://docs.mongodb.org/manual/core/geospatial-indexes/)

## Other Indexing Options

- TTL a collection: Must be used on a date field.


    > db.collection.ensureIndex({ timestamp: 1 }, { expireAfterSeconds: 500 });

- Drop duplicate documents:


    > db.collection.ensureIndex({ type: 1, user: 1 }, { unique: true, dropDups: true });

- Do not kill your database:


    > db.collection.ensureIndex({ sessId: 1 }, { background: true });


  - Beware of doing this on replica sets. See the [docs](http://docs.mongodb.org/manual/tutorial/build-indexes-on-replica-sets/#index-building-replica-sets).

# Optimization

Optimize:

- `update`'s first because of the db-level lock.
- `find`'s second to minimize working set churn.

> When comparing indexing performance, emulate your app; don't just run the same query 100 times.


    > db.collection.find({ mykey: { $gt: Math.random() } }).explain();

## Keeping Indexes in memory

- Indexes are second to the [working set](http://docs.mongodb.org/manual/reference/glossary/#term-working-set) to be kept in memory
- _Working set_ is basically the amount of data AND indexes that will be active/in use by your system.
- It is much much much much better to keep your indexes in memory.
- It is not always necessary to keep _all_ of your index in memory, only the most used.

## Optimizing Queries

- [Profiling](http://docs.mongodb.org/manual/reference/command/profile/): `db.runCommand({ profile: 1 });`
- Log files - mongoDB will log all long queries (set by `slowms` parameter in profiler)
- Remember slow queries may not be badly indexed, but instead held up by locks
- [dex](https://github.com/mongolab/dex) - The mongoDB indexing tool: `dex -f my/mongodb.log -n "mydb.mycoll" mongodb://myHost:12345/mydb`


## Log Optimizing

> Remember that a long query might just be delayed by locking, be sure to look around for the culprit.


    Fri Aug 17 16:21:43 [conn2472] update qa.products query: { UK.META.PRODUCT_GROUP_LABEL: { $exists: true } } update: { $unset: { UK.META.PRODUCT_GROUP_LABEL: 1.0 } } 133ms
    Fri Aug 17 16:21:43 [conn2472] update qa.products query: { UK.META.PRODUCT_GROUP: { $exists: true } } update: { $unset: { UK.META.PRODUCT_GROUP: 1.0 } } 116ms


1. First we check to make sure theres not index already on that field. (`getIndexes()`)

2. Then we make sure to explain the query (just to make sure its unindexed and slow): `find().explain()`
3. Then we index the field: `ensureIndex({ 'UK.META.PRODUCT_GROUP_LABEL': 1 });`

# Questions

I'm Jon Eisen.

[http://joneisen.me](http://joneisen.me)

[twitter://@jm_eisen](http://twitter.com/jm_eisen)

[git://github.com/yanatan16](http://github.com/yanatan16)

Presentation: [https://github.com/yanatan16/mongodb-indexing-presentation](https://github.com/yanatan16/mongodb-indexing-presentation)

Hosted via [revealme](https://github.com/yanatan16/revealme) using [reveal.js](http://lab.hakim.se/reveal-js/).
