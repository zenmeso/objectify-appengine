# Release Procedure #

When it is time for a release please do the following (in order).

Requirements:
  * java 1.6 sdk
  * ant
  * svn checkout of trunk
  * svn checkout of maven (_recommended_)

## Prepare Documents ##
  * Update wiki ReleaseNotes, mainPage
  * Build and commit javadocs
    * **Ensure you have [correct svn mime type](http://stuffthathappens.com/blog/2007/11/09/howto-publish-javadoc-on-google-code/) setup, the default is wrong.**  Do not commit javadocs without changing your subversion properties.

## Build it ##
  * Tag it (/tags/2.0)
  * `ant javadoc dist` (specify version when asked)

## Upload it ##
  * Upload to googlecode (title: Objectify-Appengine v2.0 , tags: OsSys-All,Featured)
  * Deprecate old versions
  * Commit maven checkout

## Announce it ##
  * Push an email out to user list, appengine-java list
  * Announce on TSS/Other