# Upgrading #

Objectify v5 brings two major changes:

  * The native storage format for embedded objects has changed from dot-separated property names to the new `EmbeddedEntity` native structure.
  * The code that translates back and forth between POJOs and native data structures has been rewritten from scratch.

This allows Objectify to offer great new features like polymorphism and lifecycle methods in embedded objects. However, it may require to migrate the structure of existing data.

# Incredibly Important Migration Instructions #

**IMPORTANT! IMPORTANT! IMPORTANT! IMPORTANT! IMPORTANT! IMPORTANT!**

If you have used `@Embed` or `@EmbedMap` to store data with previous versions of Objectify, and you do not follow these instructions, **you will lose data**. Furthermore, once you have followed these instructions, **you cannot rollback to a previous version of Objectify** or your data will be lost. If you have not used `@Embed` or `@EmbedMap`, the migration is unnecessary.

  1. Upgrade to v4.1.x
  1. Set `ObjectifyFactory.setSaveWithNewEmbedFormat(true)`
  1. For every entity that has an `@Embed` structure, `load()` + `save()` the entity (ideally in a transaction).

This will convert embedded structures into the format that Objectify 5 understands.

# V5 Changes #

Converting from 4.1 to 5 involves minimal changes:

  * `@EntitySubclass` has been renamed to `@Subclass`.
  * Remove any reference to the `@Embed` annotation. It does not exist in Objectify 5; any class which is not natively stored will be treated as an embedded class automatically.
  * Similarly, `@EmbedMap` has been removed. Fields of type `Map<String, ?>` are assumed to be embedded maps.
  * The `@Owner` annotation has been renamed to `@Container`.

Many limitations of Objectify4 no longer apply to Objectify5:

  * Nesting of collections of embedded objects is unrestricted.
  * Embedded objects can be polymorphic.
  * Embedded objects can have lifecycle methods.

The API for custom translators has changed. If you use custom translators, check the javadocs.

As always, seek help at the [Objectify Google Group](http://groups.google.com/group/objectify-appengine) if you have any questions.