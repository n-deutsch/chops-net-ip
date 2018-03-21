# Chops - Connective Handcrafted Openwork Software Medley

The Chops Medley is a collection of C++ libraries for networking and distributed processing and simulation. It is written using modern C++ design idioms and the latest (2017) C++ standard.

This project is licensed under the terms of the MIT license.

## Chops Release Status

Release 0.2 is now (Feb 25, 2018) merged to the master branch:

- Significant changes have been made to the `start` method function parameters of the `basic_net_entity` class. There are two function object parameters for callbacks, the first corresponding to an IO state change, and the second to errors. This makes better conceptual sense and cleans up logical inconsistencies in the callback interface. Specifically:
  - IO state change callbacks correspond to a TCP connection being created or destroyed, or a UDP socket being opened or closed. There is one state change callback invocation for creation or open and one state change callback invocation for destruction or close. In both cases the `basic_io_interface` is tied to a valid IO handler. This allows for simpler state management and consistent associative container usage.
  - Error callbacks correspond to errors or shutdowns. These can happen in either the IO handling or in the higher level net entity handling. There may or may not be a valid IO handler referred to in the `basic_io_interface` object.
- The indirect memory leaks reported by address sanitizer have been fixed.
- A more consistent approach to exceptions and error returns is now in place for the `basic_io_interface` and `basic_net_entity` methods.

- UDP multicast support is the top priority for the next feature to be implemented.
- Strand design and support is under consideration.
- Multiple compiler support is under way, VC++ first.
- CMake file improvement is under way.

All tests run, although they are still limited (see next steps and constraints).

Release 0.1 is now (Feb 13, 2018) merged to the master branch:

- All initial Chops Net IP planned functionality is implemented except TCP multicast (there may be API changes in limited areas, see next steps below).
- All of the code builds under G++ 7.2 with "-std=c+1z" and is tested on Ubuntu 17.10.
- All tests are built with and run under the Catch2 testing framework.
- All code has been sanitized with "-fsanitize=address".

Known problems in release 0.1:

- Address sanitizer (Asan) is reporting indirect memory leaks, which appear to be going through the `std::vector` `resize` method on certain paths (where `start_io` is called from a different thread where most of the processing is occurring). This is being actively worked.
- The primary author (Cliff) is not happy with the function object callback interfaces through the `basic_net_entity.start` method (state change, error reporting callbacks). There are multiple possibilities, all of which have pros and cons. The message frame and message handler function object callback API is good and solid and is not likely to change.

### Next Steps, ToDo's, Problems, and Constraints:

- This is a good point to ask for project help and collaboration, which will be greatly appreciated (for many reasons).
- There are likely to be Chops Net IP bugs, specially relating to error and shutdown scenarios.
- Most of the testing has been "loopback" testing on one system. This will soon expand to distributed testing among multiple systems, specially as additional operating system build and testing is performed.
- Example code needs to be written and tested (there is a lot of code under the Catch framework, but that is not the same as stand-alone examples).
- The code is well doxygenated, and there is a good start on the high level descriptions, but tutorials and other high-level documentation is needed. A "doxygen to markdown" procedure is needed (or an equivalent step to generate the documentation from the embedded doxygen).
- The code only compiles on one compiler, but VC++ and Clang support (with the latest C++ standard flags) is expected soon. Compiling and building on Windows 10 is also expected to be supported at that time. Once multiple compilers and desktop environments are tested, development will expand to smaller and more esoteric environments (e.g. Raspberry Pi).
- Attention will be devoted to performance bottlenecks as the project matures.
- The makefiles and build infrastructure components are not yet present. A working CMakeLists.txt is needed as well as Github continuous integration procedures (e.g. Jenkins and Travis).
- Code coverage tools have not been used on the codebase.
- The "to do's" that are relatively small and short-term (and mentioned in code comments):
  - Implement multicast support
  - Investigate specific error logic on TCP connect errors - since timer support is part of a TCP connector, determine which errors are "whoah, something bad happened, bail out", and which errors are "hey, set timer, let's try again a little bit later"
  - UDP sockets are opened in the "start" method with a ipv4 flag when there is not an endpoint available (i.e. "send only" UDP entities) - this needs to be re-thought, possibly leaving the socket closed and opening it when the first send is called (interrogate the first endpoint to see if it is v4 or v6)

