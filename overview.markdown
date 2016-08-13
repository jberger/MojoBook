## Overview

### The Relationship Between Mojo and Mojolicious

While the names Mojo and Mojolicious get used almost interchangably in common usage they are very distinct in a technical sense.
Mojolicious (and the other classes in the Mojolicious namespace) is the high level web framework.
It is built atop Mojo (and its related classes) which are a set of reusable tools used to implement web frameworks.

Mojolicious was historically (and in some sense still is) a reference implementation of a framework built on Mojo.
As Mojolicious has become popular however it has taken on an importance which most reference implementations never do.
The development of the two namespaces has been related and Mojo is well tailored to implement Mojolicious.

That said care is taken to keep Mojo separated from Mojolicious.
Most of the components you will see in a request response cycle are actually Mojo classes.
The servers, transactions, messages, templates, parsers, and encoders are all Mojo classes.
So how does a Mojo server serve a Mojolicious application?

It might come as a surprise but there are actually two levels of web application you can implement.
Your application is typically either an instance of Mojolicious (especially if it is a Lite app) or else an instance of a subclass of it.
However Mojolicious is actually a subclass of the basic web application, Mojo.

Mojo is a low-level web application "framework".
At a basic level a Mojo application implements a method called `handler`.
When a Mojo::Server (or subclass) gets a new request, it builds a Mojo::Transaction (technially it asks your app to to so, but this is usually pretty transparent).
The message parsing is done at this level, creating Mojo::Message instances for the request and the response.
Then the server calls your application's `handler` method with that transaction as an argument.

What the Mojolicious class does is implement this handler method in such a way as to provide the rich api that Mojolicious authors experience.
The routing is done via Mojolicious::Routes, the controllers are Mojolicious::Controllers.
The stash, flash, sessions and renderer are all implemented at the Mojolicious level.

If you are still confused, remember the following.
It is common for Mojolicious-level classes to build and use objects from the Mojo namespace.
Mojo-level classes never build Mojolicious level classes.

### The Transaction Object

When the server gets a request from its TCP socket, the first thing it does is get a Mojo::Transaction object to represent it.
For consistency, all classes in the Mojo echosystem that hold a transaction store it in an attribute called `tx`.
Additionally, by convention a variable that contains a transaction is called `$tx`.

This object is a container holding both a Mojo::Message::Request (`$tx->req`) and a Mojo::Message::Response (`$tx->res`), both of which are subclasses of Mojo::Message.
The request object will contain the parsed request after it has been parsed; the response object is where the application will define the message to be written to the client.

Interestingly, the server doesn't actually create the transaction object, it asks the application object to build one.
This is done so that your application can take some action very early in the request/response cycle.
In a pure Mojo app all it does is build the transaction object.
In a Mojolicious object it does that and also emits the `after_build_tx` hook (a hook is a special kind of application-wide event).
Technically an application class can overload the `build_tx` method that does this, but it is suggested not to do so but rather to make use of that emitted hook.

### The Controller Object

After the transaction is built and the Mojo handler is invoked now the Mojolicious framework is in full control.
On the first request the application does some setup, but on all requests the first thing that the application does is call `build_controller`.
In a Mojolicious application, the request/response cycle is governed by a Mojolicious::Controller instance (or an instance of a subclass).
A new instance is created for each request and is used throughout the handling of it from the routing to rendering a response.
The controller presents many useful methods and contains all of the data.
By convention, the variable that holds the controller is called `$c`.

The controller is the invocant for controller actions (the code body that is run to handle a request), router callbacks, and helpers (which are methods that share code application-wide), all of which will be discussed at length later.
They can be used to render templates or other data to the client.
Although normally a response is sent to the client all at once (a simple reply), data can also be sent in chunks, streamed, or even as a websocket for bidirectional communication with the client.

Remember that Mojolicious is built to be nonblocking and therefore all data must be contained on an instance of an object rather than in global variables.
The three most important data containers are the transaction (`$c->tx`), the stash (`$c->stash`) and the session (`$c->session`), of which, the transaction has already been discussed.

