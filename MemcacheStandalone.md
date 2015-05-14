Objectify's write-through memcache can be used directly with the low level API, independent of the rest of Objectify.  You can use this to add caching behavior to apps built on the low level API or other datastore APIs that rely on the low level API.

IntroductionToObjectify documents how to use memcache with Objectify; this document assumes you are not using Objectify.  It assumes you are familiar with the [low level API](http://code.google.com/appengine/docs/java/datastore/).

# Introduction #

Objectify's memcache integration provides the `DatastoreService` and `AsyncDatastoreService` interfaces by wrapping the Google-provided objects and adding cache behavior.  By providing these interfaces, memcaching behavior can be added to any code built on top of the low-level API.

What you should know about the cache:

  * get(), put(), and delete() are cached.  query() does not use or affect the cache in any way.
  * Explicit transactions work normally - get()s bypass the cache to appear "frozen in time", put()s and delete()s affect the cache only on successful commit().
  * The cache is near-transactional.  Under normal operation, the cache will not go out of sync with the datastore, even under heavy contention.
    * The exception to this is `DeadlineExceededException` at the 30s (10m for tasks) hard deadline.  If Google kills your process before the cache update occurs, there's nothing we can do.
  * You can control cache expiration on a per-kind basis.
  * You can track hit/miss statistics.

**Warnings** about the cache:

  * Implicit (thread-local) transactions are not supported.  If you start a transaction, you must pass it explicitly into put(), get(), etc.  You can only use the methods without txn parameter outside of all transactions.
  * If you use asynchronous operations, you must install the [AsyncCacheFilter](http://objectify-appengine.googlecode.com/svn/trunk/javadoc/com/googlecode/objectify/cache/AsyncCacheFilter.html).  Please [star this issue](http://code.google.com/p/googleappengine/issues/detail?id=4271).

# Setup #

  1. Add objectify-X.X.X.jar to your project.  No other jars are necessary.
  1. Instead of `DatastoreServiceFactory.getDatastoreService()`, use **Caching**`DatastoreServiceFactory.getDatastoreService()`

That's it.

# Cache Expiration #

You can control cache expiration for the `DatastoreService` instance by passing it in to the factory method:

```
// Time is in seconds
DatastoreService ds = CachingDatastoreServiceFactory.getDatastoreService(600);
```

You can control cache expiration on a key-by-key basis:

```
EntityMemcache em = new EntityMemcache("yournamespace", new CacheControl() {
    @Override
    public Integer getExpirySeconds(Key key) {
        if ("SomeEntity".equals(key.getKind()))
            return 600;
        else if ("Permanent".equals(key.getKind()))
            return 0;    // cache without limit
        else
            return null;    // do not cache

        return expirySeconds;
    }
});

DatastoreService ds = CachingDatastoreServiceFactory.getDatastoreService(em);
```

# Tracking Hits and Misses #

You can construct an `EntityMemcache` which records cache statistics.

```
CacheControl cc = ...

EntityMemcache em = new EntityMemcache("yournamespace", cc, new MemcacheStats() {
    @Override public void recordHit(Key key) { /* do something with the hit */ }
    @Override public void recordMiss(Key key) { /* do something with the miss */ }
});

DatastoreService ds = CachingDatastoreServiceFactory.getDatastoreService(em);
```