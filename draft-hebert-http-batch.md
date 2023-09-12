---
title: "HTTP Batched Request Format"
abbrev: "HTTP BATCH"
category: std

docname: draft-hebert-http-batch-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number: 1
date: 2023
consensus: true
v: 3
area: "Applications"
workgroup: "HyperText Transfer Protocol"
keyword:
 - I-D
 - http
 - batch
venue:
  group: "HyperText Transfer Protocol"
  type: "Working Group"
  mail: "http-wg@hplb.hp.com"
  arch: "https://www.ics.uci.edu/pub/ietf/http/hypermail"
  github: "GregoireHebert/draft-hebert-http-batch-00"
  latest: "https://GregoireHebert.github.io/draft-hebert-http-batch-00/draft-hebert-http-batch.html"

author:
 -
    initials: G.H.
    surname: Hébert
    fullname: Grégoire Hébert
    organization: Les-Tilleuls.coop
    email: gregoire@les-tilleuls.coop

normative:

informative:


--- abstract

This specification defines a set of formats for packaging multiple,
independent HTTP requests into a single payload and processing rules.
The formats are primarily intended for use with the HTTP POST method
as a means of providing applications a method of grouping sets of
individual HTTP requests for processing.

--- middle

# Introduction

This specification defines a set of formats for packaging multiple,
independent HTTP requests into a single payload and processing rules.
The formats are primarily intended for use with the HTTP POST method
as a mean of providing applications a method of grouping sets of
individual HTTP requests for processing. This approach is
designed to circumvent the lack of active pipelining in some HTTP/1.1 infrastructures,
and to work around a number of limitations that currently exist
such as the inability to pipeline a mix of idempotent and non-idempotent requests.
In addition, using HTTP/1.1, HTTP/2 or HTTP/3, it addresses the inability
of a server to determine the completeness and optimum order of processing
for a logical collection of requests before attempting to process any of
the individual requests. Leaving the choice of sequentialisation or parallelisation.

Note that batching HTTP requests using the mechanism defined here, is not
intended to replace pipelining.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# HTTP Batch Requests

A batch request MUST be represented as a Multipart MIME v1.0 Message [RFC2046],
a standard format allowing the representation of multiple parts, each of which
may have a different content type, within a single request.

## Headers

The client MUST use the multipart/mixed media type when the sub requests are independent.
By using the multipart/parallel media type, the client express the wish of a parallelisation
of the sub requests, but the server MAY NOT be able.

In addition to The header fields that begin with "Content-", each sub request MUST redefine
the start-line, with its own verb, URL, headers, and body. The type parameter of the content type
and each sub request MUST use the application/http Content-Type as specified by Section 10.1 of [RFC9112].
The HTTP request MUST only contain the path portion of the URL; full URLs are not allowed in batch requests.
The HTTP headers for the outer batch request, except for the Content- headers such as Content-Type,
apply to every request in the batch. If you specify a given HTTP header in both
the outer request and an individual call, then the individual call header's value overrides
the outer batch request header's value. The headers for an individual call apply only to that call.

## Body

Preambles and epilogues in the multipart batch request body, as defined in [RFC2046],
are valid but are assigned no meaning and thus MUST be ignored by processors of multipart batch requests.

The body of a batch request is made of a series of individual requests and change sets,
each represented as a distinct MIME part (i.e. separated by the boundary defined in the Content-Type header).
Each individual HTTP requests and response messages MUST be encapsulated using
the application/http Content-Type as specified by Section 10.1 of [RFC9112]. Each part MUST specify
a Content-ID header specifying a reference identifier for the HTTP request message.

```
POST /example/application HTTP/1.1
Host: example.org
Authorization: Basic amFtZXM6c25lbGw=
Content-Length: 123832
Content-Type: multipart/mixed;version=1.1;
msgtype=request;boundary=batch
Mime-Version: 1.0

--batch
Content-ID: <df536860-34f9-11de-b418-0800200c9a66@example.org>

POST /example/application HTTP/1.1
Host: example.org
Content-Type: text/plain
Content-Length: 3

Foo
--batch
Content-ID: <226e35d0-34fa-11de-b418-0800200c9a66@example.org>

PUT /example/application/resource HTTP/1.1
Host: example.org
Content-Type: image/jpg
Content-Length: 123456

{jpg image data}
--batch
Content-ID: <22fe12d0-ef12-e1de-c456-0834212c9a88@example.org>

GET /example/application/resource2 HTTP/1.1
Host: example.org
--batch--
```

## Request Dependencies

Requests within a batch SHOULD NOT have dependencies on other requests.

The HTTP server receives the batch request message and processes each included request
individually and independently of the others. The server can choose to process those messages
in parallel or in an asynchroneous way, to one another or in any order the server determines to be appropriate.

Once the batched request messages have been processed, a single HTTP Batch Response message
is created wherein each part contains the HTTP Response Messages.
Each part of the response MUST contain an Content-ID header specifying the Content-ID
of the HTTP request message that correlates to the response.

In case of an error for one of the sub requests, it is up to the service implementation to define rollback semantics
to undo any requests that may have been applied before another request and thereby apply an all-or-nothing requirement.

To simplify client processing of the response, the ordering of response messages in
the Batch Response SHOULD match the ordering of request messages in the Batch Request.
However, client applications MUST NOT assume such ordering of responses and
MUST use the Content-ID headers to correlate HTTP request and response messages
in the Batch Requests and Batch Responses.

While there are no dependencies between the subrequests, if some of them fails individually, but the root cause can
be identified by the serveur as one of the result of another requests within the changet set that the batch request, then
the server MAY use a 424 Failed Dependency HTTP Status as defined in [RFC4918] section 11.4

## Responding to a Batch Request

A server need not wait until all batched requests have been processed to create
and begin sending the Batch Response back to the client.

