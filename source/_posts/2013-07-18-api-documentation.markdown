---
layout: post
title: "API Documentation"
date: 2013-07-18 16:15
comments: true
categories: [OPW, Twisted]
---
My focus for the past few weeks has been on reviewing the API
documentation for Twisted Mail and filling in the missing pieces.
Why bother with documentation when there are bugs to fix and code to write?
The quality of API documentation directly affects the experience that a
developer has employing or further developing a library.
It can be the difference between a piece of software being successfully integrated
into a larger project or being discarded because the user just can't figure out
how
it works.

The [Twisted API documentation](http://twistedmatrix.com/documents/current/api/) 
is automatically generated from comments embedded in the source code.
The [Twisted Coding Standards](http://twistedmatrix.com/documents/current/core/development/policy/coding-standard.html)
dictate that all methods, functions, classes, and modules have docstrings which
describe their purpose and detail all parameters and attributes.
The docstrings are written in 
[Epytext Markup Language](http://epydoc.sourceforge.net/manual-epytext.html) and are processed
by pydoctor, an API documentation generator, to produce the API documentation.
Epytext provides a way to specify formatting information, to annotate
parameters, return values, and their types, and to create cross-reference links.
Here's an example of a docstring written in Epytext:
``` python
def exists(address, domain):
    """
    Checks whether a user exists in a domain.

    @param address: The user
    @type address: L{Address}

    @param domain: The domain
    @type domain: L{IDomain}

    @rtype: C{bool}
    @return: C{True} if the user exists in the domain, 
             C{False} otherwise
    """
```

And here’s the API documentation that would be generated from that docstring:

{% img /images/exists.png %}

Currently, a good deal of Twisted Mail’s methods, functions, and classes are
missing docstrings. Even those methods and functions that have docstrings
often lack a full description of parameters, return values and their types.
My task is to provide that missing information.  It's a great opportunity to
dive deeply into the code.  And, once the code is fully documented, I'll find
it much easier to fix bugs and introduce new features.
