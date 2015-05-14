# Caching #

Objectify provides two different types of caches:

  * A _session cache_ which holds entity instances inside a specific `Objectify` instance.
  * A _global cache_ which holds entity data in the appengine memcache service.

The session cache is enabled automatically.  The global memcache requires an explicit annotation on your entity POJOs.

## Session Cache ##

The session cache, as it sounds, is associated with a specific request session.  When you obtain and begin using an `Objectify` instance, the session keeps track of all entities that have been loaded in this session.

  * The session cache holds your **specific entity object instances**.  If you `load()` the same entity, you will receive the exact same Java entity object instance.
  * The session cache is local to the `Objectify` instance.  If you start a new session (via `ObjectifyFactory.begin()`), it will have a separate cache.  If you use the thread-local `ObjectifyService.ofy()` method, the session cache will "just work" appropriately.
  * A get-by-key operation (single or batch) for a cached entity will return the entity instance **without** a call to the datastore or even to the memcache (if the global cache is enabled).  The operation is a simple hashmap lookup.
  * A query operation will return cached entity instances, however the (potentially expensive) call to the datastore will still be made.
  * The session cache is **not** thread-safe.  You should never share an `Objectify` instance between threads.
  * The session cache appears to be very similar to a JPA, JDO, or Hibernate session cache with one exception - there is no dirty change detection.  As per standard Objectify behavior, if you wish to change an entity in the datastore, you must explicitly `save()` your entity.
  * If you iterate through very large quantities of entities, you should periodically call `Objectify.clear()` to empty the cache.  This will prevent memory issues as the session grows without bounds.

### Transactions and the Session Cache ###

When you enter a transaction context, you receive a fresh, empty session cache.  There are two reasons for this:

  1. We must not allow arbitrary state to contaminate isolated transactions
  1. In order to enlist an entity in the transaction, a fetch must be made all the way through to the datastore.

When the transaction commits, the contents of the transaction's session cache are written into the "parent" session cache (the context before the transaction).  This may replace instances that have been loaded before.

## Global Cache ##

Objectify can cache your entity data globally in the appengine memcache service for improved read performance.  This cache is shared by all running instances of your application and can both improve the speed and reduce the cost of your application.  Memcache requests are free and typically complete in a couple milliseconds.  Datastore requests are metered and typically complete in tens of milliseconds.

The global cache is enabled by default, however you must still annotate your entity classes with `@Cache` to make them cacheable:

```
@Entity
@Cache
public class MyEntity {
    @Id Long id;
    ...
}
```

That's it!  Objectify will utilize the memcache service to reduce read load on the datastore.

What you should know about the global cache:

  * The fields of your entity are cached, not your POJO class itself.  Your entity objects will not be serialized (although any @Serialized fields will be).
  * Only get-by-key, save(), and delete() interact with the cache.  Query operations are not cached.
  * Writes will "write through" the cache to the datastore.  Performance is only improved on read-heavy applications.
  * Negative results are cached as well as positive results.
  * Transactional reads bypass the cache.  Only successful commits modify the cache.
  * You can define an expiration time for each entity in the annotation: `@Cache(expirationSeconds=600)`.  By default entities will be cached until memory pressure (or an 'incident' in the datacenter) evicts them.
  * You can disable the global cache for an operation:  `ofy().cache(false).load()...`
  * The global cache works in concert with the session cache.
    * Remember:  The session cache caches entity Java object instances, the global cache caches entity data.

Objectify's global cache provides near-transactional consistency with the datastore, even under heavy contention.  There is still, however, one circumstance in which the cache could go out of synchronization with the datastore: If your requests are cut off by DeadlineExceededException.

You can also use this cache with the Low Level API directly; see MemcacheStandalone.

## Hybrid Queries ##

Even though the datastore can run queries, it's actually a key-value store.  When you execute a query, GAE under-the-covers performs the rough equivalent of a keys-only fetch followed by a get-by-key for the keys.  Aside from cost, these two operations are almost identical, and perform about the same:

```
// This will cost 6 read ops
ofy().load().type(Thing.class).limit(5);

// This will cost 6 read ops + 5 small ops
ofy().consistency(Consistency.EVENTUAL).load().keys(ofy().load().type(Thing.class).limit(5).keys());
```

Note that queries in GAE are always eventually consistent, so the direct equivalent of a query is to explicitly make the get-by-key step eventually consistent.

Why is this useful?  Because if we perform keys-only queries followed by get-by-key, we might pull the entity out of memcache instead of the datastore.  With a warm cache, this kind of "hybrid query" will cost 1 read op (all queries cost one read op) + 5 small ops (one for each key) and execute _much_ faster than the standard full query.

Objectify is smart enough to take care of this for you.  It will automatically convert queries to hybrid queries when:

  * There is an explicit type() defined, and the type has a `@Cache` annotation
  * Caching is not disabled for the query (ie `ofy().cache(false)`
  * The consistency level is set to STRONG (the default)

You can explicitly enable or disable this behavior like this:

```
// If Thing has @Cache, this disables the hybrid query:
ofy().load().type(Thing.class).hybrid(false);

// Typeless queries are not hybridized by default since we don't know if they are cacheable:
ofy().load().ancestor(parent).hybrid(true);
```