# Chops Major Components

## Chops Net IP

### Overview

Chops Net IP is an asynchronous general purpose networking library layered on top of the C++ Networking Technical Standard (TS) handling Internet Protocol (IP) communications. It is designed to simplify application code for processing data on multiple simultaneous TCP connections or UDP endpoints in an asynchronous, efficient manner. Every application interaction with Chops Net IP operations is no-wait (i.e. there are no blocking methods) and all network processing operations are performed asynchronously.

Example environments where Chops Net IP is a good fit:

- Applications that are event driven or highly asynchronous in nature.
- Applications where data is generated and handled in a non-symmetric manner. For example, data may be generated on the TCP acceptor side, or may be generated on a TCP connector side, or on both sides depending on the use case. Similarly, applications where the data flow is bi-directional and sends or receives are data-driven versus pattern-driven work well with this library.
- Applications interacting with multiple (many) connections (e.g. handling multiple sensors or inputs or outputs), each with low to moderate throughput needs (i.e. IoT environments, chat networks, gaming networks).
- Small footprint or embedded environments, where all network processing is run inside a single thread. In particular, environments where a JVM (or similar run-time support) is too costly in terms of system resource, but have a relatively rich operating environment (e.g. Linux running on a small chip) are a very good fit. (Currently the main constraint is small system support in the Networking TS implementation.)
- Applications with relatively simple network processing that need an easy-to-use and quick-for-development networking library.
- Applications with configuration driven networks that may need to switch (for example) between TCP connect versus TCP accept for a given connection, or between TCP and UDP for a given communication path.
- Peer-to-peer applications where the application doesn't care which side connects or accepts.
- Frameworks or groups of applications where abstracting wire-protocol logic from message processing logic makes sense.

Chops Net IP:

- simplifies the creation of various IP (Internet Protocol) networking entities including TCP acceptors and connectors, UDP senders and receivers, and UDP multicast senders and receivers.
- simplifies the resolution of network names to IP addresses (i.e. domain name system lookups).
- abstracts the message concepts in TCP (Transmission Control Protocol) and provides customization points in two areas:
  1. message framing, which is the code and logic that determines the begin and end of a message within the TCP byte stream.
  2. message processing, which is the code and logic that processes a message when the framing determines a complete message has arrived.
- provides buffer lifetime management for outgoing data.
- provides customization points for state changes in the networking entities, including:
  - a TCP connection has become active and is ready for input and output.
  - a UDP endpoint has been created and is ready for input and output.
  - a TCP connection has been destroyed or a UDP socket has closed.
- implements the "plumbing" for asynchronous processing on multiple simultaneous connections.
- abstracts many differences between network protocols (TCP, UDP, UDP multicast), allowing easier application transitioning between protocol types.
- allows the application to control threading (no threads are created or managed inside Chops Net IP).
- is agnostic with respect to data marshalling or serialization or "wire protocols" (application code provides any and all data marshalling and endian logic).
- does not impose any structure on network message content.

Chops Net IP is designed to make it easy and efficient for an application to create hundreds (or thousands) of network connections and handle them simultaneously. In particular, there are no threads or thread pools within Chops Net IP, and it works well with only one application thread invoking the event loop (an executor, in current C++ terminology).

A detailed overview is [available here](doc/net_ip.md).

## Future Components

### MQTT

(More info to be added.)

### Discrete Event Sim

(More info to be added.)

### Bluetooth

(More info to be added.)

### Serial

(More info to be added.)

# Chops Minor Components

## Timer

### Periodic Timer

The Periodic Timer class is an asynchronous periodic timer that wraps and simplifies C++ Networking Technical Standard (TS) timers when periodic callbacks are needed. The periodicity can be based on either a simple duration or on timepoints based on a duration.

Asynchronous timers from the C++ Networking Technical Specification (TS) are relatively easy to use. However, there are no timers that are periodic. This class simplifies the usage, using application supplied function object callbacks. When the timer is started, the application specifies whether each callback is invoked based on a duration (e.g. one second after the last callback), or on timepoints (e.g. a callback will be invoked each second according to the clock).

A detailed overview is [available here](doc/timer.md).

## Queue

### Wait Queue

