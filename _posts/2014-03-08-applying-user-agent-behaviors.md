---
layout: post
title: Applying User-Agent Behaviors in Web Applications to Enable Runtime Extensions
published: false
---

The Web uses a typed-linking standard that's simple to use, and which achieves most of the same goals of [IDLs](http://en.wikipedia.org/wiki/Interface_description_language) without resorting to complex documents (such as [WSDL](http://en.wikipedia.org/wiki/Web_Services_Description_Language)). It's most commonly used in HTML:

```markup
<link rel="stylesheet" href="bootwhack.css">
```

Of course, this imports a stylesheet and decorates the page with it. Rather than exchange the details of the interface, the rel value ("stylesheet") indicates a handling spec which browsers implement as a runtime behavior. There's no need to publish machine-readable documents for generating bindings as the link itself provides the necessary definition. Behaviors are implicit, but well-defined.

If you look at the behavior model behind css links, you find it follows [reference-driven behaviors](http://en.wikipedia.org/wiki/HATEOAS) which use the reltypes to coordinate between unfamiliar clients and services. As browsers receive links, they interpret them to create the application; the references are the limit of familiarity between the nodes. This enables smart client behaviors without strong binding to the remote endpoints, unlike modern JS Web apps which don't formally declare their references and so include implicit knowledge of the endpoints.

While we traditionally leave reference-driven programming to the browsers, there's no reason the Web applications themselves can't take on the same model. As such, I'll call these "User Agent" behaviors, as a style which page javascript or desktop apps may use as well as browsers.

The User Agent model uses HTTP as a messaging language. Links (like the stylesheet) are functionally identical to the indexes in databases, and it's apt to view the User Agent as creating a "session database" as it navigates and accumulates references. The User Agent runs queries against the attributes in links, and derives the meanings of values from the `reltype` attributes. If the User Agent doesn't know a reltype, it doesn't support the service on the other end. Compatibility is, therefore, automatically negotiated.

Traditionally, HTTP is viewed as an over-the-network protocol, but its message format can apply within a process or between processes as a language-agnostic interface. Javascript's recent adoption of promises makes the Ajax interface much easier to integrate with the syntax and much easier to syncronize, and so HTTP is a suitable (and, in-fact, straight-forward) messaging standard for webapp dev.

The challenges of net-service interoperation are similar to that of process/plugin interop (identify capabilities, establish the interface, reason about remote state) and so the same techniques apply within a single machine. An HTTP-messaging environment implements a local hostmap and routes its requests to functions, peer processes, and across the IP network as usual.

A brief example from [a JS implementation of HTTP messaging, called Local.js](https://grimwire.com/local):

```javascript
// Add a function to the httpl hostmap
local.addServer('myhost', function(req, res) {
  if (req.method == 'GET' && req.path == '/') {
    res.writeHead(200, 'OK');
    res.end('Hello, world');
  } else {
    res.writeHead(404, 'Not found').end();
  }
});
// Dispatch to the function
local.GET('httpl://myhost').then(function(res) {
  console.log(res.body); // => 'Hello, world'
});
```

It should be immediately clear that there's an advantageous uniformity to this interface, as a Web service may be replaced with a function simply by changing the URI. In cases where an application may need to support offline operation, it may implement local proxies which read from the cache and queue operations during network downtime. There's also some utility in team-coordination, as frontend developers can implement local scaffolds of services, then share the scaffold with backend implementers.

Another exploitable benefit is in describing GUI interactions. By registering catchall event handlers at the root of the document (the `<body>`) applications can specify the interactions in HTML, rather than with individual event-listeners:

```markup
<a href="httpl://user.gui" method="REFRESH">Refresh Profile</a>
<form action="httpl://signup" method="POST">
  <p>First name: <input type="text" name="fname"></p>
  <p>Email: <input type="email" name="email"></p>
  <p><button type="submit">Signup</button></p>
</form>
```

[TodoSOA](https://grimwire.com/todosoa/) uses the TodoMVC application-mold to illustrate some of these techniques. ([Detailed explanation here](https://grimwire.com/local/#docs/todosoa.md).)

However the real value of HTTP messaging derives from [IPC](http://en.wikipedia.org/wiki/Inter-process_communication) between unfamiliar nodes. In the context of a single machine, this enables programs/plugins to coordinate and extend each-other's capabilities at runtime. If [virtual machines](http://en.wikipedia.org/wiki/Virtual_machine) are used to enforce security barriers, HTTP may be used as a bridge across their trust zones, and this is the principle behind leveraging [Web Workers](http://en.wikipedia.org/wiki/Web_worker). Bridging trust zones enables a range of benefits, including freely-shared user plugins and proper data-containment during 3rd-party code execution, as will be outlined in a future post.

This form of interoperation requires that we generate requests toward links rather than to predefined URLs, as links confer interface guarantees. Referring back to the User Agent model, we'll require tools to: transfer & collect links, query against them, and (for performance reasons) parameterize their URIs. Most of these tools are readily available in existing standards.

To transfer links, the HTML `<link>` element is available, but poorly-suited to the protocol, as are other content-type-specific formats such as [JSON-HAL](http://stateless.co/hal_specification.html), as they exclude the ability to use other content-types. This is not to say HTML or JSON-HAL are ineffective in all cases; they are simply ill-suited to the HTTP flow. Instead, the Link header ([defined in RFC5988](http://tools.ietf.org/html/rfc5988)) offers an elegant alternative which can be retrieved specifically with [HEAD requests](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.4).

The Link format's [relation types](http://tools.ietf.org/html/rfc5988#section-4) carry some notable features as RFC5998 defines. The first is that multiple reltypes may be used, and, while the inclusion of one should not affect the other, the definitions of both apply equally. This can be used to support highly-generalized definitions, such as "CRUD" behaviors, in addition to more specific type-definitions such as [schemas](http://schema.org/Thing). The second notable feature is that of the "extension types," which are labelled using URIs under the control of the reltype developer. Documentation and additional resources for the reltype can be hosted at the URL, giving a clear procedure for distributing specs between organizations.

To query against the links, a User Agent can use various optimization techniques, but the broad process (which I'll call the "index scan") is simply to iterate the links and run tests against their attributes. In the tests, the reltype takes particular prominence, as it determines the semantics of the other attributes. The description of values to test against - the query - becomes a descriptive reference within the client code.

An example from the Local.js toolset:

```javascript
var links = parse(response.header('Link')); // where `parse` produces an object format
local.queryLinks(links, { rel: 'stylesheet' });
// => [{ rel: 'stylesheet', href: 'bootwhack.css' }]
```

Though it is not explicit in RFC5988, the links imply an order which is followed during the index scan. In cases where multiple matches are found, yet only one reference is needed, the client gives preference to the earliest match. As a result, the server can specify priority through the order, and should put the more specific links earlier in the index to match them first.

For example:

```
Link: <bootwhack-3.1.0.css>; rel="stylesheet bootwhack.com/rel/versioned"; vmajor="3"; vminor="1"; vpatch="0",
<bootwhack-3.0.0.css>; rel="stylesheet bootwhack.com/rel/versioned"; vmajor="3"; vminor="0"; vpatch="0"
```

In this case, a search for a specific version (eg `{rel:"stylesheet bootwhack.com/rel/versioned", vmajor: 3, vminor: 0, vpatch: 0}`) will match the second entry, while a more general query (eg `{rel:"stylesheet bootwhack.com/rel/versioned", vmajor: 3}`) will take the first match - in this case, 3.1.0 (the latest).

It's not hard to imagine that the size of the Link header can grow to unmanageable sizes, degrading transfer- and scan-performance. This can be mitigated using query parameters to control how many links are given, for instance with pagination. However, another more general solution is to use [URI Templates](http://tools.ietf.org/html/rfc6570) in order to compress the index. Values which are expected in the link's attributes are instead included as tokens in the URI.

For example:

```
Link: <bootwhack-{vmajor}.{vminor}.{vpatch}.css>; rel="stylesheet bootwhack.com/rel/versioned"
```

During the index scan, URI tokens are treated as matches, then populated with the query's values. For instance, the query for `{vmajor: 3, vminor: 0, vpatch: 0}` will produce `bootwhack-3.0.0.css`. Unfortunately, the UriTemplates standard doesn't support default values, and so attributes which require fallbacks must use additional links:

```
Link: <bootwhack-{vmajor}.{vminor}.{vpatch}.css>; rel="stylesheet bootwhack.com/rel/versioned",
<bootwhack-{vmajor}.{vminor}.0.css>; rel="stylesheet bootwhack.com/rel/versioned"; vpatch=0,
<bootwhack-{vmajor}.1.0.css>; rel="stylesheet bootwhack.com/rel/versioned"; vminor=1; vpatch=0,
<bootwhack-3.1.0.css>; rel="stylesheet bootwhack.com/rel/versioned"; vmajor=3; vminor=1; vpatch=0
```

These techniques are combined in Local.js to create an `Agent` helper which behaves somewhat like a database cursor. It follows the fetch, scan, construct process, then produces a new agent at the discovered location which maintains a back-reference to its parent. It calls this a "navigation," and it can chain navigations to form more complex queries:

```javascript
local.agent('http://myservice.com')
  .follow({ rel: 'foo.com/rel/coll', id: 'users' })
  .follow({ rel: 'foo.com/rel/item schema.org/person', id: 'bob' })
  .GET();
```

In addition to the two navigation queries, a starting point is supplied. Until a request is made (in this case, GET) the navigations are left unresolved, making them "lazy." Once the target URL is needed for a request, the agent works up its ancestor chain to the starting point, then runs the sequence of HEAD requests and index-scans to arrive at the endpoint. The example above would result in 3 requests (2 HEAD, 1 GET) but could cache the agents in subsequent use so that the 2 HEAD requests only occur at initialization.

If any navigations fail, the response promise is "rejected" with a status 1, Link Query Failed. This means that a successful response will carry the guarantees of the reltypes used, and the client can (in this case) expect the [Person schema](http://schema.org/person).

With these tools, we've tackled the question of interfacing, but still need to prepare Web Workers to use them. The first matter is message de/serialization, and, while HTTP's native wire format is available, I've opted to use a [json-encoded variant](https://github.com/pfraze/grimwire/wiki/HTTPL%3A-JSON-encoded-message-streams-with-HTTP-semantics) in Local.js for performance reasons, and to introduce multiplexing into the WebWorker `postMessage` channel. The second matter is lifecycle, as workers must be active to handle the request, and the third matter is of naming (assigning URIs). These two items feed into each other.

Local.js' URI scheme, `httpl://`, uses the format HTTP/S, but creates its own hostmap unrelated to DNS. However, `httpl` includes an extension to the authority segment called the "source path," and it is used to construct universal hostnames for workers. It appears as in this example:

```
httpl://my-service.com(path/to/worker.js)/
```

Within the parenthesis is a path which, when concatenated to the hostname, will provide the javascript to construct the script. In this case, that URI would be `https://my-service.com/path/to/worker.js`.

This extension enables Local.js to automate the construction and lifecycle of Workers. If an httpl request maps to a missing host, it checks for a source-path in order to load the Worker and fulfill the response. After fulfillment of all queued requests, the Worker is immediately deconstructed to preserve resources. The environment can choose to keep the Worker alive, either using Local.js APIs, or by leaving an open request. As a result, links are free to reference resources hosted by these Workers, and Worker lifecycle is mostly automated.

[A demo of this behavior.](http://dev.grimwire.com/env.html)

I've experimented with variations on this architecture in different applications, including also the use of WebRTC as an HTTPL channel. Automation of Worker lifecycle is the most recent addition, prompted by the difficulty that manual management presented users. At present, I'm experimenting with chat which automatically fetches the metadata of "spoken" URIs (via HEAD request) and attempts to incorporate their resource into a dynamically-constructed application. This is off to a promising start, though it's led me to discover that, as it happens, mimetypes are *not* reliably-applied on the Web. (Expect posts in the future on the development of type-classifiers.)

What does appear to work well &ndash; when HEAD requests yield usable data &ndash; is the use of the URI as a principle tool. Because workers' services are now universally-addressable, users can easily publish dynamic behaviors via CDN. Workers are restricted to only messaging the page, and so can have permissions and data-containment applied. As I mentioned before, I'll write more on this in the future.

As a post-script, I'll mention an experimental URI scheme, `nav:||`, which serializes User Agent queries in a format which can be embedded in HTML and supplied to the Ajax interface.

```javascript
local.agent('http://foobar.com')
  .follow({ rel: 'collection', id: 'users' })
  .follow({ rel: 'item', id: 'bob' })
  .GET();
```

May be rewritten as:

```javascript
local.GET('nav:||http://foobar.com|collection=users|item=bob');
```

The scheme is constructed of segments separated with vertical pipes. The first segment supplies the start-URI, with subsequent segments each defining a navigation. Navigation segments use a slightly complex notation:

`|` reltypes [`=` id] [`,` attribute `=` value]*

Multiple reltypes may be combined with a `+`, and so the query `{ rel: 'collection foo.com/rel/users', online: 1 }` would be written `|collection+foo.com/rel/users,online=1`. Somewhat intriguingly, the starting url can be dropped to create a "relative nav URI," creating a generalized resource-description. For instance, `nav:|collection+foo.com/rel/users,online=1` describes "online users."

I've yet to use the nav scheme in an application, but find it interesting for a possible future use.

~pfrazee