# Migrating Schemas #

It is a rare schema that remains unchanged through the life of an application.  The datastore's schemaless nature is both a blessing and a curse - you can easily change schemas object-by-object on the fly, but you can't easily do it in bulk with an ALTER TABLE.  Objectify provides some simple but powerful tools to help with common types of structure change.

The basic process of schema migration using Objectify looks like this:

  1. Change your entity classes to reflect your desired schema.
  1. Use Objectify's annotations to map data in the old schema onto the new schema.
  1. Deploy your code, which now works with objects in the old schema and the new schema.
  1. Let your natural load()/save() churn convert objects for as long as you care to wait.
  1. Run a batch job to load() & save() any remaining entities.
  1. Remove the now-obsolete mapping code

Here are some common cases.

## Adding Or Removing Fields ##

This is the easiest - just do it!

You can add any fields to your classes; if there is no data in the datastore associated with that field, it will be left at its default value when the class is initialized.  This is worlds better than the exceptions you often get from JDO.

You can remove a field from your classes.  The data in the datastore will be ignored when the entity is load()ed.  When you next save() the entity, the entity will be saved without this field.

## Renaming A Field ##

Let's say you have an entity that looks like this:

```
@Entity
public class Person {
    @Id Long id;
    String name;
}
```

You're doing some refactoring and you want to rename the field "name" to "fullName".  You can!

```
@Entity
public class Person {
    @Id Long id;
    @AlsoLoad("name") String fullName;
}
```

When a Person is load()ed, the `fullName` field will be loaded either the value of _fullName_ or _name_.  If both fields exist, an IllegalStateException will be thrown.  When save()d, only _fullName_ will be written.

Caveat:  Queries do not know about the rename; if you filter by "fullName", you will only get entities that have been converted.  You can still filter by "name" to get only the old ones.  When renaming fields which are indexed, you way wish to perform two update passes so that both indexes can coexist.  After you switch queries to use the new field/index, you can delete the old field.

## Transforming Data ##

Now that you've migrated all of your data to the new Person format, let's say you now want to store separate first and last names instead of a single fullName field.  Objectify can help:

```
@Entity
public class Person {
    @Id Long id;
    String firstName;
    String lastName;

    void importCruft(@AlsoLoad("fullName") String full) {
        String[] names = full.split(" ");
        this.firstName = names[0];
        this.lastName = names[1];
    }
}
```

You can specify `@AlsoLoad` on the parameter of any method that takes a single parameter.  The parameter must be type-appropriate for what is in the datastore; you can pass Object and use reflection if you aren't sure.  Process the data in whatever way you see fit.  When the entity is save()d again, it will only have _firstName_ and _lastName_.

Caution:  Objectify has no way of knowing that the importCruft() method has loaded the firstName and lastName fields.  If both fullName and firstName/lastName exist in the datastore, the results are undefined.

## Changing Enums ##

Changing enum values is just a special case of transforming data.  Enums are actually stored as Strings (and actually, all fields can be converted to String automatically), so you can use an @AlsoLoad method to process the data.

Let's say you wanted to delete the AQUA color and replace it with GREEN:

```
public enum Color { RED, GREEN }    // AQUA has been removed from code but it still exists in the datastore

@Entity
public class Car {
    @Id Long id;
    @IgnoreLoad Color color;

    void importColor(@AlsoLoad("color") String colorStr) {
        if ("AQUA".equals(colorStr))
            this.color = Color.GREEN;
        else
            this.color = Color.valueOf(colorStr);
    }
}
```

We must use `@IgnoreLoad` on the original `color` field otherwise Objectify would load the field _and_ call the `@AlsoLoad` method.  You can `@AlsoLoad` the same value into any number of fields or methods.

## Moving Fields ##

Changing the structure of your entities is by far the most challenging kind of schema migration; perhaps you want to combine two entities into one, or perhaps you want to move an `@Embed` field into a separate entity.  There are many possible scenarios that require many different approaches.  Your essential tools are:

  * `@AlsoLoad`, which lets you load from a variety of field names (or former field names), and lets you transform data in methods.
  * `@IgnoreLoad`, which lets you have fields which are save-only.
  * `@IgnoreSave`, which lets you load data into fields without saving them again.
  * `@OnLoad`, which lets you execute arbitrary code after all fields have been loaded.
  * `@OnSave`, which lets you execute arbitrary code before your entity gets written to the datastore.

Let's say you have some embedded address fields and you want to make them into a separate Address entity.  You start with:

```
@Entity
public class Person {
    @Id Long id;
    String name;
    String street;
    String city;
}
```

You can take two general approaches, either of which can be appropriate depending on how you use the data.  You can perform the transformation on save or on load.  Here is how you do it on load:

```
@Entity
public class Address {
    @Id Long id;
    String street;
    String city;
}

@Entity
public class Person {
    @Id Long id;
    String name;

    @IgnoreSave String street;
    @IgnoreSave String city;

    Key<Address> address;

    @OnLoad void onLoad() {
        if (this.street != null || this.city != null) {
            this.address = ofy().save().entity(new Address(this.street, this.city)).now();
            ofy().save().entity(this);
        }
    }
}
```

If changing the data on load is not right for your app, you can change it on save:

```
@Entity
public class Address {
    @Id Long id;
    String street;
    String city;
}

@Entity
public class Person {
    @Id Long id;
    String name;

    @IgnoreSave String street;
    @IgnoreSave String city;

    Key<Address> address;

    @OnSave void onSave() {
        if (this.street != null || this.city != null) {
            this.address = ofy().save().entity(new Address(this.street, this.city)).now();
        }
    }
}
```

If you have an especially difficult transformation, post to the objectify-appengine google group.  We're happy to help.