.. index:: Activator, template

Getting Started
===============

The easiest way to get started is with a working application in Typesafe Activator_. The following
templates are created by the Slick team, with updated versions being made for new Slick releases:

* To learn the basics of Slick, start with the `Hello Slick template`_. It contains an extended
  version of the tutorial and code from this page.

* The `Slick Plain SQL Queries template`_ shows you how to do SQL queries with Slick.

* The `Slick Multi-DB Patterns template`_ shows you how to write Slick applications that can use
  different database systems and how to use custom database functions in Slick queries.

* The `Slick Direct Embedding template`_ shows you how to use Slick's experimental Direct Embedding
  query API.

* The `Slick TestKit Example template`_ shows you how to use Slick TestKit to test your own Slick drivers.

There are more Slick templates created by the community, as well as versions of our own templates for other
Slick releases. You can find `all Slick templates <https://typesafe.com/activator/templates#filter:slick>`_
on the Typesafe web site.

.. index:: Maven, sbt, artifacts, build, dependency

Adding Slick to Your Project
============================

To include Slick in an existing project use the library published on Maven Central.  For sbt projects add the
following to your build definition - ``build.sbt`` or ``project/Build.scala``:

.. parsed-literal::
  libraryDependencies ++= Seq(
    "com.typesafe.slick" %% "slick" % "|release|",
    "org.slf4j" % "slf4j-nop" % "1.6.4"
  )

For Maven projects add the following to your ``<dependencies>``:

.. parsed-literal::
  <dependency>
    <groupId>com.typesafe.slick</groupId>
    <artifactId>slick_2.10</artifactId>
    <version>\ |release|\ </version>
  </dependency>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-nop</artifactId>
    <version>1.6.4</version>
  </dependency>

.. index:: logging, SLF4j

Slick uses SLF4J_ for its own debug logging so you also need to add an SLF4J
implementation. Here we are using ``slf4j-nop`` to disable logging. You have
to replace this with a real logging framework like Logback_ if you want to see
log output.

Quick Introduction
==================

*Note:* The rest of this chapter is based on the `Hello Slick template`_. The prefered
way of reading this introduction is in Activator_, where you can edit and run the code
directly while reading the tutorial.

To use Slick you first need to import the API for the database you will be using, like:

.. includecode:: code/FirstExample.scala#imports

Since we are using H2_ as our database system, we need to import features
from Slick's ``H2Driver``. A driver's ``simple`` object contains all commonly
needed imports from the driver and other parts of Slick such as
:doc:`session handling <connection>`.

.. _gettingstarted-dbconnection:

Database Connection
-------------------

In the body of the application we create a ``Database`` object which specifies
how to connect to a database, and then immediately open a session, running all
code within the following block inside that session:

.. includecode:: code/FirstExample.scala#setup

In a Java SE environment, database sessions are usually created by connecting
to a JDBC URL using a JDBC driver class (see the JDBC driver's documentation
for the correct URL syntax). If you are only using
:doc:`plain SQL queries <sql>`, nothing more is required, but when Slick is
generating SQL code for you (using the :doc:`direct embedding <direct-embedding>`
or the :ref:`lifted embedding <lifted-embedding>`), you need to make sure to use
a matching Slick driver (in our case the ``H2Driver`` import above).

Schema
------

We are using the :ref:`lifted embedding <lifted-embedding>` in this
application, so we need ``Table`` row classes and ``TableQuery``
values for our database tables. You can either use the :doc:`code generator <code-generation>`
to automatically create them for your database schema or you
can write them by hand:

.. includecode:: code/FirstExample.scala#tables

All columns get a name (usually in camel case for Scala and upper case with
underscores for SQL) and a Scala type (from which the SQL type can be derived
automatically).
The table object also needs a Scala name, SQL name and type. The type argument
of the table must match the type of the special ``*`` projection. In simple
cases this is a tuple of all columns but more complex mappings are possible.

The ``foreignKey`` definition in the ``coffees`` table ensures that the
``supID`` field can only contain values for which a corresponding ``id``
exists in the ``suppliers`` table, thus creating an *n to one* relationship:
A ``Coffees`` row points to exactly one ``Suppliers`` row but any number
of coffees can point to the same supplier. This constraint is enforced at the
database level.

Populating the Database
-----------------------

The connection to the embedded H2 database engine provides us with an empty
database. Before we can execute queries, we need to create the database schema
(consisting of the ``coffees`` and ``suppliers`` tables) and insert some test
data:

.. includecode:: code/FirstExample.scala#create

The ``TableQuery``'s ``ddl`` method creates ``DDL`` (data definition language) objects
with the database-specific code for creating and dropping tables and other
database entities. Multiple ``DDL`` values can be combined with ``++`` to
allow all entities to be created and dropped in the correct order, even when
they have circular dependencies on each other.

Inserting the tuples of data is done with the ``+=`` and ``++=`` methods,
similar to how you add data to mutable Scala collections. Note that by default
a database ``Session`` is in *auto-commit* mode.
Each call to the database like ``+=`` or ``++=`` executes atomically
in its own transaction (i.e. it succeeds or fails completely but can never
leave the database in an inconsistent state somewhere in between). In this
mode we have to populate the ``suppliers`` table first because the
``coffees`` data can only refer to valid supplier IDs.

We could also use an explicit transaction bracket encompassing all these
statements. Then the order would not matter because the constraints are only
enforced at the end when the transaction is committed.

Querying
--------

The simplest kind of query iterates over all the data in a table:

.. includecode:: code/FirstExample.scala#foreach

This corresponds to a ``SELECT * FROM COFFEES`` in SQL (except that the ``*``
is the table's ``*`` projection we defined earlier and not whatever the
database sees as ``*``). The type of the values we get in the loop is,
unsurprisingly, the type parameter of ``Coffees``.

Let's add a *projection* to this basic query. This is written in Scala with
the ``map`` method or a *for comprehension*:

.. includecode:: code/FirstExample.scala#projection

The output will be the same: For each row of the table, all columns get
converted to strings and concatenated into one tab-separated string. The
difference is that all of this now happens inside the database engine, and
only the resulting concatenated string is shipped to the client. Note that we
avoid Scala's ``+`` operator (which is already heavily overloaded) in favor of
``++`` (commonly used for sequence concatenation). Also, there is no automatic
conversion of other argument types to strings. This has to be done explicitly
with the type conversion method ``asColumnOf``.

Joining and filtering tables is done the same way as when working with Scala
collections:

.. includecode:: code/FirstExample.scala#join

Note the use of ``===`` instead of ``==`` for comparing two values for
equality. Similarly, the lifted embedding uses ``=!=`` instead of ``!=`` for
inequality. (The other comparison operators are the same as in Scala:
``<``, ``<=``, ``>=``, ``>``.)

The generator expression ``suppliers if s.id === c.supID`` follows the
relationship established by the foreign key ``Coffees.supplier``. Instead of
repeating the join condition here we can use the foreign key directly:

.. includecode:: code/FirstExample.scala#fkjoin