A server MUST determine the status of an HTTP request containing a Batch Request based solely
on its ability to successfully process the Batch Request as a whole and not on the status
of each individual contained message. For instance, if the server is capable of understanding
and processing the Batched Request, even if some or all of the contained requests fail individually,
the server SHOULD respond with a status of either 200 OK, 202 Accepted or 207 Multi-Status.
The 201 Created, 204 No Content, 205 Reset Content, and 206 Partial Content responses are not
appropriate responses for a successfully processed batch request.

While it is possible for Batch Request and Batch Response messages to be transferred using Chunked Transfer Coding, the individual HTTP request and response messages contained within MUST NOT use Chunked Transfer Coding.

Batch Responses are not cacheable. Individual HTTP response messages contained within the Batch Response might be cacheable.

A service MUST process the components of the Batch in the order received, when the
media type is multipart/mixed. A service MAY process the components of the Batch
asynchroneously, when the media type is multipart/parallel.

If the set of request headers of a Batch request are valid and the Content-Type is set
multipart/parallel, the service MAY return a 202 Accepted HTTP response code to indicate
that the request was accepted for processing, but the processing is yet to be completed.
The requests within the body of the batch may subsequently fail or be malformed; however,
this enables batch implementations to stream the results.

With this 202 status, the response SHOULD include an indication of the requests current status
and either include a Location header to a status monitor or some estimate of when the user can expect the request to be fulfilled with
an optional Retry-After header indicating the time the client should wait before querying the service for status.

A GET request to the status monitor resource again returns 202 Accepted response if the asynchronous processing has not finished.
This response MUST again include a Location header and MAY include a Retry-After header to be used for a subsequent request.
The Location header and optional Retry-After header may or may not contain the same values as returned by the previous request.
A GET request to the status monitor resource returns 200 OK once the asynchronous processing has completed.

```
HTTP/1.1 202 Accepted
Location: http://service/monitors/1
Retry-After: ###
```

Example when interrogating the monitor URL, only the first request in the batch has finished processing,
and all the remaining requests are still being processed. Note that the actual multipart batch response itself
is contained in an application/http wrapper as it is a response to a status monitor resource:

```
HTTP/1.1 200 Ok
Content-Type: application/http

HTTP/1.1 200 Ok
Content-Length: ####
Content-Type: multipart/mixed; boundary=batch

--batch
Content-Type: application/http
Content-ID: ###

HTTP/1.1 200 Ok
Content-Type: application/json
Content-Length: ###
Content-ID: ###

<JSON representation>

--batch
Content-Type: application/http
Content-ID: ###

HTTP/1.1 202 Accepted
Location: http://service/monitor/1
Retry-After: ###

--batch--
```

After some time the client makes a second request using the returned monitor URL,
not explicitly accepting application/http. The batch is completely processed and the response is the final result.

```
HTTP/1.1 200 Ok
Content-Type: application/http

HTTP/1.1 200 Ok
Content-Length: ####
Content-Type: multipart/mixed; boundary=batch

--batch
Content-Type: application/http
Content-ID: ###

HTTP/1.1 200 Ok
Content-Type: application/json
Content-Length: ###
Content-ID: ###

<JSON representation>

--batch
Content-Type: application/http
Content-ID: ###

HTTP/1.1 404 Not Found
Content-Type: application/problem+json

<JSON representation>
--batch--
```

If the Content-Type is set to multipart/mixed, the server's response is a single standard
HTTP response with a multipart/mixed content type; each part is the response to one of the requests
in the batched request.

Like the parts in the request, each response part contains a complete HTTP response,
including a status code, headers, and body. And like the parts in the request, each response part
is preceded by a Content-Type header that marks the beginning of the part.

While it is possible for Batch Request and Batch Response messages to be transferred
using Chunked Transfer Coding, the individual HTTP request and response messages contained within
MUST NOT use Chunked Transfer Coding.

```
HTTP/1.1 200 OK
Date: Wed, 29 Apr 2023 20:00:00 GMT
Content-Length: 291
Content-Type: multipart/mixed;version=1.1
;msgtype=request;boundary=batch
Mime-Version: 1.0

--batch
Content-ID: <226e35d0-34fa-11de-b418-0800200c9a66@example.org>

HTTP/1.1 415 Unsupported Media Type
Date: Wed, 29 Apr 2023 20:00:00 GMT
--batch
Content-ID: <22fe12d0-ef12-e1de-c456-0834212c9a88@example.org>

HTTP/1.1 401 Unauthorized
Date: Wed, 29 Apr 2023 20:00:00 GMT
--batch
Content-ID: <df536860-34f9-11de-b418-0800200c9a66@example.org>

HTTP/1.1 204 OK
Date: Wed, 29 Apr 2023 20:00:00 GMT
--batch--
```

# Referencing New Entities

Entities created by a POST request can be referenced in the request URL of subsequent requests.

# Security Considerations

It is possible that a malicious client could attempt to bypass security policies by batching requests
that otherwise would have been blocked by protective firewalls, proxies and server configurations.
Applications supporting batched requests must be prepared to deal with such requests and should be
implemented so that security policies are properly enforced on batched requests.

The HTTP request MUST only contain the path portion of the URL; full URLs are not allowed in batch requests.
Thus restricting the sub request to lie with the service domain, mitigating cross origin security issues and allowing
to forward and dispatch the request localy.

## Authorization

Each body part that represents a single request MUST NOT include authentication or authorization related HTTP headers,
Expect, From, Max-Forwards, Range, or TE headers.

If you provide an Authorization header for the outer request, then that header applies to all of the
individual calls.

When the server receives the batched request, it applies the outer request's query parameters and headers
(as appropriate) to each part, and then treats each part as if it were a separate HTTP request.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
