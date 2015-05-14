

# Add `objectify.jar` and `guava.jar` to your project #

Add `objectify-N.N.N.jar` and `guava-N.N.N.jar` to your project's WEB-INF/lib directory. [Google Guava](https://code.google.com/p/guava-libraries/) provides a number of incredibly useful classes that you should be using in your project anyways. Note: You want the regular guava jar, not the -gwt jar.

If you use Maven, see the MavenRepository. Objectify will automatically bring in the transitive dependency on guava.

# Enable `ObjectifyFilter` for your requests #

Objectify requires a filter to clean up any thread-local transaction contexts and pending asynchronous operations that remain at the end of a request.  Add this to your WEB-INF/web.xml:

```
<filter>
	<filter-name>ObjectifyFilter</filter-name>
	<filter-class>com.googlecode.objectify.ObjectifyFilter</filter-class>
</filter>
<filter-mapping>
	<filter-name>ObjectifyFilter</filter-name>
	<url-pattern>/*</url-pattern>
</filter-mapping>
```

## Guice Alternative ##

If you use Guice, you don't use web.xml to define filters.  Add this to your servlet module:

```
filter("/*").through(ObjectifyFilter.class);
```

...and this to your business module:

```
bind(ObjectifyFilter.class).in(Singleton.class);
```

# Enable static imports in Eclipse #

This step is optional, but will help prevent you from typing `ObjectifyService.ofy()` over and over again.

Eclipse does not automatically add static imports when you Organize Imports.  By default, it won't even complete static imports when you type _`ofy`[cmd-space]_.  If you want to save yourself a lot of typing, add a "Favorite" static import for `ObjectifyService.ofy()`.

Visit **Window » Preferences » Java » Editor » Content Assist » Favorites** and add:

```
com.googlecode.objectify.ObjectifyService.ofy
```

Now, when you type _`ofy`[cmd-space]_, Eclipse will add a static import for you.

More details can be found in [this stackoverflow question](http://stackoverflow.com/questions/288861/eclipse-optimize-imports-to-include-static-imports).

# Building Objectify From Source #

If you would like to build Objectify from source, see ContributingToObjectify.