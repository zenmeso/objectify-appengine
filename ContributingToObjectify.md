Interested in helping out with this project?  GREAT!  Here's some advice to get you started.

Discussions on the [Objectify Google Group](http://groups.google.com/group/objectify-appengine) are often helpful.

# The Repository #

Google Code holds the "official" git repository for Objectify, but githubbers can use https://github.com/stickfigure/objectify

# Build Environment #

Objectify developers use Eclipse and Maven.  Install the M2E maven plugin into Eclipse and open the project; all dependencies (including the appropriate GAE SDK) will download automatically.

# Check Out And Build #

```
git clone git://github.com/stickfigure/objectify.git
cd objectify
mvn package
```

# Code Standards #

Objectify code uses 4-space HARD tabs.  It uses K&R style bracing conventions.  Look at existing source code for an example and follow the pattern (although beware, there is still some lingering Pascal-style bracing).

All code submissions should be well-commented and include TestNG unit tests.

# Running Unit Tests #

To run the unit tests , install the TestNG plugin for Eclipse.

Note that there are a few tests that fail "out of the box".  These expose broken behavior in the GAE SDK itself and are documented as such.  If in doubt, look at the test code - if it it doesn't say "this is expected to fail", it shouldn't fail.

# Submitting Code #

Submit pull requests to https://github.com/stickfigure/objectify

# Welcome Aboard! #

The Objectify team is very interested your ideas and submissions.  That said, we are also very picky about what features we include in the core - Objectify is intended to be a thin, convenient layer on top of the low-level datastore API rather than a "kitchen sink" framework.  We are happy to link to related projects from the [Extensions](Extensions.md) page.

If you are wondering where to start, look through the issue tracker.  Start a discussion early on the [Objectify Google Group](http://groups.google.com/group/objectify-appengine) - we will take the time to offer API advice and architectural guidance as well as to evaluate the proposal.

The absolute best way to start is to submit a pull request with some unit tests - even (especially!) if they fail.