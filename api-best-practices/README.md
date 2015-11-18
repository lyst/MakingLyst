API Design Best Practices
------------------------------

_version 0.3 2015-08-25_

This guide offers advice for design & development questions that you might encounter creating API services that use HTTP.

HTTP is a flexible transport protocol and there are many different ways to build an API service on top of it. There is no "true" way. But there are ways that support HTTP standards best to provide clear, useful services. This document avoids using the term "REST" throughout and aims to focus on good HTTP and use of hypermedia instead. This document aims to encourage your team to think more about how to use HTTP in the best way so you can connect clients to the service you have built.

## Contents

- [Ask Why](#ask-why)
- [Design](#design)
    - [Design First, Code Second](#design-first-code-second)
    - [Meet with Stakeholders](#meet-with-stakeholders)
    - [URLs and Methods](#urls-and-methods)
    - [Resources](#resources)
    - [Subresourcing](#subresourcing)
    - [HTTP Methods](#http-methods)
    - [Filtering](#filtering)
    - [Response Formats](#response-formats)
        - [Encoding](#encoding)
        - [Structured Response Formats](#structured-response-formats)
        - [Hypermedia](#hypermedia)
        - [Data types](#data-types)
        - [Pagination](#pagination)
    - [Response statuses](#response-statuses)
- [Develop](#develop)
    - [Tests](#tests)
    - [Endpoints](#endpoints)
    - [Identifying Resource Instances](#identifying-resource-instances)
    - [Terminology](#terminology)
    - [Form-encoded Data](#form-encoded-data)
    - [Supporting PATCH](#supporting-patch)
    - [ISO standards](#iso-standards)
    - [Authentication](#authentication)
    - [Statelessness](#statelessness)
    - [Protocols](#protocols)
    - [CORS](#cors)
- [Document](#document)
- [Support](#support)
    - [Versioning and backwards compatability](#versioning-and-backwards-compatability)
    - [Status website](#status-website)
- [Common Pitfalls](#common-pitfalls)
    - [Bad API resource endpoints](#bad-api-resource-endpoints)
    - [Using the wrong HTTP method](#using-the-wrong-http-method)
    - [Trying to be too RESTful](#trying-to-be-too-restful)
- [Credits](#credits)

For a Lyst-specific tooling guide, see the [tooling](TOOLING.md) document.


## Ask why

Why does your team want to build an HTTP API? This is the first thing you should ask yourself.

Is it to expose functionality to a microservice? Is it to share some data from a database? Sometimes HTTP won't be the best suited transport protocol for the solution. If your team need  asynchronous or real time communication then a protocol like [websockets](https://en.wikipedia.org/wiki/WebSocket) or [MQTT](https://en.wikipedia.org/wiki/MQTT) might be a better solution. Always consider this: HTTP is not always the answer. That said, HTTP can be the answer in many, many situations.


## Design

### Design First, Code Second

One of the biggest mistakes made by a team developing a new API service is to jump straight into the code. This is a bad practice as the team will not develop the correct solution the first time.

The first step to creating a good API service is to research the problem and design it. This should be done before any code is written. This will reduce the number of iterations required by the team to deliver the desired service. The designs will also help later when the team come to write documentation and build tests. Most designs are suitable as the foundation for great documentation and acceptance tests.


### Meet with Stakeholders

Part of the design process should be meeting with the stakeholders. These are the people who will be communicating with or using the API service.

Your team need to discuss with the stakeholders and define what data needs to be communicated. You also need to determine what functionality needs to be exposed. You should make an effort to keep the API service as simple as possible. If the API service looks like it is providing too much then it might need to be split up into smaller services.

After this you should start to separate the concerns of the data and functionality. Break down functionality and data into modularised, independent components. For example: a service that generates reports, and accepts new report data should provide two separate endpoints:

- Accepting new report data
- Generating reports

An endpoint that performs both of these services in one go is not functional or useful.

### URLs and Methods

After discovering the requirements you should begin a simple design document that addresses the URI endpoints in the new API service.

#### Resources

The URI endpoints need to address resources. These are the "things" your API service exposes. The HTTP standard [describes a resource](http://tools.ietf.org/html/rfc2616#section-1.3) as:

> _A network data object or service that can be identified by a URI._

This is a vague term and is often confused with an action or a method in an HTTP API service. Let's discuss what a resource is and how we can correctly decide what the resources should be in an API service.

Most of the data you want to provide in an API service is related to objects or models. This is especially true if you are using a relational database backend in your service. These objects can easily map to resources. Some common examples:

Internal object(s) | HTTP resource
-------------------|--------------
Report data | Reports
Product data | Products
User data | Users

Additionally HTTP resources do not have to be directly related to a single internal object. They can be an amalgamation of many objects where it makes sense.

Resources get harder to define when we are trying to provide functionality in an API service. When given this problem try and determine if the functionality creates things or gives you back any structured data. This can help to reveal the resources. Some examples:

Internal functionality | Data created from functionality | HTTP resource
-----------------------|---------------------------------|--------------
Follow a new user | A relationship object | Relationships
Generate reports | A report object | Reports
Order some products by input value | Products | Products
Return a list of related users ids for a given user ID | Users | Users


> **NOTE** Both examples above have the same `Products` and `Users` resource deliberately. If this was an API service that provided product data, but also provided products ordered by input data, then we need to rethink the scope of the API service we're providing. Clearly these have two different concerns that need to be separated.

Resources should always use a plural noun. This allows you to address collections of a resource and a single instance of a resource in a semantic way:

```bash
# A collection of users
/users
# A single user
/users/3a68a2a6-e7c8-49c3-8bae-2c3d0b00cac
```

#### Subresourcing

For related resources you can use subresourcing to make it easier to access related data. For example, getting all the user resources for an event resource:

```bash
# Return user resources for an event
/events/51bcc968-bacd-4f5b-bf19-15daef1ce87e/users
```

Subresourcing is semantically nicer than alternative ways of performing the same action. If you do not want to do subresourcing, an alternative would be to implement filtering on a resource. For example:

```bash
# Return users that match an event
/users?attending_event_uuid=51bcc968-bacd-4f5b-bf19-15daef1ce87e
```

#### HTTP Methods

Use HTTP Methods to perform actions on resources. Classically, HTTP methods are compared to CRUD functionality:

CRUD action | HTTP Method
------------|------------
Create | POST
Read | GET
Update | PUT
Delete | DELETE

Yet HTTP API services do not always provide CRUD functionality. In this case the temptation to throw caution to the wind and use which ever HTTP method often arises. Do not do this. Pick the most appropriate HTTP method for the action you are trying to perform:

Action | HTTP Method
-------|------------
Describe the available methods for a resource or return just the meta data for a resource. | HEAD
Return data about an instance or collection of resources. Return static data from the database based on a set of query parameters (such as filering). | GET
Perform an action that writes new data or instances to the database. Compute a complicated algorithm based on input data. Perform an action that creates a state change on the server. | POST
Perform an idempotent action that can be repeated more than once but does not create new data. | PUT
Destroy or delete data from a database, or cancel / disable an action. | DELETE

Also, there are _"safe"_ and _"unsafe"_ methods. Safe methods (GET, HEAD) should be used when actions do not cause any state change in the database or server. Unsafe methods (PUT, POST and DELETE) should be used when changing state on the database or server.

The PUT and DELETE methods are idempotent. Calling idempontent endpoints two, three,  or _n_ times has exactly the same result as calling it once. For example using DELETE to delete a resource instance twice will mean the instance is still deleted and no other resource will be affected.

#### Filtering

Filtering should be performed using query parameters as most filtering is performed on safe HTTP methods.

```bash
# Return all users who are aged 25 and are in the UK
/users?age=25&country=gb
```

In the situation where you need to pass a list of items to filter against (such as users aged 25 and users aged 26), pass duplicate parameters:

```bash
# Return all users who are aged 25 and 26
/users?age=25&age=26
```

### Response Formats

The response format is just as important as the endpoint of an API. Good response formats should be consistent, well structured, and versbose.

#### Encodings

[JSON](http://json.org) should be the default encoding used in response formats. JSON is widely supported and the defacto standard for nearly all API services on the web. You should provide alternative encodings such as [XML](http://www.w3.org/XML/) or [YAML](http://yaml.org) only when they are more suitable for the clients using the API service.

#### Structured response formats

The response formats should be similar between resources in your API service. They should follow standardised conventions and naming for common attributes. [JSON API](http://jsonapi.org) and [HAL](http://stateless.co/hal_specification.html) provide a common standard for describing structured response formats. These may be appropriate for your API service or they may not.

Regardless of chosen response format, be consistent throughout your API service and document it explictly.

#### Hypermedia

[Hypermedia](https://en.wikipedia.org/wiki/Hypermedia) should be used to provide links to related resources in response formats. This facilities a hypermedia driven way of navigating and using an API service programmatically. The client does not need to be taught the URLs and links to related resources, they are described in the responses instead. For example, the following response for an Event resource provides a hypermedia link to users attending that event:

```json
{
    "name": "My awesome event",
    "users": "https://myapi.com/events/51bcc968-bacd-4f5b-bf19-15daef1ce87e/users"
}
```

An API client can read this and determine that the "user" attribute is a related resource.

#### Data types

Always use strings in response formats. This includes decimal, floats and integer values. For example, every encoding standard handles floating point conversion slightly different. This can cause rounding errors. This can be dangers for services that work with monetary values like currency. To avoid this use strings wherever possible.

For URL links, provide full path URLs include the protocol and domain name. As HTTP request/responses are stateless, the reciever of the response might not be aware of the base URL for the API service. Providing full URL paths avoids this issue.

#### Pagination

Any sufficiently large collection of data should be paginated. When addressing collections of resources larger than 10-20 items use pagination to split up the collection. This increases response times. Paginated response formats should be standardised across your API service.
Here is an example paginated response format using JSON and paged pagination:

```json
{
    "next": "https://myapi.com/users?page=3",
    "previous": "https://myapi.com/users?page=1",
    "count": "40",
    "data": {
        ...
    }
}

```

There are many types of pagination. Use the best method that is most appropriate for your service:

- Paged pagination returns "pages" of data with a certain number of items.
- Limit/offset returns a number (the limit) of items, starting at a number (the offset).


### Response Statuses

HTTP provides [many status codes for free](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10). Your API service should use as many of these as possible. For example:

Status | HTTP Status code
-------|-----------------
Successfully retrieved a resource or collection of resources. | 200 OK
Written to the database and created new data. | 201 CREATED
Successfully processed the http request and queued an action. | 202 ACCEPTED
Removed data from the database successfully. The operation completed successfully, but nothing was generated. | 204 NO CONTENT
The request contained bad data or a domain-specific error was encountered. | 400 BAD REQUEST
No authentication was provided. | 401 UNAUTHORIZED
The authentication token does not grant access to this action. | 403 FORBIDDEN
The request resource was not found. | 404 NOT FOUND

For good responses use 200 OK by default and 201 CREATED when an action has generated content on the server.

Never use 5XX responses for **application** errors. All application errors should be caught and handled gracefully. 5XX responses should be reserved for service or server issues. Use the appropriate HTTP status code for errors that occur when using the API service.

If the standard HTTP status codes do not adequately describe the error then provide your own. Your own error responses should fall under a 400 BAD REQUEST response. These bespoke errors should follow a common format across the entire API service. Like all things, they must be verbose.

For example, an indexing API service that fails to properly index the supplied input values could return the following error response:

```json
HTTP/1.1 400 BAD REQUEST
{
    "error_code": "10",
    "title": "Index failure",
    "description": "The index service failed to index the supplied values.",
    "documentation": "https://myapi.com/documentation/errors/10"
}
```

The error response formats should be human readable for developers integrating with the service. They should provide a link to more information about the error. They should also provide a machine-readable error code so software clients can handle the errors.


## Develop

### Tests

The most prefered method of developing API services is to write the tests first. This should be easy to do if your team has spent time designing the URIs in the API service and the desired input / output data. Integration tests should be written that map to these designs. As well as writing tests that complete actions successfully, tests should also be written that cause bad things to happen. For example, sending malformed input data or simulating a server error in your API service. Writing tests that try to deliberately cause errors will encourage your team to develop a robust and error-free service.

### Endpoints

Write endpoints one at a time. Your designs should have made efforts to make sure your endpoints are modular enough for this. Developing the endpoints one at a time allows you to focus on the functionality of each endpoint. Afterwards, you can take the steps to link up the related endpoints. This is better than developing the functionality of the entire service in one go. For example, when developing the events `/users` example shown throughout this document, we might want to develop it in this order:

```
                Events endpoint
                      |
                      |
                      V
                Users endpoint
                      |
                      |
                      V
        Event users subresource endpoint
                      |
                      |
                      V
      Hypermedia links between Events and Users
```

### Identifying Resource Instances

Never use Primary Key IDs to identify individual items. This exposes potentially dangerous information to external services (such as your competitors!). This is especially releveant if your are developing a public API service for third party clients.

Instead, use UUIDs to uniquiely address instances.

### Terminology

Your API should use consistent and user-friendly terminology to denote resources and actions. It should not follow internal terminology. This is especially relevant if you are providing an API service for a third party client who is unaware of the internals of your API service. They do not need to know how it works inside.

### Form-encoded Data

You should accept JSON BODY data in POST and PUT requests by default. You should also accept form-encoded data too. This way, the client is less likely to cause an ambigious error because of the slight differences in the Content-Type sent. However, you should not support application/x-www-form-urlencoded, as this appends form data to the URL and can be extremely messy.

### Supporting PATCH

[PATCH](http://tools.ietf.org/html/rfc5789) is an extension method for HTTP that supports partial resource updates. This means your API service can support updating single attributes at a time instead of entire resources, which is what a strict PUT method would support. Try to support PATCH where you can. When you're unable to, try and make the PUT method support partial updates of attributes for resources too.

### ISO standards

For datetimes, always format using [ISO8601](http://www.iso.org/iso/home/store/catalogue_tc/catalogue_detail.htm?csnumber=40874) and always make datetimes UTC by default.

For country codes, always use [ISO3166](http://www.iso.org/iso/country_codes.htm).

When in doubt, use an ISO standard.

### Authentication

Unless you isolate your API service and it is only used on an internal network by trusted clients, you must provide some method of authentication.

The most basic is to provide token authentication. Generate a token that is shared between your client and server. The token should be sent as an HTTP header to the server when making requests. If the token is absent the request should return a 401 UNAUTHORIZED response. Token authentication can be used to user-agnostic, or can be associated with a user account.

For a more granular method of identifying user accounts through the API service, we recommend using [OAuth2](http://oauth.net/2/) authentication. Often considered complicated, OAuth2 is actually well supported in almost all languages.

Other authentication methods to consider include:

* [Hawk](https://github.com/hueniverse/hawk)
* [JSON Web Token](https://tools.ietf.org/html/rfc7519)

### Statelessness

A concept of HTTP is that request/response groups should be stateless between each other. For many this means "no storing of state on the server" or "don't use cookies". Both of these are true - we can use authentication tokens with each HTTP request to avoid cookies and state does not need to be maintained by the client or server.

More than that, statelessness also means everything required to make a successfull request and response should be included in the request and response: do not assume any stateful knowlege by the server or client. Always provide URL links, always use full URLs. Always return explicit clear information.

### Protocols

Only support HTTPS. HTTP is insecure and should never be used.

### CORS

Support [CORS](http://enable-cors.org) for cross site origin resource sharing. This will allow clients developing frontend applications to use your API service.


## Document

You should always write documentation for your API service. A brilliant API with no documentation is a bad API. Depending on the users and the purpose of the API service, different levels of documentation is considered acceptable. The only thing not considered acceptable is no documentation.

The design documents created at the start of the project can serve as a great Minimum Viable Document for describing your API service, but ultimately you should aim for a much better set.

For internal API services where your clients have access to the code repository, docstrings can be considered acceptable. For example, in Python you could provide the following docstring for an API endpoint:


```python
def post(self, request):
    """ Create a new user

    POST /users

    POST parameters:
    :param username: username of the new user.
    :param email: email of the new user.
    :param password: password for the new user.
    :returns: User resource instance.
    """
```

This is a Minimum Viable Document that allows your clients to figure out how to use this endpoint.

For large or public API services writing extensive documentation is essential. You should document every endpoint clearly and provide _"getting started"_ tutorials to get your clients familiar with the service.

Your examples should be language agnostic and use cURL examples that are easy to copy and paste. Most engineers understand cURL and can take the agnostic example and interpret it in their language of choice with little effort.

We recommend [MkDocs](mkdocs.org) as a simple markdown based document generator that can be uploaded to github pages.

## Support

Supporting your API service is as important as designing and developing it. Support for the API service should be considered in the design phase.

### Versioning and backwards compatability

Backwards compatability should never be broken. This means the first release of the API service should contain functionality that you never intend to deprecate. Once an API service is live and in the wild it is very hard to remove functionality.

Adding new functionality is not so tricky and is encouraged. Your clients may ask for new features (beware of scope creep) and new versions of the API service can be provided to accomodate them.

The difficulty of the feature addition should denote the action required to add the new feature. If you are just adding an attribute to a resource, that should require zero version bumping - just add it. For entirely new endpoints and functionality, ensure they are documented well.

If you do need to make breaking changes to an API service then you must use versioning. Verisioning API services is hard and there is no best way to do it. Verisioning using the URI is the easiest but not encouraged:

```bash
GET /api/v1/users
GET /api/v2/users
```

URI versioning is discouraged because it requires the client to do hard coded changes, usually manipulating a string.

A better but more complicated solution is to use HTTP Headers or content negotiation.

**HTTP Headers**

```bash
GET /users HTTP/1.1
Host: api.example.com
X-MY-API-VERSION: 1.0
```

**Content Negotiation**
```bash
GET /users HTTP/1.1
Host: api.example.com
Content-Type: application/my-api.v1.json
```

Both of these are complicated for the developer as they have to learn how to use them. However, they are much nicer and separate the URI and the version appropriately. If you opt to do this, make sure you implement a default version if no content negotiation is given to your API service.

A final method is to tie versions to the authentication tokens used by a client. This works so long as a client has a single token for a long time and is not using something like OAuth2, where tokens are refreshed regularly. This method is implict to the client. However they do not have to worry about versioning: for them the API service acts the same as always.

### Status website

If you are providing a public API service you must provide a public status dashboard. [Twilio's](http://status.twilio.com) status page is a good example. The status page should be hosted on an isolated server from the API. An API status page is no good if the page goes down when the API goes down.


## Common Pitfalls

Many API services do things well, many of those API services do things badly too. Here we address some common pitfalls, some with examples, of what **not to do**.

### Bad API resource endpoints

```bash
https://api-content.dropbox.com/1/files_put/auto/<path>?param=val
```

[Dropbox's _file\_put_](https://www.dropbox.com/developers/core/docs#files_put) endpoint states the method in the resource. This might be considered explict and helpful to the developer, but the documentation points out the methods allowed for the endpoint, so the semantic naming of the URI is pointless. Additionally, the `/files` endpoint is a similar resource but only supports the GET method. It's clear they should be grouped under the same URI.

### Using the wrong HTTP method

Flickr's "REST" API has a lot wrong with it, particularly it's vague distinction between GET and POST requests:

```bash
GET https://api.flickr.com/services/rest/?method=flickr.test.echo&name=value
POST https://api.flickr.com/services/rest/?method=flickr.test.echo&name=value
```

Regardless of the HTTP method used, these perform identical actions. Commonly, API services will use the POST method to accept parameters to endpoints that just request data. In situations like these, we believe poor effort has been made to design the endpoint and it's purpose. Always design your endpoints and consider the data being shared.

### Trying to be too RESTful

RESTful API services are very nice and consistent. In an ideal world they are the perfect HTTP API service. However, we often have to try and adapt HTTP to perform functionality that it was not originally intended for. Even common things like video streaming and e-commerce are concepts not considered when HTTP 1.0 and HTTP 1.1 were being developed. HTTP API services should be pragmatic and useful, but still not dangerously break the standards of HTTP.

At Lyst, we tried to develop a RESTful API endpoint that allowed us to do two things:

1. Update the details of a user if you have access rights.
2. "follow" the user by also sending `{"follow": "true" }`.

It's nice and RESTful because:

1. We're addressing a resource (user).
2. We're acting upon that resource (updating it, or following it).


However it wasn't practical. To the developer this endpoint was confusing. It performs two actions - it should be two separate endpoints! How do we fix this? In this instance we created a new logical subresource that we call "follows", which takes care of the follows functionality. We pass this the single `{"follow": "true"}` data we previously sent to the first endpoint.

```bash
# Update the following relationship between me and this user
PUT users/paul-hallett/follows
# Update the details of this user (assuming I have permission)
PUT users/paul-hallett
```

This is a more pragmatic and useful way of providing the different functionality whilst still sticking to good HTTP practices.


## Credits

This was written by [Paul Hallett](https://twitter.com/phalt_) for Lyst's engineering open guidelines.

We've tried to offer real-world and practical advice. If you find this useful or disagree, open an issue and start a discussion. If you find a spelling mistake, submit a pull request by following the contributing guide in [contributing](CONTRIBUTING.md).

This document was inspired by the following resources:

- GoCardless HTTP API Design Standards - [https://github.com/gocardless/http-api-design](https://github.com/gocardless/http-api-design)
- Interagent HTTP API Design - [https://github.com/interagent/http-api-design](https://github.com/interagent/http-api-design)
- Whitehouse API Design - [https://github.com/WhiteHouse/api-standards](https://github.com/WhiteHouse/api-standards)
- Build APIs You Won't Hate, Phil Sturgeon - [http://apisyouwonthate.com](http://apisyouwonthate.com)
