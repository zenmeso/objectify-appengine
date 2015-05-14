# Lifecycle Callbacks #

Objectify supports two lifecycle callbacks:  `@OnLoad` and `@OnSave`.  If you mark methods on your POJO entity or embedded class (or any superclasses) with these annotations, they will be called:

  * `@OnLoad` methods are called after your data has been populated on your POJO class from the datastore.
  * `@OnSave` methods are called just before your data is written to the datastore from your POJO class.

You can have any number of these callback methods in your POJO class or its superclasses.  They will be called in order of declaration, with superclass methods called first.

```
class MyEntityBase {
    String foo;
    String lowercaseFoo;
    @OnSave void maintainCaseInsensitiveSearchField() { this.lowercaseFoo = foo.toLowerCase(); }
}

class MyEntity extends MyEntityBase {
    @Id Long id;

    @Ignore Date loaded;
    @OnLoad void trackLoadedDate() { this.loaded = new Date(); }

    List<String> stuff = new ArrayList<String>();
    int stuffSize;   // indexed so we can query by list size
    @OnSave void maintainStuffSize() { this.stuffSize = stuff.size(); }
}
```

**Caution**: You can't update @Id or @Parent fields in a @OnSave callback.