

# Frequently Asked Questions #

## Why aren't my queries working properly? ##

All of Objectify's intermediate command objects are immutable. This will not work:
```
Query<Foo> q = ofy().load().type(Foo.class);
q.filter("bar", bar);
List<Foo> foos = q.list();
```

The filter command did nothing because you did not reassign `q`. You need this:

```
q = q.filter("bar", bar);
```

Alternatively, chain the whole sequence in a single statement.  [Read more here.](https://code.google.com/p/objectify-appengine/wiki/Queries#Executing_Queries)

## `NoSuchMethodError`?!? ##

This is almost always a result of having the wrong (or a duplicate) jar in your classpath. Ensure that you have one and only one `objectify.jar` on your classpath, that it is the correct version, and that you are running against the correct minimum version of the Google App Engine SDK.  Check the Objectify ReleaseNotes to see what minimum version is required.  Google is rapidly developing Appengine; as they add new features, we add support to Objectify.  This tends to require up-to-date versions of the SDK.

Also be sure that you have the `guava.jar` dependency on your classpath. If you are using Maven, the dependency will be included automatically.

## Strange things are showing up in my session cache! (or missing from it!) ##
This is almost always caused by one of two problems:

  * You did not enable the `ObjectifyFilter` in your web.xml. You must install this filter.
  * You are sharing an `Objectify` instance across threads. `Objectify` instances represent a single session of work and are not thread-safe.

Make sure the `ObjectifyFilter` is installed, and always use the static `ofy()` method whenever you perform Objectify operations. If you never hold on to an `Objectify` reference, you will not get into trouble.

## ClassCastException: com.google.appengine.api.datastore.Entity cannot be cast to XXX ##
If you load an entity kind that has not been registered, Objectify leaves it untranslated as the raw datastore Entity. This is a convenience feature.

This means that if you have any kind of race condition that allows load() operations to proceed before registration, you may receive ClassCastExceptions when you try to cast the value returned by ofy().load() to your expected entity type.

See BestPractices for more about how to register entities.

## Will GAE complete my asynchronous operations if I don't call now()? ##
Yes. GAE completes all asynchronous operations at the end of the request. This means your request will block until they finish; async operations are not a way of getting around the 60s request deadline.

Also, asynchronous operations are completed at the time of transaction commit.

## Can I use Dependency Injection (Spring, Guice, Weld) with Objectify? ##
Yes!  Objectify is designed to integrate with DI systems and can even use injectors to create entities and internal classes.  See the Motomapia [Example](Examples.md).

However, we do **not** recommend using DI to inject `Objectify` instances.  As you enter and exit transaction contexts, your code will use different `Objectify` instances; a single injected instance creates a significant risk that you will mix up transaction contexts.  Always use the static `ofy()` method.

## How do I shut off the datanucleus byte code enhancer? ##
  * In Eclipse, go to the Project Properties -> Google -> App Engine and **uncheck** "Use Datanucleus JDO/JPA to access the datastore"

## Can I change data after it is loaded or before it is stored? ##
Absolutely, see the LifecycleCallbacks

## Should I Use a String (Name) Id? ##
If it is a natural key, sure; Objectify allows this no problem. Here is a little more of a discussion [about it.](http://groups.google.com/group/google-appengine-java/browse_thread/thread/998289f853f5a688/c50446d98eb90017?#c50446d98eb90017)

## How do I do a like query (LIKE "foo%") ##
You can do something like a startWith (or endWith if you reverse the order when stored and searched). You do a range query with the starting value you want, and a value just above the one you want.

```
String start = "foo";
... = ofy().load().type(MyEntity.class).filter("field >=", start).filter("field <", start + "\uFFFD"); 
```

See this discussion for [more background and details](http://groups.google.com/group/objectify-appengine/browse_thread/thread/51cebf7fdf13b9aa/895ddc627ed639cc)

## How do I migrate from JDO to Objectify? ##
You should have no trouble using Objectify side-by-side with JDO.

For the most part, migrating JDO classes to Objectify classes should be pretty straightforward.  The only trick will be mapping the relationships.  Look in the datastore viewer to see what entity structure JDO created - relationships are just a Key on one end or both.  If you make this field a Key<?>, Ref<?>, or concrete entity reference (or collection thereof if appropriate) in your Objectify entities, it will map just fine.

## Does Objectify support JDO Fetch Groups? ##
There is no such thing as a fetch group in the GAE datastore; entities are fetched in their entirety or not at all.  Objectify reflects the underlying nature of the datastore so it does not require this JDO-ism.

## Does Objectify support JDO detach copy? ##
There is no need for this JDO-ism either.  Objectify entities are exactly as you define them.  They are not bytecode-enhanced, proxied, or manipulated in any way.  Want to serialize your entity?  Just make sure it is Serializable.

## Can I mix Low Level and Objectify writes in a single Objectify transaction? ##
Example...

```
ofy().save().entity(makeSomeObject()).now();
ofy().getDatastore().put(makeSomeLowLevelEntity());
```

Yes you can! With a few caveats...

  * The direct-datastore put() will bypass the session cache.  If the session cache already held a value for that key, subsequent a load() from the Objectify instance will produce the (now incorrect) cached value.
  * The global cache will remain consistent as long as you obtain the DatastoreService from ofy().getDatastore(); if you get it directly from DatastoreServiceFactory it will bypass Objectify's global (memcache) cache.

You're free to interleave Objectify and Low-Level API calls.