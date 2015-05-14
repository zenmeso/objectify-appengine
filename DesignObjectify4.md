# What's wrong with the old API? #
  * Putting Key<?> in the data model is a PITA sometimes
    * Hard to create tidy data models for jsonification or serialization to clients; tends to require ugly @Transient fields
    * Manually populating @Transient fields is errorprone and can be inefficient
  * Asynchrony shoehorned in
    * Most operations already have interfaces that can hide asynchrony, for example:
      * `Map<Key<Thing>, Thing> map = ofy.get(k1, k2)` can be async
      * `Result<Map<Key<Thing>, Thing>> map = ofy.async().get(k1, k2)` is unnecessary
  * As get() methods for refs and unfetched entities added, API on Objectify is exploding
  * Reliance on JPA annotations is problem for some
  * Defaulting fields to indexed was a bad idea

# Design goals #
  * Make Key<?> optional in the data model
  * Automatic fetching of object graphs
    * Fine tuning with easy-to-use fetch groups (not JDO fetch groups!)
  * Use interfaces to hide asynchrony from client
    * **Everything** should be asynchronous!
  * Use Result<?> when asynchrony cannot be hidden (ie, returning concrete classes)
  * Consistent interface for batch get vs filtering
  * All core config/command objects are immutable
  * Fix poor initial design decisions

# Cleanup #
  * Stop using JPA annotations
  * Convert all annotations to imperative form (eg @Index instead of @Indexed)
  * All fields default to not indexed
  * Session cache enabled by default
  * Eliminate ObjectifyOpts, provide a fluent way to define options
  * More control of Objectify instance; switch read consistency, disable/enable caches, etc
    * Objectify instance remains immutable, however
      * Well not exactly, because the session cache is shared among an instance chain

# Annotation changes #
All annotations live in com.googlecode.objectify.annotation.  JPA annotations are not used at all.

| Old | New | Notes |
|:----|:----|:------|
| @Entity | @Entity | unchanged |
| @Indexed | @Index |  |
| @Unindexed | @Unindex |  |
| @Transient | @Ignore |  |
| @NotSaved | @IgnoreSave |  |
|  | @IgnoreLoad | Needed because @AlsoLoad doesn't "steal" anymore |
| @PostLoad | @OnLoad |  |
| @PrePersist | @OnSave |  |
| @Cached | @Cache |  |
| @Embedded | @Embed |  |
| @Serialized | @Serialize |  |
|  | @Load | previously proposed as @Fetch |
|  | @OnLoadDelegate | see below |
|  | @OnSaveDelegate | see below |

New annotations which can be put at Class level:
  * @OnSaveDelegate(SomeClass.class)
  * @OnLoadDelegate(SomeClass.class)

In this case, SomeClass implements standard interfaces, something like:
```
interface LoadLifecycle<T> {
   void onLoad(T entity, Objectify ofy);
}
interface SaveLifecycle<T> {
   void onSave(T entity, Objectify ofy);
}
```

Since all such objects are created with the overridable method ObjectFactory.create(), it's easy to pull these from Guice.

Note: lifecycle delegates have not been implemented yet.

# Example of class with fetching #
```
class Beastie {
    public static class BigGroup {}
    public static class SmallGroup{}

   @Parent
   @Load
   Ref<ParentThing> parent;

   @Id Long id;

   // Note these direct entity references are not implemented yet
   @Reference
   SomeThing some;    // implies @Load
   @Reference
   List<OtherThing> others;    // implies @Load

   @Load({BigGroup.class, SmallGroup.class})
   Ref<OtherThing> refToOtherThing;

   Ref<OtherThing> anotherRef;  // no @Load means never fetched automatically
}
```

# The Ref class #
```
class Ref<T> {
    Key<T> key();	// also aliased to getKey() for javabeans-compliance
    Key<T> safeKey() throws NotFoundException;
    T get();	// also aliased to getValue() for javabeans-compliance
    T safeGet() throws NotFoundException;
}
```

# API examples #
```
// Note LoadResult<T> extends Result<T>, adds safe() and other methods

// Simple key fetch, always async
LoadResult<Thing> th = ofy.load().key(thingKey);
Thing th =      ofy.load().key(thingKey).now();
Thing th =      ofy.load().key(thingKey).safe();	// throws NotFoundException

LoadResult<Thing> th = ofy.load().entity(entity);	// Fetching "unfetched" entities
LoadResult<Thing> th = ofy.load().value(rawKey);	// "anything" including raw datastore keys, Key<?>s, Ref<?>s, entities, etc

// With a fetch group - fetch groups are arbitrary marker classes
Thing th = ofy.load().group(SomeGroup.class).key(thingKey).now();

// Multi get is async, hides asynchrony behind Map interface
Map<Key<Thing>, Thing> ths = ofy.load().keys(thingKey1, thingKey2, thingKey3);	

// Fetching by id
Result<Thing> th = ofy.load().type(Thing.class).id(123L);
Thing th =      ofy.load().type(Thing.class).id(123L).now();

// With a parent
Result<Thing> th = ofy.load().type(Thing.class).parent(par).id(123L);

// Batch get, asynchrony is hidden behind Map
Map<Key<Thing>, Thing> ths = ofy.load().type(Thing.class).ids(123L, 456L);

// Queries (asynchrony hidden of course)
QueryResultIterable<Key<Thing>> ths =  ofy.load().type(Thing.class).filter("foo", foo).keys().iterable();
QueryResultIterable<Thing> ths =       ofy.load().type(Thing.class).filter("foo", foo).iterable();

// Getting query results as a List
List<Key<Thing>> ths =  ofy.load().type(Thing.class).filter("foo", foo).keys().list();
List<Thing> ths =       ofy.load().type(Thing.class).filter("foo", foo).list();

// Get first item whole
LoadResult<Thing> th = ofy.load().type(Thing.class).filter("foo", foo).first();
Thing th =       ofy.load().type(Thing.class).filter("foo", foo).first().now();
Thing th =       ofy.load().type(Thing.class).filter("foo", foo).first().safe();	// throws NotFoundException

// Get first item but only the key; the entity will have id/parent but otherwise be unintialized
LoadResult<Key<Thing>> th = ofy.load().type(Thing.class).filter("foo", foo).keys().first();
Key<Thing>       th = ofy.load().type(Thing.class).filter("foo", foo).keys().first().now();
Key<Thing>       th = ofy.load().type(Thing.class).filter("foo", foo).keys().first().safe();	// throws NotFoundException

// Query with fetch group
QueryResultIterable<Thing> ths = ofy.load().group(SomeGroup.class).type(Thing.class).filter("foo", foo).iterable();

// Save
ofy.save().entities(e1, e2, e3);		// asynchronous
ofy.save().entities(e1, e2, e3).now();	// synchronous

// Delete
ofy.delete().keys(e1, e2, e3);		// asynchronous
ofy.delete().keys(e1, e2, e3).now();	// synchronous
```