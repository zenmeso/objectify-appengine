The Google App Engine datastore is fundamentally a key/value store.  However, by defining indexes on arbitrary fields, you can query for entities in a way almost reminiscent of an RDBMS.  There are some limitations:

  * Queries only return values in an index.  If an entity field is not indexed, queries on that field will return  no results.
  * Queries without an ancestor() restriction are always weakly consistent.  Queries with an ancestor() restriction follow the `Objectify.consistency()` setting, which defaults to STRONG.
  * Queries without an ancestor() restriction cannot be used within a transaction.

This document is not intended to be a comprehensive explanation of queries and indexes; only what you need to know to use Objectify.  You should read and understand the [GAE Documentation For Queries and Indexes](https://developers.google.com/appengine/docs/java/datastore/queries).



# Defining Indexes #

Objectify does **not** index properties by default.  You must explicitly define single-property indexes with the `@Index` annotation:

```
import com.googlecode.objectify.annotation.Entity;
import com.googlecode.objectify.annotation.Id;
import com.googlecode.objectify.annotation.Index;

@Entity
public class Car {
    @Id Long id;
    @Index String vin;
}
```

The index will be created **when the entity is saved**.  Changing the `@Index` annotation in your Java class does **not** affect data already stored in the datastore; you must re-save individual entities to add or remove the index.

You can change the default index state for fields of a class by putting `@Index` on the class:

```
@Entity
@Index
public class Car {
    @Id Long id;
    String vin;
    String license;
    @Unindex int color;
}
```

# Partial Indexes #

Indexes are expensive to create and maintain.  Writing a new single-property index consumes two datastore write operations (one for the ascending index, one for the descending index).  _Changing_ an index value requires _four_ write operations:  two to delete the old value, and two to write the new one.  With several indexed properties per entity, this adds up fast.

Often you only need to query on a particular subset of values for a field.  If these represent a small percentage of your entities, why index all the rest?  Some examples:

  * You might have a boolean "admin" field and only ever need to query for a list of the (very few) admins.
  * You might have a "status" field and never need to query for inactive values.
  * Your queries might not include null values.

Objectify gives developers the ability to define arbitrary conditions for any field.  You can create your own `If` classes or use one of the provided ones:

```
import com.googlecode.objectify.annotation.Entity;
import com.googlecode.objectify.annotation.Id;
import com.googlecode.objectify.annotation.Index;
import com.googlecode.objectify.condition.IfTrue;
import com.googlecode.objectify.condition.IfNotNull;
import com.googlecode.objectify.condition.IfNotZero;

@Entity
public class Person {
    @Id Long id;
    String name;

    // The admin field is only indexed when it is true
    @Index(IfTrue.class) boolean admin;

    // You can provide multiple conditions, any of which will satisfy
    @Index({IfNotNull.class, IfNotZero.class}) Long serialNumber;
}
```

These `If` conditions work with both `@Index` and `@Unindex` on fields.  You cannot specify `If` conditions on class-level annotations.

Check the [javadocs](http://www.javadochub.com/doc/com.googlecode.objectify/objectify) for available classes.  Here are some basics to start:  `IfNull.class, IfFalse.class, IfTrue.class, IfZero.class, IfEmpty.class, IfDefault.class`

### `IfDefault.class` ###

`IfDefault` is special.  It tests true when the field value is whatever the default value is when you construct an object of your class.  Here is an example of using the inverse, `@IfNotDefault`:

```
@Entity
public class Account {
    @Id Long id;

    // Only indexed when status is something other than INACTIVE
    @Index(IfNotDefault.class) StatusType status = StatusType.INACTIVE;
}
```

Note that you can initialize field values inline (as above) or in your no-arg constructor; either will work.

### Custom Conditions ###

You can easily create your own custom conditions by extending `ValueIf` or `PojoIf`.  `ValueIf` is a simple test of a field value.  For example:

```
public static class IfGREEN extends ValueIf<Color> {
    @Override
    public boolean matches(Color value) {
        return color == Color.GREEN;
    }
}

@Entity
public class Car {
    @Id Long id;
    @Index(IfGREEN.class) Color color;
}
```

You can use `PojoIf` to examine other fields to determine whether or not to index!  This example is inspired by the example in the [Partial Index](http://en.wikipedia.org/wiki/Partial_index) Wikipedia page, and will use a static inner class for convenience:

```
// We are modeling:  create index partial_salary on employee(age) where salary > 2100;
@Entity
public class Employee {
    static class SalaryCheck extends PojoIf<Employee> {
        @Override
        public boolean matches(Employee pojo) {
            return pojo.salary > 2100;
        }
    }

    @Id Long id;
    @Index(SalaryCheck.class) int age;
    int salary;
}
```

Examine the [source code](http://code.google.com/p/objectify-appengine/source/browse/#git%2Fsrc%2Fcom%2Fgooglecode%2Fobjectify%2Fcondition) of the `If` classes to see how to construct your own.  Most are one or two lines of code.

`If` conditions can be used with `@IgnoreSave` as well.

# Executing Queries #

Queries are a type of load() operation:

```
import static com.googlecode.objectify.ObjectifyService.ofy;

// Operators are >, >=, <, <=, in, !=, <>, =, ==
List<Car> cars = ofy().load().type(Car.class).filter("year >", 1999).list();
List<Car> cars = ofy().load().type(Car.class).filter("year >=", 1999).list();
List<Car> cars = ofy().load().type(Car.class).filter("year !=", 1999).list();
List<Car> cars = ofy().load().type(Car.class).filter("year in", yearList).list();

// No operator means ==
Car car = ofy().load().type(Car.class).filter("vin", "123456789").first().now();

// The Query itself is Iterable
Query<Car> q = ofy().load().type(Car.class).filter("vin >", "123456789");
for (Car car: q) {
    System.out.println(car.toString());
}

// Queries within transactions require ancestor()
List<Car> cars = ofy().load().type(Car.class).ancestor(parent).list();

// You can filter keys as well, but if we don't specify a type, we get List<Object>
List<Object> range = ofy().load().filterKey(">=", startKey).filterKey("<", endKey).list();

// You can query for just keys, which will return Key objects much more efficiently than fetching whole objects
Iterable<Key<Car>> allKeys = ofy().load().type(Car.class).keys();

// Useful for deleting items
ofy().delete().keys(allKeys);
```

**`Query` objects are immutable**.  You can build them up by reassigning the variable:

```
Query<Car> q = ofy().load().type(Car.class);
q = q.filter("vin >", "123456789");
q = q.filter("color", RED);
```

Query result objects (`Iterable<?>`, `List<?>`, `Ref<?>`) are inherently asynchronous.  The `Query` itself does not start execution:

```
// Query implements Iterable, but this does not start an actual query
Iterable<Car> query = ofy().load().type(Car.class).filter("vin >", "123456789");

// This starts executing an asynchronous query
Iterable<Car> cars = ofy().load().type(Car.class).filter("vin >", "123456789").iterable();
```

## Polymoprhism ##

Queries are polymorphic.  However, you must index `@EntitySubclass` as described in [Entity Polymorphism](http://code.google.com/p/objectify-appengine/wiki/Entities#Native_Representation):

```
@Entity
public class Animal {
    @Id Long id;
    String name;
}
	
@Subclass(index=true)
public class Mammal extends Animal {
    boolean longHair;
}
	
@Subclass(index=true)
public class Cat extends Mammal {
    boolean hypoallergenic;
}

Animal annie = new Animal();
annie.name = "Annie";
ofy().save().entity(annie).now();

Mammal mam = new Mammal();
mam.name = "Mam";
m.longHair = true;
ofy().save().entity(mam).now();

Cat nyan = new Cat();
nyan.name = "Nyan";
nyan.longHair = true;
nyan.hypoallergenic = true;
ofy().save().entity(nyan).now();

// This query will produce three objects, the Animal, Mammal, and Cat
Query<Animal> all = ofy().load().type(Animal.class);

// This query will produce the Mammal and Cat
Query<Mammal> mammals = ofy().load().type(Mammal.class);
```

## Cursors ##

Cursors let you take a "checkpoint" in a query result set, store the checkpoint elsewhere, and then resume from where you left off later.  This is often used in combination with the Task Queue API to iterate through large datasets that cannot be processed in the 60s limit of a single request.

### Cursor Example ###

The `Iterable`s provided by Objectify (including the `Query` object) are actually `QueryResultIterable`.  This will produce a `QueryResultIterator`, which allows you to obtain a `Cursor`.

This is an example of a servlet that will iterate through **all** the Car entities, 1000 at a time:

```
@Override
protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    Query<Car> query = ofy().load().type(Car.class).limit(1000);

    String cursorStr = request.getParameter("cursor");
    if (cursorStr != null)
        query = query.startAt(Cursor.fromWebSafeString(cursorStr));

    boolean continu = false;
    QueryResultIterator<Car> iterator = query.iterator();
    while (iterator.hasNext()) {
        Car car = iterator.next();

        ... // process car

        continu = true;
    }

    if (continu) {
        Cursor cursor = iterator.getStartCursor();
        Queue queue = QueueFactory.getDefaultQueue();
        queue.add(url("/pathToThisServlet").param("cursor", cursor.toWebSafeString()));
    }
}
```

# Embedded Classes #

The datastore does not natively index properties of `EmbeddedEntity`, the native representation for embedded objects. Objectify works around this by creating synthetic, indexed dot-separated fields in the top-level `Entity`. These fields are only used for queries; they are not used to load your POJO objects.

## Indexing Embedded Classes ##

As with normal entities, all fields within embedded classes are unindexed by default.  You can control this:

  * Putting `@Index` or `@Unindex` on a class (entity or embedded) will make all of its fields default to indexed or unindexed, respectively.
  * Putting `@Index` or `@Unindex` on a field will make it indexed or unindexed, respectively.
  * `@Index` or `@Unindex` status for nested classes and fields are generally inherited from containing fields and classes, except that:
    * `@Index` or `@Unindex` on a field overrides the default of the class containing the field.
    * `@Index` or `@Unindex` on a field which holds an embedded class will override the default on the class inside the field (be it a single class or a collection).

```
@Index
class LevelTwo {
    @Index String gamma;
    String delta;
}

@Index
class LevelOne {
    String beta;
    @Unindex LevelTwo two;
}

@Entity
class EntityWithComplicatedIndexing {
    @Id Long id;
    LevelOne one;
    String alpha;
}
```

If you persist one of these EntityWithComplicatedIndexing objects, you will find:

| `alpha` | not indexed |
|:--------|:------------|
| `one.beta` | indexed |
| `one.two.gamma` | indexed |
| `one.two.delta` | not indexed |

Note that `one.two.delta` is **not** indexed; the annotation on `LevelOne.two` overrides `LevelTwo`'s class default.

## Querying By Embedded Fields ##

For any indexed field, you can query like this:

```
ofy().load().type(EntityWithEmbedded.class).filter("one.two.bar =", "findthis");
```

Filtering works for embedded collections just as it does for normal collections:

```
ofy().load().type(EntityWithEmbeddedCollection.class).filter("ones.two.bar =", "findthis");
```