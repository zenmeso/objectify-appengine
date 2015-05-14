# Upgrading #

Objectify v4 is a major departure from v3 with very significant, non-backwards-compatible API changes.  This guide should help you convert a v3 project to the new API.  Before upgrading, you may find it helpful to read the DesignObjectify4 for a general overview of what changed and why.

**Note:** If you have any problems upgrading, post to the [Objectify Google Group](http://groups.google.com/group/objectify-appengine).  We will update this document.

See [Motomapia](https://github.com/stickfigure/motomapia) for a real-world example.

The steps:

## Replace the Objectify3 jar with the Objectify4 jar ##

Be absolutely certain that the Objectify3 jar is no longer on your classpath. Also make sure the Google guava jar is available.

## Completely remove JDO & JPA ##

Objectify v4 no longer uses _any_ JPA annotations.  However, several of the new annotations have identical names but different packages (eg, `javax.persistence.Entity` -> `com.googlecode.objectify.annotation.Entity`).  By removing the JPA annotations from your classpath, your IDE will highlight all the (now invalid) references to old annotations.

This step is critical; if you have a large project, it will be almost impossible to hunt down the annotation changes individually.  Let the IDE help.

In Eclipse, visit Project Properties -> Google -> App Engine and **uncheck** "Use Datanucleus JDO/JPA to access the datastore".

If you use Maven, or manually maintain your jars, ensure that the file `geronimo-jpa_3.0_spec-1.1.1.jar` is not included in your project.  Note that the actual version numbers on this file may change; there should be no JPA-related jars in your project!

## Add @Entity ##

v3 recommended, but did not require the `@Entity` annotation on entities.  You only had to register them.

v4 requires the `@Entity` annotation on all classes which will be registered as entities.

## Update annotations ##

Eclipse should now highlight all the problem annotations in your project.  See DesignObjectify4 for a mapping of old-to-new annotations.

### Add indexes ###

Objectify3 indexed all properties by default. Objectify4 inverts this; all properties are **NOT** indexed by default. You must specify `@Index` on all fields you want indexed. You can place this annotation on the class to index all properties by default; use `@Unindex` on fields to stop indexing them.

## Add `ObjectifyFilter` ##

v4 requires a servlet filter on requests that use Objectify.  See [Setup](Setup.md).

If you have `AsyncCacheFilter` configured, you should remove it.  `ObjectifyFilter` includes this behavior.

## Use `ofy()` ##

In v3, you would typically inject or pass around instances of `Objectify`.  In v4, this is discouraged.  You should always use the static `ofy()` method whenever you use Objectify; this will prevent accidentally using the wrong transaction context.

Read BasicOperations.

## Objectify API has changed ##

Really do read BasicOperations and DesignObjectify4.  Instead of get()/put()/delete()/query(), Objectify v4 has a fluent, command-oriented API.  There is no longer an `ObjectifyOpts`; you can now easily configure options on a per-command basis.

## Query objects are now immutable ##

This will require looking at your code carefully.  In v3, Query objects were mutable and allowed configuration like this:

```
Query<Thing> q = ...create query object
q.filter("foo", foo);
q.limit(10);
```

**THIS WILL NO LONGER WORK!**

Objectify v4 query objects are immutable, and therefore must be assigned like this:

```
Query<Thing> q = ...create query object
q = q.filter("foo", foo);
q = q.limit(10);

// More likely, you will do this:
Query<Thing> q = ofy().load().type(Thing.class).filter("foo", foo).limit(10);
```

## New transaction syntax ##

Objectify v3 transaction syntax is pretty much exactly the same pattern you use for the low-level API or JDO:

```
Objectify ofy = ObjectifyService.beginTransaction();
try
{
    ClubMembers cm = ofy.get(ClubMembers.class, "k123");
    cm.incrementByOne();
    ofy.put(cm);

    ofy.getTxn().commit();
}
finally
{
    if (ofy.getTxn().isActive())
        ofy.getTxn().rollback();
}
```

UGLY!  Objectify v4 has a much more concise syntax for [Transactions](Transactions.md):

```
ofy().transact(new VoidWork() {
    public void vrun() {
        ClubMembers cm = ofy().load().type(ClubMembers.class).id("k123").now()
        cm.incrementByOne();
        ofy().save().entity(cm);
    }
});
```

Note that Ofy4 transactions are retried on optimistic concurrency failure (`ConcurrentModificationException` and therefore **must be idempotent**.  If your transaction code is not idempotent (a bad idea), you can limit retry behavior by using `transactNew(1, work)`.

Also note that the `Objectify` returned by `ofy()` will be a different instance inside and outside the transaction.

Please read [Transactions](Transactions.md) carefully, especially the section WRT transaction inheritance.  This is one of the most powerful features of Objectify4, and provides almost EJB-like transaction semantics.

## Wrappers ##

Objectify used to provide Wrapper classes (ObjectifyWrapper, LoaderWrapper, etc) which provided a very complicated mechanism for adding your own behavior. These have been removed; instead simply extend ObjectifyImpl, LoaderImpl, etc directly and add/override the behavior you want. This is much simpler.