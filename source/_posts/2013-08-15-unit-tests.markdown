---
layout: post
title: "Unit Tests"
date: 2013-08-15 07:04
comments: true
categories: [OPW, Twisted]
---
One of the first Twisted mail issue tickets I tackled involved  what seemed to
be the simple matter of adding missing unit tests but led to some refactoring
of Twisted code.  The [ticket](http://twistedmatrix.com/trac/ticket/6383)
highlights the missing unit tests for the `startMessage` and `exists` functions
of both the `AbstractMaildirDomain` and `DomainQueuer` classes. 

`AbstractMaildirDomain` is a base class which is meant to be subclassed to
create mail domains where the emails are stored in the Maildir format.  Most of
the functions it provides are placeholders that need to be overridden in
subclasses.  However, it does provide implementations for two functions,
`exists` and `startMessage`.  `exists` checks whether a user exists in the
domain or an alias of it and, if so, returns a callable which returns a
`MaildirMessage` to store a message for the user. Otherwise, it raises an
`SMTPBadRcpt` exception.  `startMessage` returns a `MaildirMessage`
which stores a message for the user.

The existing unit tests for `AbstractMaildirDomain` were minimal, checking just
that it fully implemented the `IAliasableDomain` interface.
Additional test cases were needed to verify that:

* `startMessage` returns a `MaildirMessage` 
* for a valid user, `exists` returns a callable which
returns a `MaildirMessage` and that the callable returns distinct messages
when called multiple times
* for an invalid user, `exists` raises `SMTPBadRcpt`

`AbstractMaildirDomain` could not be used directly in testing `exists` and
`startMessage` because both call a function which has only a placeholder in the
base class.  So for testing purposes, I created a subclass of
`AbstractMaildirDomain`, `TestMaildirDomain`, which overrides the placeholder
functions.  

Since each test would need a `TestMaildirDomain` to exercise, I wrote a `setUp`
function which runs prior to the test and creates a `TestMaildirDomain` as well
as a temporary Maildir directory for it to use.  I also added a `tearDown`
function which runs after each test to remove the temporary directory.

``` python
class AbstractMaildirDomainTestCase(unittest.TestCase):

    def setUp(self):
        self.root = self.mktemp()
        self.service = mail.mail.MailService()
        self.domain = TestMaildirDomain(self.service, self.root)
        self.domain.addUser('user',"")

    def tearDown(self):
        shutil.rmtree(self.root)
```

The tests for `startMessage` and `exists` turned out to be pretty
straightforward:

``` python
    def test_startMessage(self):
        msg = self.domain.startMessage('user@')
        self.assertTrue(isinstance(msg, maildir.MaildirMessage))

    def test_exists(self):
        f = self.domain.exists(mail.smtp.User("user@", None, None, None))
        self.assertTrue(callable(f))
        msg1 = f()
        self.assertTrue(isinstance(msg1, mail.maildir.MaildirMessage))
        msg2 = f()
        self.assertTrue(msg1 != msg2)
        self.assertTrue(msg1.finalName != msg2.finalName)

    def test_doesntexist(self):
        self.assertRaises(mail.smtp.SMTPBadRcpt, self.domain.exists,
                mail.smtp.User("nonexistentuser@", None, None, None))
```

Like `AbstractMaildirDomain`, `DomainQueuer` acts as a domain, but instead of
saving emails for users, it puts messages in a queue for relaying. Its 
`exists` and `startMessage` functions are meant to be used in
the same way as those of `AbstractMaildirDomain`.   It seemed like it would be
an easy matter to adapt the unit tests for `AbstractMaildirDomain` to work for
`DomainQueuer`.

It turned out, however, that the test for `exists` on `DomainQueuer` failed
because a function was being called on a variable set to `None`.  Something
clearly hadn't been properly configured for the test.  

Here's the `DomainQueuer` code being exercised in the test:

``` python
class DomainQueuer:

    def exists(self, user):
        if self.willRelay(user.dest, user.protocol):
            ...
        raise smtp.SMTPBadRcpt(user)

    def willRelay(self, address, protocol):
        peer = protocol.transport.getPeer()
        return (self.authed or isinstance(peer, UNIXAddress) or
            peer.host == '127.0.0.1')
```

`exists` calls `willRelay` with the `protocol` which was passed into it as part
of the `user` argument.  `willRelay`  tries to call `getPeer` on the
`transport` instance variable of the `protocol`.  But, the minimal `User`
object passed to `exists`, which worked for the `AbstractMaildirDomain` test,
is not sufficient for the `DomainQueuer` test.  `willRelay` needs the `protocol`
to determine whether it should relay the message.

I had two thoughts about how to get around this problem, but I wasn't sure
either of them was satisfactory.  One option would be to provide the `User`
object with a fake protocol and fake transport so that `willRelay` could be
configured to return the desired value.  That seemed to be a very roundabout
way to solve the problem of getting `willRelay` to return a specific boolean
value.

A more direct way to get `willRelay` to return the desired value would be to
monkeypatch it.  That is, as part of the unit test, to replace the
`DomainQueuer.willRelay` function with one that returns the desired value.  The
problem with monkeypatching is that, even though `willRelay` is part of the
public interface of `DomainQueuer`, it could change in the future.  For
example, it could received additional optional arguments.  Then, the unit 
test which used the monkeypatched version might not reflect the behavior of
the real version of `willRelay`.

I got some great advice about how to approach this problem and unit testing in
general on the Twisted developers IRC channel.  The idea behind unit testing is
to test each unit of code independently.  Here we can consider `exists` and
`willRelay` to be two different units.  But the way `willRelay` is implemented
means that it is difficult to test `exists` independent of it.  I was advised
that sometimes the best thing to do when adding unit tests is to refactor the
code itself.  So, I attempted to do that without changing the public interface
of `DomainQueuer`.

I introduced a base class, `AbstractRelayRules`, whose purpose is to
encapsulate the rules for determining whether a message should be relayed.
Then, I defined a subclass, `DomainQueuerRelayRules`, which contains the
default rules for the `DomainQueuer` class.  

``` python
class AbstractRelayRules(object):
    """
    A base class for relay rules which determine whether a message
    should be relayed.
    """

    def willRelay(self, address, protocol, authorized):
        """
        Determine whether a message should be relayed.
        """
        return False


class DomainQueuerRelayRules(object):
    """
    The default relay rules for a DomainQueuer.
    """

    def willRelay(self, address, protocol, authorized):
        """
        Determine whether a message should be relayed.

        Relay for all messages received over UNIX sockets or from 
        localhost or when the message originator has been authenticated.
        """
        peer = protocol.transport.getPeer()
        return (authorized or isinstance(peer, UNIXAddress) or
            peer.host == '127.0.0.1')
```

In `DomainQueuer`, I changed the `__init__` function to take a new optional
parameter specifying the rules to be used to determine whether to relay.  When
`relayRules` is not provided, the default `DomainQueuerRelayRules` is used.  I
also changed the `willRelay` function to ask the `relayRules` whether to relay
instead of determining that itself.  Existing code which creates a
`DomainQueuer` without providing the extra argument works in the exact same way
as the old code. 

``` python
class DomainQueuer:

    def __init__(self, service, authenticated=False, relayRules=None):
        ...
        self.relayRules = relayRules
        if not self.relayRules:
            self.relayRules = DomainQueuerRelayRules()


    def willRelay(self, address, protocol):
        """
        Determine whether a message should be relayed according
        to the relay rules.
        """
        return self.relayRules.willRelay(address, protocol, self.authed)
```

Finally, I created another subclass of `AbstractRelayRules` to be used for
test purposes. `TestRelayRules` can be configured with a boolean value which
determines whether relaying is allowed or not.

``` python
class TestRelayRules(mail.relay.AbstractRelayRules):

    def __init__(self, relay):
        self.relay = relay

    def willRelay(self, address, protocol, authorized):
        return self.relay
```

Now, the unit tests for `DomainQueuer.exists` using `TestRelayRules` are quite
simple.

``` python
class DomainQueuerTestCase(unittest.TestCase):

    def test_exists(self):
        dq = mail.relay.DomainQueuer(self.service, False, TestRelayRules(True))
        f = dq.exists(mail.smtp.User("user@", None, None, None))
        self.assertTrue(callable(f))
        msg1 = f()
        self.assertTrue(isinstance(msg1, mail.mail.FileMessage))
        msg2 = f()
        self.assertTrue(msg1 != msg2)
        self.assertTrue(msg1.finalName != msg2.finalName)

    def test_doesntexist(self):
        dq = mail.relay.DomainQueuer(self.service, False, TestRelayRules(False))
        self.assertRaises(mail.smtp.SMTPBadRcpt, dq.exists,
                mail.smtp.User("user@", None, None, None))
```

Refactoring to decouple the rules for relaying messages from the
`DomainQueuer` certainly made the unit testing code much cleaner.  As a general
matter, difficulty in writing unit tests may highlight dependencies in the code
which should be refactored.

