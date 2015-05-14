This is a combined introduction to Objectify and to the App Engine datastore.



So, you want to persist some data.  You've probably looked at the [datastore documentation](http://code.google.com/appengine/docs/java/datastore/overview.html) and thought "crap, that's complicated!"  Entities, query languages, fetch groups, detaching, transactions... hell those things have bloody _lifecycles_!  However, the complexity of JDO is hiding a lot of simplicity under the covers.

The first thing you should do is set aside all your preconceived notions about relational databases.  **The GAE datastore is not an RDBMS**.  In fact, it acts much more like a HashMap that gives you the additional ability to index and query for values.  When you think of the datastore, imagine a persistent HashMap.

# Entities #

This document will talk a lot about entities.  An _entity_ is an object's worth of data in the datastore.  Using Objectify, an entity will correspond to a single POJO class you define.  In the datastore, an entity is a HashMap-like object of type Entity.  Conceptually, they are the same.

Since the datastore is conceptually a HashMap of keys to entities, and an entity is conceptually a HashMap of name/value pairs, your mental model of the datastore should be a HashMap of HashMaps!

# Operations #

There are only four basic operations in the datastore, and any persistence API must boil operations down to:

  * **put()** an entity whole in the datastore.  You can store many at a time.
  * **delete()** an entity from the datastore, using its identity.  You can delete many at a time.
  * **get()** an entity whole from the datastore, using its identity.  You can get many at a time.
  * **query()** for entities matching indexed criteria you define.

Objectify uses slightly different terminology, but is implemented with the four basic operations above:

  * `save()` is a put()
  * `delete()` is a delete()
  * `load()` may be a get() or a query(), depending on the request

# Keys #

All entities have either a Long **id** or a String **name**, but those values are not unique by themselves.  In the datastore, entities are identified by the id (or name) and a **kind**, which corresponds to the type of object you are storing.  So, to get Car #959 from the datastore, you need to call something equivalent to `get_from_datastore("Car", 959)` (not real code yet).

