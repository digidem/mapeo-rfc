# 0. About this Document

This document may not accurately reflect the current architecture of 
Mapeo Core, but it covers the proposed scope of improvements to be
implemented over the course of 2020.


# 1. Problems

Mapeo Core has several major drawbacks:

1. Does not adhere to the Single Responsibility Principle.
1. Entirely callback-based with no promise support.
1. Lifecycle management
1. No compatible RPC.

1.1. SRP

Although there have been improvements, Mapeo Core concerns itself with many
different aspects of the Mapeo application stack including:

* Importing and Exporting data
* Observations
* Media
* Wifi Sync state
* Peer state
* Sync files

This testing surface increases cognitive load for incoming developers, as the
Mapeo Core package handles all of these without clear separation, except for
Sync which is handled as a separate internal module. 

Furthermore, critical pieces for sync logic exist as duplicated code in both Mapeo
Desktop/Mobile client applications, which are necessary for sync to be reliable
and can cause inconsistent behavior between applications as well as make it
more difficult to test sync behaviors.

1.2 Callback-based

When Mapeo Core was originally developed, Promises were still quite new in
Node.js and did not have LTS support. Now, Mapeo Core is stable on Node
12.x and above, and core Node.js tooling for promises has become more
developed.

Callbacks require more Node.js knowledge as the approach is quite particular to
Node.js itself, whereas Promises adopt an easier-to-read and more approachable
toolset, especially for automated testing and client integration.

1.3 Lifecycle management

Mapeo Core does not manage it's own lifecycle and the lifecycle of components
it depends upon. Peer required dependencies `osm` and `media` must be the
correct versions that Mapeo Core respects, which could be incorrect without
notifying the developer, causing unexpected behavior or crash during program
execution, when the Mapeo Core library expects a different version of `osm` or
`media` than is passed to it. This puts a burden on the library consumer to use
the correct versions of all of these libraries. 

Mapeo Core also does not know about it's own lifecycle or the lifecycle of
other Mapeo Core instances on a single machine; this is akin to the PostgreSQL
prompt not being able to manage and create multiple databases. This can cause
headaches for developers and has resulted in two separate implementations in
Mapeo Desktop and Mobile of lifecycle management which are in some ways
reconcilable, but nevertheless rather complex to maintain.

In practice, client applications use Mapeo Server to instantiate a Mapeo Core
instance, which means that the applications cannot easily control the lifecycle
of Mapeo Core separately from the server.

1.4. No compatible RPC package

Applications that want to integrate into Mapeo Core need to use some sort of
RPC call to properly retrieve and control events to the Mapeo Core process.
Because there is no canonical implementation of an RPC package for Mapeo Core,
each client application must invent it's own RPC implementation. This means
higher barriers for testing and correctness for each application that wants to
be comaptible with Mapeo.

# 2. Proposal

These problems are all connected to the organization of the Mapeo Core library.
To address these problems, this RFC proposes a refactor of the application
layer that sits between the Client applications (client-specific frontend and user experience behavior) and the lower-level database (kappa-osm) and protocol (see RFC 002).

At a high level, these changes can be broken down into:

1. Overall architecture diagram
1. Application Intents API
1. Mono-repo under @mapeo/ npm organization


# 3. Architecture Diagram

# 4. Application Intents

The Intents API will be a high-level RPC interface for applications to directly
integrate with the Mapeo protocol. As a client creates an 'intent', it is
allowing the core library to make some assumptions about default behavior. This
is a common pattern in APIs from YouTube/Microsoft Azure/Android, to smaller
groups such as Cabal (`cabal-client`) library or Matrix.org `intents`.

TODO: Write Intent API


# 5. Mono Repo 

The mono repo will contain all application logic that does not include the
Mobile application and Desktop application themselves, but rather all of the
dependent modules. The following list is an initial sketch and there may be
pieces that need to be changed as exploration begins and implementation starts.

Using a mono repo for the application logic has certain benefits:

1. Particular versions and peer dependencies can be tested together and locked
   together on a particular package.
2. Development of new features that touch multiple modules and repositories can
   be locked to a particular mono branch in `@mapeo/core`, tracking and testing
   all dependencies at once.

```
- @mapeo/core
    - @mapeo/schema
    - @mapeo/sync
    - @mapeo/syncfile
    - @mapeo/server
    - @mapeo/settings
    - @mapeo/styles
    - @mapeo/default-settings
    - @mapeo/settings-builder
    - @mapeo/iD


    // New libraries

    - @mapeo/protocol
      |
      ---> multifeed
      ---> etc..

    - @mapeo/db
      |
      ---> kappa-osm
      ---> indexes

    - @mapeo/intents // Easy-to-use RPC API
    - @mapeo/manager // Managing multiple Mapeo instances on a single machine (see Desktop src/background/mapeo.js)
    - @mapeo/crypto //  Encapsulating the crypto modules
    - @mapeo/project // To discuss later
    - @mapeo/styles-builder // Creating new styles, using mapbox-style-downloader at first but encapsulate any underlying changes 
    - @mapeo/convert // Export/import mapeo data to different formats
    - @mapeo/observation // Logic for creating observations
    - @mapeo/filters // Any common filtering logic?
```