Wait Queue is a multi-reader, multi-writer FIFO queue for transferring data between threads. It is templatized on the type of data passed through the queue as well as the queue container type. Data is passed with value semantics, either by copying or by moving (as opposed to a queue that transfers data by pointer or reference). The wait queue has both wait and no-wait pop semantics, as well as simple "close" and "open" capabilities (to allow graceful shutdown or restart of thread or process communication). A fixed size container (e.g. a `ring_span`) can be used, eliminating any and all dynamic memory management (useful in embedded or performance constrained environments). Similarly, a circular buffer that only allocates on construction can be used, which eliminates dynamic memory management when pushing or popping values on or off the queue.

Wait Queue is inspired by code from Anthony Williams' Concurrency in Action book (see [References Section](#references)), although heavily modified.

A detailed overview is [available here](doc/queue.md).

## Utilities

### Shared Buffer

Reference counted byte buffer classes are used within Chops Net IP, but can be useful in other contexts. These classes are based on example code inside Chris Kohlhoff's Asio library (see [References Section](#references)).

### Repeat

Repeat is a function template to abstract and simplify loops that repeat N times, from Vittorio Romeo (see [References Section](#references)). The C++ range based `for` doesn't directly allow N repetitions of code. Vittorio's utility fills that gap.

### Erase Where

A common mistake in C++ is to forget to call `std::erase` after calling `std::remove`. This utility wraps the two together allowing either a value to be directly removed from a container, or a set of values to be removed using a predicate. This utility code is copied from a StackOverflow post by Richard Hodges (see [References Section](#references)).

### Make Byte Array

Since `std::byte` pointers are used as a general buffer interface, a small utility function from Blitz Rakete as posted on Stackoverflow (see [References Section](#references)) is useful to simplify creation of byte buffers, specially for testing purposes.

A detailed overview of the utility classes is [available here](doc/utility.md).

# C++ Language Requirements and Alternatives

A significant number of C++ 11 features are in the implementation and API. There are also limited C++ 14 and 17 features in use, although they tend to be relatively simple features of those standards (e.g. `std::optional`, `std::byte`, structured bindings). For users that don't want to use the latest C++ compilers or compile with C++ 17 flags, Martin Moene provides an excellent set of header-only libraries that implement many useful C++ library features, both C++ 17 as well as future C++ standards (see [References Section](#references)).

Using Boost libraries instead of `std::optional` (and similar C++ 17 features) is also an option, and should require minimal porting.



While the main production branch of Chops will always be developed and tested with C++ 17 features (and relatively current compilers), alternative branches and forks for older compiler versions are expected. In particular, a branch using Martin's libraries and general C++ 11 (or C++ 14) conformance is expected for the future, and collaboration (through forking, change requests, etc) is very welcome. A branch supporting a pre-C++ 11 compiler or language conformance is not likely to be directly supported through this repository (since it would require so many changes that it would result in a defacto different codebase).

# External Dependencies

The libraries and API's have minimal (as possible) library dependencies (there are heavy dependencies on the C++ standard library in all of the code). There are more dependencies in the test code than in the production code. The external dependencies:

- Version 1.11 (or later) of Chris Kohlhoff's `networking-ts-impl` (Networking TS) repository is required for some components. Note that the version number is from the Asio version and may not exactly match the Networking TS version.
- Version 2.1.0 (or later) of Phil Nash's Catch 2 is required for all test scenarios.
- Version 0.00 (or later) of Martin's Ring Span Lite is required for some test scenarios.
- Version 1.65.1 or 1.66.0 of the Boost library (specific usages below).

See [Reference Section](#references) for additional details on the above libraries.

Specific dependencies:

- All test scenarios: Catch 2
- Chops Net IP (production): `networking-ts-impl`
  - Boost.Endian (test)
- Wait Queue (production): none
  - Ring Span Lite (test)
  - Boost.Circular (test)
- Periodic Timer (production): `networking-ts-impl`
- Shared Buffer (production): none

# References

- Chris Kohlhoff is a networking expert (among other expertises, including C++), creator of the Asio library and initial author of the C++ Networking Technical Standard (TS). Asio is available at https://think-async.com/ and Chris' Github site is https://github.com/chriskohlhoff/. Asio forms the basis for the C++ Networking Technical Standard (TS), which will (almost surely) be standardized in C++ 20. Currently the Chops Net IP library uses the `networking-ts-impl` repository from Chris' Github account.

- Vinnie Falco is the author of the Boost Beast library, an excellent building block library for asynchronous (and synchronous) HTTP and WebSocket applications. His Beast library uses Asio. He is proficient in C++ including presenting at CppCon and is also active in blockchain development and other technology areas. His Github site is https://github.com/vinniefalco. While Chops does not currently depend on Beast, the choices and design rationale made by Vinnie in implementing Beast are highly helpful.

- Phil Nash is the author of the Catch C++ unit testing library. The Catch library is available at https://github.com/catchorg/Catch2.

- Anthony Williams is the author of Concurrency in Action, Practical Multithreading. His web site is http://www.justsoftwaresolutions.co.uk and his Github site is https://github.com/anthonywilliams. Anthony is a recognized expert in concurrency including Boost Thread and C++ standards efforts. It is highly recommended to buy his book, whether in paper or electronic form, and Anthony is busy at work on a second edition (covering C++ 14 and C++ 17 concurrency facilities) now available in pre-release form.

- The Boost libraries collection is a high quality set of C++ libraries, available at http://www.boost.org/.

- Martin Moene is a C++ expert and member and former editor of accu-org. His Github site is https://github.com/martinmoene. Martin provides an excellent set of header-only libraries that implement many useful C++ library features, both C++ 17 as well as future C++ standards. These include `std::optional`, `std::variant`, `std::any`, and `std::byte` (from C++ 17) as well as `ring_span` (C++ 20, most likely). He also has multiple other useful repositories including an implementation of the C++ Guideline Support Library (GSL). 

- Andrzej Krzemieński is a C++ expert and proficient blog author. His very well written blog is https://akrzemi1.wordpress.com/ and a significant portion of the Chops code is directly influenced by it.

- Bjørn Reese writes about many topics in C++ at the expert level and is active with the Boost organization. His blog is http://breese.github.io/blog/.

- Richard Hodges is a prolific C++ expert with over 1,900 answers on StackOverflow as well as numerous discussions relating to C++ standardization. His Github site is https://github.com/madmongo1.

- Kirk Shoop is a C++ expert, particularly in the area of asynchronous design, and has presented multiple times at CppCon. His Github site is https://github.com/kirkshoop.

- Vittorio Romeo is a blog author and C++ expert. His web site is https://vittorioromeo.info/ and his Github site is https://github.com/SuperV1234. Vittorio's blog is excellent and well worth reading.

- Blitz Rakete is a student software developer who has the user id of Rakete1111 on many forums and sites (including Stackoverflow). His Github site is https://github.com/Rakete1111.

- Anders Schau Knatten has a C++ blog at https://blog.knatten.org/ and is the creator and main editor of the C++ quiz site http://cppquiz.org/. One of the best overviews of lvalues, rvalues, glvalues, prvalues, and xvalues is on his blog at https://blog.knatten.org/2018/03/09/lvalues-rvalues-glvalues-prvalues-xvalues-help/.

# Supported Compilers and Platforms

Chops has been compiled and tests run on:

- g++ 7.2, Linux (Ubuntu 17.10, Linux kernel 4.13)
- (TBD, will include at least clang on linux and vc++ on Windows)

# Installation

All Chops libraries are header-only, so installation consists of downloading or cloning and setting compiler include paths appropriately. No compile time configuration macros are defined in Chops.

# About

The primary author of Chops is Cliff Green, softwarelibre at codewrangler dot net. Cliff is a software engineer and has worked for many years writing infrastructure libraries and applications for use in networked and distributed systems, typically where high reliability or uptime is required. The domains where he has worked include wireless networks, location technology, and large scale embedded and simulation systems in the military aerospace industry. He has volunteered every year at CppCon and presented at BoostCon (before it was renamed to C++ Now).

Cliff lives in the Seattle area and you may know him from other interests including volleyball, hiking, railroading (both the model variety and the real life big ones), music, or even parent support activities (if you are having major difficulties with your teen check out the Changes Parent Support Network, http://cpsn.org).

Co-authors include ...

Contributors include ...

Additional information including various author notes is [available here](doc/about.md).