By the way, I lied.  There is actually a third value which is necessary to uniquely identify an entity, called **parent**.  Parent defines a special kind of relationship, placing the child in the same [entity group](http://code.google.com/appengine/docs/python/datastore/keysandentitygroups.html) as the parent.  Entity groups will be discussed next in the section on Transactions, but for now what you need to know is that this parent (which is often simply null, creating an unparented, root entity) is also required to uniquely identify an entity.  So, to get Car #959 from the datastore, you actually need to call something equivalent to `get_from_datastore("Car", 959, null)` or `get_from_datastore("Car", 959, theOwner)`.  Yeech.

Instead of passing three parameters around all the time, the datastore wraps these values into a single object - a _Key_.  That's all a `Key` is, a holder for the three parts that uniquely identify an entity.

The native Datastore `Key` class is simple and untyped, like the native `Entity` class.  Objectify provides a generified `Key` that carries type information:

```
Key<Car> rootKey = Key.create(Car.class, 959);
Key<Car> keyWithParent = Key.create(parent, Car.class, 959);
```

In Objectify, you define your object as a Java class with a mandatory identifier (Long, long, or String) and an optional parent.  However, when you look up or reference your object, you do so by _Key_.  In fact, you can (and should) batch together a variety of requests into a single call, even if it will fetch many different kinds of objects:

```
Map<Key<Object>, Object> lotsOfThings = ofy().load().keys(carKey, airplaneKey, chairKey, personKey, yourMamaKey);
```

Actually, I lied again.  We don't force you to create keys by hand all the time.  There is a convenient shorthand for the very common case of loading a single type of object, but don't forget that this is really just creating an `Key` and get()ing it under the covers!

```
Car c = ofy().load().type(Car.class).id(959).now();
Map<Long, Car> cars = ofy().load().type(Car.class).ids(959, 911, 944, 924);
```

By the way, `Key` is used for relationship references as well.  Remember that value that defines a parent entity?  The type of this parent is `Key`:

```
// One of the overloads for static method Key.create():
public static <T> Key<T> create(Key<?> parent, Class<? extends T> kindClass, long id) { ... }
```

When you create relationships to other entities in your system, the type of the entity relationship should be `Key`.

# Transactions #

Transactions are the bread-and-butter of most business processing apps, and most RDBMSes have refined the transaction to an art form.  However, GAE's scalable, distributed datastore presents certain challenges which don't exist in a single Postgres instance.  Transactions on GAE are not quite like RDBMS transactions.  Here's what you need to know:

## Entity Groups ##

When you save() your entity, it gets stored somewhere in a gigantic farm of thousands of machines.  In order to perform an atomic transaction, the datastore requires that all the data that is a part of that atomic transaction live on the same server.  To give you some control over where your data is stored, the datastore has the concept of an **entity group**.

Remember the _parent_ that is part of a `Key`?  If an entity has a parent, it belongs to the same entity group as its parent.  If an entity does not have a parent, it is the "root" of an entity group, and may be physically located anywhere in the cluster.  All its child entities (and children of those children) are part of the same entity group and colocated in the same part of the datastore.

Note that an entity group is not defined by a _class_, but by an _instance_!  Each instance of an entity class which has a null (or absent) parent defines the root of a separate entity group.

Within a normal transaction, you can only access data from a single entity group.  If you try to access data from multiple entity groups, you will get an Exception.  This means you must pick your entity groups carefully, usually to correspond to the data associated with a single user.  This is dramatically constraining compared to the RDBMSes you are probably used to, where any piece of data is fair game for a single commit.

GAE does offer the ability to perform cross-group (XG) transactions, and Objectify enables this by default.  Just as each entity group is analogous to a separate database server, an XG transaction is analogous to a two-phase-commit distributed transaction - and this is exactly how it is implemented.  GAE allows you to enlist up to five entity groups in a single transaction, and the more you add the slower it will be.  However, this makes creating robust business logic _dramatically_ easier.

Why not store all your data with a common parent, putting it all in a single entity group?  You can, but it's a bad idea.  Google limits the number of requests per second that can be served by a particular entity group.

It is worth mentioning that the term _parent_ is somewhat misleading.  There is no "cascading delete" in the datastore; if you delete the parent entity it will NOT delete the child entities.  For that matter, you can create child entites with a parent `Key` (or any other key as a member field) that points to a nonexistant entity!  Parent is only important in that it defines entity groups; if you do not need transactions across several entities, you may wish to use a normal nonparent key relationship - even if the entities have a conceptual parent-child relationship.

## Optimistic Concurrency ##

When any entity group is changed, a timestamp is written to that group.  The timestamp is for the whole group; it is updated when any entity in that group is written.

When you start a transaction, each entity group you touch (by loading, saving, or deleting, up to the XG limit of 5 groups) is _enlisted_ in the transaction.  You can make as many changes to entities in those groups as you want.  When the transaction is committed, GAE checks the timestamps on all EGs enlisted in the transaction.  If any of those timestamps have changed (because some other transaction updated the EG), the whole transaction is rolled back.  The commit operation will throw a ConcurrentModificationException.

This is called "optimistic concurrency" or "optimistic locking" because it (optimistically) assumes that transactions will not interfere with each other and simply detects the case, throwing an error.  The opposite, "pessimistic locking", explicitly locks accessed resources and blocks other transactions who try to access the same resources.

Optimistic concurrency tends to have much higher throughput rates and eliminates the deadlocks that plague pessimistic strategies.  However, optimistic concurrency means that any transaction may fail with ConcurrentModificationException even though your logic is flawless; the data just happened to be modified while your transaction was in progress.

This means that transactions on GAE need two behaviors:

  * Transactions must be idempotent - you should be able to run the code any number of times and achieve the same result.
  * Transactions must be retried on ConcurrentModificationException - when there is a transaction collision, the right thing to do is just try again.  Even under contention, at least one transaction will succeed each round.

Objectify's [Transactions](Transactions.md) are specifically designed to make this process easy and transparent.  You define your transactions as idempotent lambda expressions (or as close to it as Java gets - anonymous inner classes); Objectify will take care of concurrency failures and retries for you.

## Transaction Limitations ##

When you execute a datastore operation, you will either be in a [transaction](http://code.google.com/appengine/docs/java/datastore/transactions.html) or you will not.  If you execute within a transaction:

  * Each EG you touch via get/put/delete/query enlists that EG in the transaction.
    * You can enlist up to 5 EGs, but single-EG transactions are fastest
  * Queries must include an ancestor which defines the EG in which to search.  You cannot query _across_ EGs at all, not even in XG transactions.
  * There are some quirks at the low-level API:  For example, get()s and query()s will see the datastore "frozen in time" and will not reflect updates even _within the transaction_.  Objectify hides this behavior from you; subsequent fetches will see the same data seen (or updated) previously.
    * Note that since queries are always run inside of GAE, indexes (ie, filtering operations) always appear to be frozen in time - Objectify can't hide this.

## Transactionless ##

If you operate on the datastore without an explicit transaction, each datastore operation is treated like a separate little transaction which is retried separately.  Note that batch operations are **not** transactional; each entity saved in a batch put() is effectively its own little transaction.  If you perform a batch save entities outside of a transaction, it is possible for some to succeed and some to fail.  An exception from the datastore does not indicate that all operations failed.

# Indexes #

When using a  traditional RDBMS, you become accustomed to issuing any ad-hoc SQL query you want and letting the query planner figure out how to obtain the result.  It may take twelve hours to linear scan five tables in the database and sort the 8 gigabyte result set in RAM, but eventually you get your result!  The appengine datastore does NOT work this way.

Appengine only allows you to run _efficient_ queries.  The exact meaning of this limitation is somewhat arbitrary and changes as Google rolls out more powerful versions of the query planner, but generally this means:

  * No table scans
  * No joins
  * No in-memory sorts

The datastore query planner really only likes one operation:  Find an index and walk it in-order.  This means that for any query you perform, the datastore must already contain properly ordered index on the field or fields you want to filter by!  And since appengine doesn't do joins, queries are limited to what you can stuff into a single index -- you can't, for example, filter by X > 5 and then sort by Y.

Actually, it's not quite true that appengine won't do joins.  It will do one kind of join - a "zig-zag" merge join which lets you perform equality filters on multiple separate properties.  But this is still an efficient query - it walks each of the property indexes in order without buffering chunks of data in RAM.

What you should be getting out of this is that **if you want queries, you need indexes tailored to the queries you want to run**.

To make this easier, the datastore has an innate ability to store each and every (single) property as "indexed" or "unindexed" (`Entity.setProperty()` vs `Entity.setUnindexedProperty()`.  This allows you to easily issue a queries based on single properties.  By default, Objectify does not index properties unless you explicitly flag the field (or class) with the `@Index` annotation.

To run queries by filtering or sorting against multiple properties (that is, if it can't be satisfied by a zigzag merge on single-property indexes), you must create a multi-value index in your `datastore-indexes.xml`.  There is a great deal written on this subject; we recommend [How Entities and Indexes are Stored](http://code.google.com/appengine/articles/storage_breakdown.html) and [Index Building](http://code.google.com/appengine/articles/index_building.html).

Note that there are some tricks to creating indexes:

  * Single property indexes are created/updated when you save an entity.  Let's say you have a `Car` with a `color` property.  If you save a `Car` with `color` unindexed, that entity instance will not appear in queries by color.  To index this entity instance, you must resave the entity.

  * Multi-property indexes are built on-the-fly by appengine.  You can add new indexes to your `datastore-indexes.xml` and appengine will slowly build a brand-new index - possibly taking hours or days depending on total system load (index-building is a low-priority task).

  * In order for an entity to be included in a multi-property index, **each of the relevant individual properties must have a single-property index**.  If your `Car` has a multi-property index on `color` and `brand`, an individual car will not appear in the multi-property index if it is saved with an unindexed `color`.