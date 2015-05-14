

# The `Objectify` Instance #

All datastore operations begin with an instance of `Objectify` ([javadoc](http://objectify-appengine.googlecode.com/git/javadoc/com/googlecode/objectify/Objectify.html)).  An `Objectify` instance holds the complete state of your persistence session:

  * A transaction context, if present
  * A cache of entity objects loaded in the current session
  * Parameters for deadline, consistency setting, and caching

**An `Objectify` instance should be used from a single thread only.**  It is a cheap throwaway object that represents a session of activity.  Typically you will re-use a single instance through a request, except when you begin transactions.

## Obtaining an `Objectify` Instance ##

The preferred way to obtain an `Objectify` instance is to call a static method:

```
Thing th = ObjectifyService.ofy().load().key(thingKey).now();
```

To keep the typing to a minimum, we recommend a [static import](http://code.google.com/p/objectify-appengine/wiki/Setup#Enable_static_imports_in_Eclipse):

```
import static com.googlecode.objectify.ObjectifyService.ofy;

Thing th = ofy().load().key(thingKey).now();
```

`ObjectifyService.ofy()` will always return the correct instance for your thread and transaction context and provides you with an "always available" persistence context without having to pass extra parameters to your business methods.  The instance returned by `ofy()` will change when you enter and exit transactions.  Be sure to install the `ObjectifyFilter` so that the last instance is cleaned up at the end of a request.

As a general practice, we recommend **_not_** holding on to `Objectify` instance variables:

```
// This is generally a bad practice:
Objectify ofy = ofy();
ofy.load()... 
ofy.save()...

// This is safer:
ofy().load()...
ofy().save()...
```

Consistently using `ofy()` eliminates potential confusion when working with multiple layers of transaction depth.  This will be discussed in more detail in the section on [Transactions](Transactions.md).

## Altering Parameters ##

`Objectify` instances maintain immutable parameter state:

  * The deadline at which datastore requests are timed out and aborted
  * The consistency level for fetches
  * Whether or not to use a global (memcache) cache

Any given `Objectify` instance has immutable state, but methods can be called to create new `Objectify` instances with different parameters:

```
th = ofy().deadline(2).load().key(thingKey).now();
th = ofy().cache(false).load().key(thingKey).now();
th = ofy().deadline(2).cache(false).load().key(thingKey).now();
th = ofy().load().key(thingKey).now();    // still has default deadline and cache policy

// You can hold on to these immutable objects
Objectify shortDeadline = ofy().deadline(2);
th = shortDeadline.consistency(Consistency.EVENTUAL).load().key(thingKey).now();
th = shortDeadline.load().key(thingKey).now();    // still has default of STRONG
```

## The Session Cache ##

Each `Objectify` instance holds a cache of entities which have been loaded in that session.  This prevents subsequent loads within that session from needing to go out to the datastore.  The cache holds entity instances and (with the exception of transactions) will consistently return the same actual object instance.

This cache is separate from the global memcache enabled/disabled with the `@Cache` annotation and the `Objectify.cache()` parameter.

The session cache is always enabled.  If you iterate through large numbers of entities, this session cache may cause memory issues.  You can clear the cache at intervals by calling:

```
ofy().clear();
```

# Loading #

`Objectify` provides a variety of ways to load entities from the datastore.  They all begin with a command chain started by `load()`.  Note that all operations are asynchronous until you manifest an actual entity POJO; all the `Map`, `List`, and `Result` interfaces hide the asynchrony.

This covers the type of requests that all translate into a get-by-key in the datastore.  For another way to load data, see [Queries](Queries.md).

```
// Simple key fetch, always asynchronous
Result<Thing> th = ofy().load().key(thingKey);

// Usually you don't need async, so call through the Ref<?>
Thing th =      ofy().load().key(thingKey).now();         // return null if not found
Thing th =      ofy().load().key(thingKey).safe();        // throws NotFoundException

Result<Thing> th = ofy().load().entity(entity);  // Equivalent to ofy().load().key(get_key_of(entity));
Result<Thing> th = ofy().load().value(rawKey);   // Accepts anything key-like; Key, Key<?>, Ref<?>, etc.

// Multi get is async, hides asynchrony behind Map interface
Map<Key<Thing>, Thing> ths = ofy().load().keys(thingKey1, thingKey2, thingKey3);	
Map<Key<Thing>, Thing> ths = ofy().load().keys(iterableOfKeys);	
// There are also entities() and values() methods

// Fetching by id - this is really just syntactic sugar for get-by-key
Result<Thing> th = ofy().load().type(Thing.class).id(123L);
Thing th =      ofy().load().type(Thing.class).id(123L).now();

// With a parent
Result<Thing> th = ofy().load().type(Thing.class).parent(par).id(123L);

// Batch get, asynchrony is hidden behind Map
Map<Long, Thing> ths = ofy().load().type(Thing.class).ids(123L, 456L, 789L);
```

## Load Groups ##

When you use [Ref<?>](http://code.google.com/p/objectify-appengine/wiki/Entities#Ref<?>_s) fields to establish relationships between entities, you can use the `@Load` annotation to automatically retrieve an object graph using the optimal number of batch fetches.  However, you do not always want to load an entire graph.

Objectify provides _load groups_, inspired by Jackson's [Json Views](http://wiki.fasterxml.com/JacksonJsonViews).  Here is the simplest usage:

```
@Entity
public class Thing {
    public static class Everything {}

    @Id Long id;
    @Load(Everything.class) Ref<Other> other;
}

// Doesn't load other
Thing th = ofy().load().key(thingKey).now();

// Does load other
Thing th = ofy().load().group(Everything.class).key(thingKey).now();
```

Here is a more complicated example:

```
@Entity
public class Thing {
    public static class Partial {}
    public static class Everything extends Partial {}
    public static class Stopper {}

    @Id Long id;
    @Load(Partial.class) Ref<Other> withPartial;
    @Load(Everything.class) Ref<Other> withEveryhthing;
    @Load(unless=Stopper.class) Ref<Other> unlessStopper;
}

// Loads 'unlessStopper' but not 'withPartial' or 'withEveryhthing'
Thing th = ofy().load().key(thingKey).now();

// Loads 'withPartial' and 'unlessStopper' but not 'withEveryhthing'
Thing th = ofy().load().group(Partial.class).key(thingKey).now();

// Loads all three because Everything extends Partial
Thing th = ofy().load().group(Everything.class).key(thingKey).now();

// Loads 'withPartial' and 'withEveryhthing' but not 'unlessStopper'
Thing th = ofy().load().group(Everything.class, Stopper.class).key(thingKey).now();
```

You can combine required and 'unless', and also specify multiple groups:

```
// Loaded if Green or Yellow is activated, unless Red is also activated.
@Load(value={Green.class, Yellow.class}, unless=Red.class) Ref<Other> other;
```

# Saving #

```
ofy().save().entity(thing1);         // asynchronous
ofy().save().entity(thing1).now();   // synchronous

ofy().save().entities(thing1, thing2, thing3);        // asynchronous
ofy().save().entities(thing1, thing2, thing3).now();  // synchronous

List<Thing> things = ...
ofy().save().entities(things);        // asynchronous
ofy().save().entities(things).now();  // synchronous
```

Be careful when saving entities with an autogenerated Long `@Id` field.  A synchronous save() will populate the generated id value on the entity instance.  An asynchronous save() will not; the operation will be pending until the async operation is completed.

```
@Entity
public class Thing {
    @Id Long id;    // capital-L Long will be autogenerated
}

Thing th = new Thing();

Result<Key<Thing>> result = ofy().save().entity(th);
assert th.id == null;    // It has not been generated yet!
result.now();
assert th.id != null;    // Now the id is available
```


# Deleting #

```
ofy().delete().key(thingKey1);        // asynchronous
ofy().delete().key(thingKey1).now();  // synchronous

ofy().delete().keys(thingKey1, thingKey2, thingKey3);        // asynchronous
ofy().delete().keys(thingKey1, thingKey2, thingKey3).now();  // synchronous

ofy().delete().entity(thing);        // asynchronous
ofy().delete().entity(thing).now();  // synchronous

ofy().delete().entities(thing1, thing2, thing3);        // asynchronous
ofy().delete().entities(thing1, thing2, thing3).now();  // synchronous

ofy().delete().type(Thing.class).id(123L);        // asynchronous
ofy().delete().type(Thing.class).id(123L).now();  // synchronous

ofy().delete().type(Thing.class).ids(123L, 456L, 789L);        // asynchronous
ofy().delete().type(Thing.class).ids(123L, 456L, 789L).now();  // synchronous

ofy().delete().type(Thing.class).parent(par).ids(123L, 456L, 789L);        // asynchronous
ofy().delete().type(Thing.class).parent(par).ids(123L, 456L, 789L).now();  // synchronous
```

# Deferring Operations #

All of the previously mentioned operations invoke datastore operations immediately - including (especially!) the asynchronous ones. This is inconvenient when you may end up conditionally making several changes to an entity. For example, when synchronizing multiple fields of an entity, you typically have to track a 'changed' state:

```
public void synchronize(Thing other) {
    if (fieldA != other.fieldA) {
        fieldA = other.fieldA;
        changed = true;
    }

    if (fieldB != other.fieldB) {
        fieldB = other.fieldB;
        changed = true;
    }

    if (changed)
        ofy().save(this);
}
```

Deferred operations are delayed until the end of a unit-of-work. That is, a transaction commit or the end of a request. Here is the above using deferred operations:

```
public void synchronize(Thing other) {
    if (fieldA != other.fieldA) {
        fieldA = other.fieldA;
        ofy().defer().save().entity(this);
    }

    if (fieldB != other.fieldB) {
        fieldB = other.fieldB;
        ofy().defer().save().entity(this);
    }
}
```

This will produce one datastore save at the end of the unit-of-work.

In the case of both `save()` and `delete()` operations deferred for the same entity, the last operation wins.