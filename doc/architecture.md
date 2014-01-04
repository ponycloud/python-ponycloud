% PonyCloud Architecture

# Motto

> Virtual infrastructure should behave like a real one.

The real infrastructure never tells you that it's impossible to turn
off a server, you just pull the plug. In reality, popular open source
solutions will eventually introduce you to endlessly spinning spinners,
undeletable servers that are no longer in the rack and all other kinds of
new trouble you did not have with the plain old metal.

Whenever you, as the user or the operator, say that a constantly failing
operation shall not be carried out after all, the platform needs to
accomodate. It needs to stop trying and just listen to you. After all, you
are the one giving orders.


# Sparkle

Sparkle is a Python [Twisted][] daemon that conveys configuration stored in
a [PostgreSQL][] database to many Twilight-running hosts while
simultaneously providing managed access to the database via it's custom
RESTful API. It also holds information about current state of the platform
that is readily accessible to users and operators using the same API.

Whenever the database is not available, for example as a result of an
outage, Sparkle continues to manage all Twilight instances from memory.
This is possbile because it holds complete copy of configuration in it's
internal data structures in order to be able to serve user requests and
make decisions really fast.

When the platform is booting up and the database is in a managed instance,
Sparkle is able to inquire Twilights about last known bootstrap
configuration and start with that instead of a real database. With this
information the database will come up eventually. This makes small setups
without a dedicated controller entirely possible. With some external help,
Sparkle can even be made high-available in such settings.

## API

Endpoints are generated from schema using the parent/child relations.
Because some entity types have multiple parents, they can be reached
under different URLs.

There are two fundamental types of endpoints: collections and entities.

Collection endpoints support three HTTP methods.  `GET` to retrieve list
of child entities, `POST` to create a new child entity and finally,
`PATCH` that will apply a JSON Patch with collection as the root document.

Entities also support three HTTP methods.  `GET` to retrieve their state,
`DELETE` for their complete removal, and `PATCH` that will apply a JSON Patch
with entity as the root document.

Note that although `/collection/entity/collection/...` paths are used in
the API, the patches use a slightly different hierarchy that reflects the
presence of `children` element within every entity.


## Database Dictionary

Both primary and foreign keys are immutable.  They can only be changed
by using the `move` patch operation.  Since both source and destination
paths are validated, it is not possible to move an entity under different
tenant without being an administrator.

It is also not possible to perform a move along a different ownership path.
In other words, it is impossible to perform this move:

    {
      "op": "move",
      "from": "/cpu_profile/<cpu-profile-id>/instance/<instance-id>",
      "path": "/tenant/<tenant-id>/instance/<instance-id>"
    }

On the other hand, this move would be valid (for an administrator):

    {
      "op": "move",
      "from": "/tenant/<tenant-id-2>/instance/<instance-id>",
      "path": "/tenant/<tenant-id-1>/instance/<instance-id>"
    }


# Twilight

Twilight is a Python [Twisted][] daemon that translates configuration
obtained from Sparkle to state of the system it is running on. The most
notable responsibilities of Twilight are management of networks, both
traditional, management of storage for the distributed storage feature
and management of virtual machines (instances).

Twilight hosts are mostly stateless, which means that they can be booted
directly from the network (which is something Luna does, by the way). Once
running, Twilight reaches out to Sparkle in order to get it's configuration
and also hooks to [libvirt][] and udev to gather information about the
system. Aditional info is taken from sys.

Twilight hosts may be mostly stateless, but there is an exception to this
rule. Since each system has an unique `uuid` (we kind of assume the one
provided by the BIOS is, in fact, unique), we can look at the disks as they
come in and assemble the RAID with identical `uuid`. This RAID carries a
small (1GB) file system mounted on `/boot` that holds copy of Twilight
image plus the "bootstrap configuration" dump we mentioned in the Sparkle
section. The RAID has old-style metadata and carries a bootloader, which,
all combined, makes it possible to boot the whole platform even without a
dedicated controller. But more on that later.


# Messaging

All communication between Twilight and Sparkle is implemented in the form
of JSON messages transported by [ZeroMQ][] sockets in the `ROUTER` mode.
The messages from Twilight to Sparkle always include `uuid` of the machine
the Twilight is running on.

Messages from Sparkle do not need to include such identification, since
there is at most one active Sparkle instance at a time.

Example message from Sparkle to Twilight can look like this:

    {
      "event": "sparkle-resync"
    }

And a response from Twilight then:

    {
      "event": "twilight-update-state",
      "uuid": "9ece4546-1199-42b0-acf5-1b68f393683a",
      "incarnation": "35a446d6-6f8b-40bb-ac13-17eedf2e7c3e",
      "seq": 0,
      "changes": []
     }

# The Two States

As mentioned in the introduction, Twilight/Sparkle operate with two
so-called states. The "desired" state and the "current" state. Desired
state reflects contents of the PostgreSQL database. It represents the state
platform should eventually end up in. The current state, on the other hand,
reflects state the platform is in at any given moment.

The desired state is altered (with extremely small amount of exceptions) by
users or operators issuing API requests. With various consistency checks
implemented right in the database and authentication plus authorization
checks done by the Sparkle the desired state can only change in supported
ways.

The current state originates in the Twilight daemons as they join the
cloud, as new devices are discovered, as virtual instances change their
state etc.

Both states are replicated from their respective origin to the other
daemon. In other words, all changes done by users on Sparkle are quickly
replicated over to responsible Twilights and implemented, which leads to
replication of adjusted current state back to Sparkle from where users can
readily reach it.


## In-Memory Representation

Every entity/table in database have a corresponding object in memory. These
objects hold rows indexed by primary key(s) plus contain auxiliary indexes
on interesting columns. In addition to all this, a N:M relations using join
tables are also supported.

Each row is capable of holding both desired and current state in their
respective "part". Both these parts must share the same primary key for
pairing to work correctly and indexes cover them both.

This means that it is completely possible to query "host instances" and
obtain mix of instances that are running elsewhere but are in the process
of migration on this host and instances that have been there for ages and
have both state parts from this host.


## State Exchange

State exchange is not a very complicated process. Both peers start with an
incarnation, which is just a random number and a sequence identificator 1.
Since both parties expect first message to have sequence number 0 and
`null` incarnation, initial exchange always fails which triggers request
for full state resync.

Full resync is just a fancy expression for sending all relevant data in a
bulk with sequence number of 0 and an unchanged incarnation, which is now
fully accepted and the peer is ready to receive partial updates. Partial
updates bump sequence number to prevent missed updates (in which case the
full resync is requested again).

Since we want to detect missed messages quickly, an empty partial update is
sent every 15 seconds.


# Notifications

When something interesting happens on a Twilight, it is usually somehow
reflected in the current state. It can be also a trigger for notification.

Notifications are quick messages that do not partake in the state
replication. They usually serve as means by which one-time events such as
errors are reported back to Sparkle and users.

Each notification is bound to a specific host and can also be bound to a
specific virtual instance. This allows not only operators to learn about
platform issues, but also allows users to gain a bit better insight into
what goes on with their instances.


[PostgreSQL]: http://www.postgresql.org/
[ZeroMQ]:     http://www.zeromq.org/
[Twisted]:    http://twistedmatrix.com/
[libvirt]:    http://libvirt.org/
[KVM]:        http://www.linux-kvm.org/
