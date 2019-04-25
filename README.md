<a href="http://hapijs.com"><img src="https://raw.githubusercontent.com/hapijs/assets/master/images/family.svg" width="180px" align="right" /></a>

# hawk

**Hawk** is an HTTP authentication scheme using a message authentication code (MAC) algorithm to provide partial
HTTP request cryptographic verification.

Note: the protocol has not changed since version 1.1. The version increments reflect changes in the node API.

[![Build Status](https://travis-ci.org/hapijs/hawk.svg?branch=master)](https://travis-ci.org/hapijs/hawk)

# Table of Content

- [**Introduction**](#introduction)
  - [Replay Protection](#replay-protection)
  - [Usage Example](#usage-example)
  - [Protocol Example](#protocol-example)
    - [Payload Validation](#payload-validation)
    - [Response Payload Validation](#response-payload-validation)
  - [Browser Support and Considerations](#browser-support-and-considerations)
  - [hapi Plugin](#hapi-plugin)
- [**Single URI Authorization**](#single-uri-authorization)
  - [Usage Example](#bewit-usage-example)
- [**Security Considerations**](#security-considerations)
  - [MAC Keys Transmission](#mac-keys-transmission)
  - [Confidentiality of Requests](#confidentiality-of-requests)
  - [Spoofing by Counterfeit Servers](#spoofing-by-counterfeit-servers)
  - [Plaintext Storage of Credentials](#plaintext-storage-of-credentials)
  - [Entropy of Keys](#entropy-of-keys)
  - [Coverage Limitations](#coverage-limitations)
  - [Future Time Manipulation](#future-time-manipulation)
  - [Client Clock Poisoning](#client-clock-poisoning)
  - [Bewit Limitations](#bewit-limitations)
  - [Host Header Forgery](#host-header-forgery)
- [**Frequently Asked Questions**](#frequently-asked-questions)
- [**Implementations**](#implementations)
- [**Acknowledgements**](#acknowledgements)

# Introduction

**Hawk** is an HTTP authentication scheme providing mechanisms for making authenticated HTTP requests with
partial cryptographic verification of the request and response, covering the HTTP method, request URI, host,
and optionally the request payload.

Similar to the HTTP [Digest access authentication schemes](http://www.ietf.org/rfc/rfc2617.txt), **Hawk** uses a set of
client credentials which include an identifier (e.g. username) and key (e.g. password). Likewise, just as with the Digest scheme,
the key is never included in authenticated requests. Instead, it is used to calculate a request MAC value which is
included in its place.

However, **Hawk** has several differences from Digest. In particular, while both use a nonce to limit the possibility of
replay attacks, in **Hawk** the client generates the nonce and uses it in combination with a timestamp, leading to less
"chattiness" (interaction with the server).

Also unlike Digest, this scheme is not intended to protect the key itself (the password in Digest) because
the client and server must both have access to the key material in the clear.

The primary design goals of this scheme are to:
* simplify and improve HTTP authentication for services that are unwilling or unable to deploy TLS for all resources,
* secure credentials against leakage (e.g., when the client uses some form of dynamic configuration to determine where
  to send an authenticated request), and
* avoid the exposure of credentials sent to a malicious server over an unauthenticated secure channel due to client
  failure to validate the server's identity as part of its TLS handshake.

In addition, **Hawk** supports a method for granting third-parties temporary access to individual resources using
a query parameter called _bewit_ (in falconry, a leather strap used to attach a tracking device to the leg of a hawk).

The **Hawk** scheme requires the establishment of a shared symmetric key between the client and the server,
which is beyond the scope of this module. Typically, the shared credentials are established via an initial
TLS-protected phase or derived from some other shared confidential information available to both the client
and the server.


## Replay Protection

Without replay protection, an attacker can use a compromised (but otherwise valid and authenticated) request more
than once, gaining access to a protected resource. To mitigate this, clients include both a nonce and a timestamp when
making requests. This gives the server enough information to prevent replay attacks.

The nonce is generated by the client, and is a string unique across all requests with the same timestamp and
key identifier combination.

The timestamp enables the server to restrict the validity period of the credentials where requests occurring afterwards
are rejected. It also removes the need for the server to retain an unbounded number of nonce values for future checks.
By default, **Hawk** uses a time window of 1 minute to allow for time skew between the client and server (which in
practice translates to a maximum of 2 minutes as the skew can be positive or negative).

Using a timestamp requires the client's clock to be in sync with the server's clock. **Hawk** requires both the client
clock and the server clock to use NTP to ensure synchronization. However, given the limitations of some client types
(e.g. browsers) to deploy NTP, the server provides the client with its current time (in seconds precision) in response
to a bad timestamp.

There is no expectation that the client will adjust its system clock to match the server (in fact, this would be a
potential attack vector). Instead, the client only uses the server's time to calculate an offset used only
for communications with that particular server. The protocol rewards clients with synchronized clocks by reducing
the number of round trips required to authenticate the first request.


## Usage Example

Server code:

```js
const Http = require('http');
const Hawk = require('@hapi/hawk');


// Credentials lookup function

const credentialsFunc = function (id) {

    const credentials = {
        key: 'werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn',
        algorithm: 'sha256',
        user: 'Steve'
    };

    return credentials;
};

// Create HTTP server

const handler = async function (req, res) {

    let payload, status;
    
    // Authenticate incoming request
    
    try {
        const { credentials, artifacts } = await Hawk.server.authenticate(req, credentialsFunc);
        payload = `Hello ${credentials.user} ${artifacts.ext}`;
        status = 200;
    } catch (error) {
        payload = 'Shoosh!';
        status = 401;
    }

    // Prepare response

    const headers = { 'Content-Type': 'text/plain' };

    // Generate Server-Authorization response header

    const header = Hawk.server.header(credentials, artifacts, { payload, contentType: headers['Content-Type'] });
    headers['Server-Authorization'] = header;

    // Send the response back

    res.writeHead(status, headers);
    res.end(payload);
};

// Start server

Http.createServer(handler).listen(8000, 'example.com');
```

Client code:

```js
const Request = require('request');
const Hawk = require('@hapi/hawk');


// Client credentials

const credentials = {
    id: 'dh37fgj492je',
    key: 'werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn',
    algorithm: 'sha256'
}

// Request options

const requestOptions = {
    uri: 'http://example.com:8000/resource/1?b=1&a=2',
    method: 'GET',
    headers: {}
};

// Generate Authorization request header

const { header } = Hawk.client.header('http://example.com:8000/resource/1?b=1&a=2', 'GET', { credentials: credentials, ext: 'some-app-data' });
requestOptions.headers.Authorization = header;

// Send authenticated request

Request(requestOptions, function (error, response, body) {

    // Authenticate the server's response

    const isValid = Hawk.client.authenticate(response, credentials, header.artifacts, { payload: body });

    // Output results

    console.log(`${response.statusCode}: ${body}` + (isValid ? ' (valid)' : ' (invalid)'));
});
```

**Hawk** utilized the [**SNTP**](https://github.com/hueniverse/sntp) module for time sync management. By default, the local
machine time is used. To automatically retrieve and synchronize the clock within the application, use the SNTP 'start()' method.

```js
Hawk.sntp.start();
```


## Protocol Example

The client attempts to access a protected resource without authentication, sending the following HTTP request to
the resource server:

```
GET /resource/1?b=1&a=2 HTTP/1.1
Host: example.com:8000
```

The resource server returns an authentication challenge.

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Hawk
```

The client has previously obtained a set of **Hawk** credentials for accessing resources on the "`http://example.com/`"
server. The **Hawk** credentials issued to the client include the following attributes:

* Key identifier: `dh37fgj492je`
* Key: `werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn`
* Algorithm: `hmac sha256`
* Hash: `6R4rV5iE+NPoym+WwjeHzjAGXUtLNIxmo1vpMofpLAE=`

The client generates the authentication header by calculating a timestamp (e.g. the number of seconds since January 1,
1970 00:00:00 GMT), generating a nonce, and constructing the normalized request string (each value followed by a newline
character):

```
hawk.1.header
1353832234
j4h3g2
GET
/resource/1?b=1&a=2
example.com
8000

some-app-ext-data

```

The request MAC is calculated using HMAC with the specified hash algorithm "`sha256`" and the key over the normalized request string.
The result is base64-encoded to produce the request MAC:

```
6R4rV5iE+NPoym+WwjeHzjAGXUtLNIxmo1vpMofpLAE=
```

The client includes the **Hawk** key identifier, timestamp, nonce, application specific data, and request MAC with the request using
the HTTP `Authorization` request header field:

```
GET /resource/1?b=1&a=2 HTTP/1.1
Host: example.com:8000
Authorization: Hawk id="dh37fgj492je", ts="1353832234", nonce="j4h3g2", ext="some-app-ext-data", mac="6R4rV5iE+NPoym+WwjeHzjAGXUtLNIxmo1vpMofpLAE="
```

The server validates the request by calculating the request MAC again based on the request received and verifies the validity
and scope of the **Hawk** credentials. If valid, the server responds with the requested resource.


### Payload Validation

**Hawk** provides optional payload validation. When generating the authentication header, the client calculates a payload hash
using the specified hash algorithm. The hash is calculated over the concatenated value of (each followed by a newline character):
* `hawk.1.payload`
* the content-type in lowercase, without any parameters (e.g. `application/json`)
* the request payload prior to any content encoding (the exact representation requirements should be specified by the server for payloads other than simple single-part ascii to ensure interoperability)

For example:

* Payload: `Thank you for flying Hawk`
* Content Type: `text/plain`
* Algorithm: `sha256`
* Hash: `Yi9LfIIFRtBEPt74PVmbTF/xVAwPn7ub15ePICfgnuY=`

Results in the following input to the payload hash function (newline terminated values):

```
hawk.1.payload
text/plain
Thank you for flying Hawk

```

Which produces the following hash value:

```
Yi9LfIIFRtBEPt74PVmbTF/xVAwPn7ub15ePICfgnuY=
```

The client constructs the normalized request string (newline terminated values):

```
hawk.1.header
1353832234
j4h3g2
POST
/resource/1?b=1&a=2
example.com
8000
Yi9LfIIFRtBEPt74PVmbTF/xVAwPn7ub15ePICfgnuY=
some-app-ext-data

```

Then calculates the request MAC and includes the **Hawk** key identifier, timestamp, nonce, payload hash, application specific data,
and request MAC, with the request using the HTTP `Authorization` request header field:

```
POST /resource/1?b=1&a=2 HTTP/1.1
Host: example.com:8000
Authorization: Hawk id="dh37fgj492je", ts="1353832234", nonce="j4h3g2", hash="Yi9LfIIFRtBEPt74PVmbTF/xVAwPn7ub15ePICfgnuY=", ext="some-app-ext-data", mac="aSe1DERmZuRl3pI36/9BdZmnErTw3sNzOOAUlfeKjVw="
```

It is up to the server if and when it validates the payload for any given request, based solely on its security policy
and the nature of the data included.

If the payload is available at the time of authentication, the server uses the hash value provided by the client to construct
the normalized string and validates the MAC. If the MAC is valid, the server calculates the payload hash and compares the value
with the provided payload hash in the header. In many cases, checking the MAC first is faster than calculating the payload hash.

However, if the payload is not available at authentication time (e.g. too large to fit in memory, streamed elsewhere, or processed
at a different stage in the application), the server may choose to defer payload validation for later by retaining the hash value
provided by the client after validating the MAC.

It is important to note that MAC validation does not mean the hash value provided by the client is valid, only that the value
included in the header was not modified. Without calculating the payload hash on the server and comparing it to the value provided
by the client, the payload may be modified by an attacker.


## Response Payload Validation

**Hawk** provides partial response payload validation. The server includes the `Server-Authorization` response header which enables the
client to authenticate the response and ensure it is talking to the right server. **Hawk** defines the HTTP `Server-Authorization` header
as a response header using the exact same syntax as the `Authorization` request header field.

The header is constructed using the same process as the client's request header. The server uses the same credentials and other
artifacts provided by the client to constructs the normalized request string. The `ext` and `hash` values are replaced with
new values based on the server response. The rest as identical to those used by the client.

The result MAC digest is included with the optional `hash` and `ext` values:

```
Server-Authorization: Hawk mac="XIJRsMl/4oL+nn+vKoeVZPdCHXB4yJkNnBbTbHFZUYE=", hash="f9cDF/TDm7TkYRLnGwRMfeDzT6LixQVLvrIKhh0vgmM=", ext="response-specific"
```


## Browser Support and Considerations

A browser script is provided for including using a `<script>` tag in [lib/browser.js](/lib/browser.js). It's also a [component](http://component.io/hueniverse/hawk).

**Hawk** relies on the _Server-Authorization_ and _WWW-Authenticate_ headers in its response to communicate with the client.
Therefore, in case of CORS requests, it is important to consider sending _Access-Control-Expose-Headers_ with the value
_"WWW-Authenticate, Server-Authorization"_ on each response from your server. As explained in the
[specifications](http://www.w3.org/TR/cors/#access-control-expose-headers-response-header), it will indicate that these headers
can safely be accessed by the client (using getResponseHeader() on the XmlHttpRequest object). Otherwise you will be met with a
["simple response header"](http://www.w3.org/TR/cors/#simple-response-header) which excludes these fields and would prevent the
Hawk client from authenticating the requests.You can read more about the why and how in this
[article](http://www.html5rocks.com/en/tutorials/cors/#toc-adding-cors-support-to-the-server)


## hapi Plugin

**hawk** includes an authentication plugin for **hapi** which registers two authentication schemes.

### hawk Strategy

The scheme supports payload authentication. The scheme requires the following options:

- `getCredentialsFunc` - credential lookup function with the signature `[async] function(id)` where:
    - `id` - the Hawk credentials identifier.
    - _throws_ an internal error.
    - _returns_ `{ credentials }` object where:
        - `credentials` a credentials object passed back to the application in `request.auth.credentials`. Set to be `null` or `undefined` to
          indicate unknown credentials (which is not considered an error state).
- `hawk` - optional protocol options passed to `Hawk.server.authenticate()`.

```js
const Hapi = require('@hapi/hapi');
const Hawk = require('@hapi/hawk');

const credentials = {
    d74s3nz2873n: {
        key: 'werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn',
        algorithm: 'sha256'
    }
};

const getCredentialsFunc = function (id) {

    return credentials[id];
};

const start = async () => {

    const server = Hapi.server({ port: 4000 });

    await server.register(Hawk);

    server.auth.strategy('default', 'hawk', { getCredentialsFunc });
    server.auth.default('default');

    server.route({
        method: 'GET',
        path: '/',
        handler: function (request, h) {

            return 'welcome';
        }
    });

    await server.start();

    console.log('Server started listening on %s', server.info.uri);
};

start();

// Ensure process exits on unhandled rejection

process.on('unhandledRejection', (err) => {

    throw err;
});

```

### bewit Strategy

The scheme can only be used with 'GET' requests and requires the following options:

- `getCredentialsFunc` - credential lookup function with the signature `async function(id)` where:
    - `id` - the Hawk credentials identifier.
    - _throws_ an internal error.
    - _returns_ `{ credentials }` object where:
        - `credentials` a credentials object passed back to the application in `request.auth.credentials`. Set to be `null` or `undefined` to
      indicate unknown credentials (which is not considered an error state).
- `hawk` - optional protocol options passed to `Hawk.server.authenticateBewit()`.

```js
const Hapi = require('@hapi/hapi');
const Hawk = require('@hapi/hawk');

const credentials = {
    d74s3nz2873n: {
        key: 'werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn',
        algorithm: 'sha256'
    }
};

const getCredentialsFunc = function (id) {

    return credentials[id];
};

const start = async () => {

    const server = Hapi.server({ port: 4000 });

    await server.register(Hawk);

    server.auth.strategy('default', 'bewit', { getCredentialsFunc });
    server.auth.default('default');

    server.route({
        method: 'GET',
        path: '/',
        handler: function (request, h) {

            return 'welcome';
        }
    });

    await server.start();

    console.log('Server started listening on %s', server.info.uri);
};

start();

// Ensure process exits on unhandled rejection

process.on('unhandledRejection', (err) => {

    throw err;
});
```

To send an authenticated Bewit request, the URI must contain the `'bewit'` query parameter which can be generated using the Hawk module:

```js
const Hawk = require('@hapi/hawk');

const credentials = {
    id: 'd74s3nz2873n',
    key: 'werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn',
    algorithm: 'sha256'
};

let uri = 'http://example.com:8080/endpoint';
const bewit = Hawk.client.getBewit(uri, { credentials: credentials, ttlSec: 60 });
uri += '?bewit=' + bewit;
```


# Single URI Authorization

There are cases in which limited and short-term access to a protected resource is granted to a third party which does not
have access to the shared credentials. For example, displaying a protected image on a web page accessed by anyone. **Hawk**
provides limited support for such URIs in the form of a _bewit_ - a URI query parameter appended to the request URI which contains
the necessary credentials to authenticate the request.

Because of the significant security risks involved in issuing such access, bewit usage is purposely limited only to GET requests
and for a finite period of time. Both the client and server can issue bewit credentials, however, the server should not use the same
credentials as the client to maintain clear traceability as to who issued which credentials.

In order to simplify implementation, bewit credentials do not support single-use policy and can be replayed multiple times within
the granted access timeframe.


## Bewit Usage Example

Server code:

```js
const Http = require('http');
const Hawk = require('@hapi/hawk');


// Credentials lookup function

const credentialsFunc = function (id) {

    const credentials = {
        key: 'werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn',
        algorithm: 'sha256'
    };

    return credentials;
};

// Create HTTP server

const handler = function (req, res) {

    try {
        const { credentials, attributes } = await Hawk.uri.authenticate(req, credentialsFunc);
        res.writeHead(200, { 'Content-Type': 'text/plain' });
        res.end('Access granted');
    }
    catch (err) {
        res.writeHead(401, { 'Content-Type': 'text/plain' });
        res.end('Shoosh!');
    }
};

Http.createServer(handler).listen(8000, 'example.com');
```

Bewit code generation:

```js
const Hawk = require('@hapi/hawk');


// Client credentials

const credentials = {
    id: 'dh37fgj492je',
    key: 'werxhqb98rpaxn39848xrunpaw3489ruxnpa98w4rxn',
    algorithm: 'sha256'
}

// Generate bewit

const duration = 60 * 5;      // 5 Minutes
const bewit = Hawk.uri.getBewit('http://example.com:8000/resource/1?b=1&a=2', { credentials: credentials, ttlSec: duration, ext: 'some-app-data' });
const uri = 'http://example.com:8000/resource/1?b=1&a=2' + '&bewit=' + bewit;

// Output URI

console.log('URI: ' + uri);
```


# Security Considerations

The greatest sources of security risks are usually found not in **Hawk** but in the policies and procedures surrounding its use.
Implementers are strongly encouraged to assess how this module addresses their security requirements. This section includes
an incomplete list of security considerations that must be reviewed and understood before deploying **Hawk** on the server.
Many of the protections provided in **Hawk** depends on whether and how they are used.

### MAC Keys Transmission

**Hawk** does not provide any mechanism for obtaining or transmitting the set of shared credentials required. Any mechanism used
to obtain **Hawk** credentials must ensure that these transmissions are protected using transport-layer mechanisms such as TLS.

### Confidentiality of Requests

While **Hawk** provides a mechanism for verifying the integrity of HTTP requests, it provides no guarantee of request
confidentiality. Unless other precautions are taken, eavesdroppers will have full access to the request content. Servers should
carefully consider the types of data likely to be sent as part of such requests, and employ transport-layer security mechanisms
to protect sensitive resources.

### Spoofing by Counterfeit Servers

**Hawk** provides limited verification of the server authenticity. When receiving a response back from the server, the server
may choose to include a response `Server-Authorization` header which the client can use to verify the response. However, it is up to
the server to determine when such measure is included, to up to the client to enforce that policy.

A hostile party could take advantage of this by intercepting the client's requests and returning misleading or otherwise
incorrect responses. Service providers should consider such attacks when developing services using this protocol, and should
require transport-layer security for any requests where the authenticity of the resource server or of server responses is an issue.

### Plaintext Storage of Credentials

The **Hawk** key functions the same way passwords do in traditional authentication systems. In order to compute the request MAC,
the server must have access to the key in plaintext form. This is in contrast, for example, to modern operating systems, which
store only a one-way hash of user credentials.

If an attacker were to gain access to these keys - or worse, to the server's database of all such keys - he or she would be able
to perform any action on behalf of any resource owner. Accordingly, it is critical that servers protect these keys from unauthorized
access.

### Entropy of Keys

Unless a transport-layer security protocol is used, eavesdroppers will have full access to authenticated requests and request
MAC values, and will thus be able to mount offline brute-force attacks to recover the key used. Servers should be careful to
assign keys which are long enough, and random enough, to resist such attacks for at least the length of time that the **Hawk**
credentials are valid.

For example, if the credentials are valid for two weeks, servers should ensure that it is not possible to mount a brute force
attack that recovers the key in less than two weeks. Of course, servers are urged to err on the side of caution, and use the
longest key reasonable.

It is equally important that the pseudo-random number generator (PRNG) used to generate these keys be of sufficiently high
quality. Many PRNG implementations generate number sequences that may appear to be random, but which nevertheless exhibit
patterns or other weaknesses which make cryptanalysis or brute force attacks easier. Implementers should be careful to use
cryptographically secure PRNGs to avoid these problems.

### Coverage Limitations

The request MAC only covers the HTTP `Host` header and optionally the `Content-Type` header. It does not cover any other headers
which can often affect how the request body is interpreted by the server. If the server behavior is influenced by the presence
or value of such headers, an attacker can manipulate the request headers without being detected. Implementers should use the
`ext` feature to pass application-specific information via the `Authorization` header which is protected by the request MAC.

The response authentication, when performed, only covers the response payload, content-type, and the request information
provided by the client in its request (method, resource, timestamp, nonce, etc.). It does not cover the HTTP status code or
any other response header field (e.g. `Location`) which can affect the client's behaviour.

### Future Time Manipulation

The protocol relies on a clock sync between the client and server. To accomplish this, the server informs the client of its
current time when an invalid timestamp is received.

If an attacker is able to manipulate this information and cause the client to use an incorrect time, it would be able to cause
the client to generate authenticated requests using time in the future. Such requests will fail when sent by the client, and will
not likely leave a trace on the server (given the common implementation of nonce, if at all enforced). The attacker will then
be able to replay the request at the correct time without detection.

The client must only use the time information provided by the server if:
* it was delivered over a TLS connection and the server identity has been verified, or
* the `tsm` MAC digest calculated using the same client credentials over the timestamp has been verified.

### Client Clock Poisoning

When receiving a request with a bad timestamp, the server provides the client with its current time. The client must never use
the time received from the server to adjust its own clock, and must only use it to calculate an offset for communicating with
that particular server.

### Bewit Limitations

Special care must be taken when issuing bewit credentials to third parties. Bewit credentials are valid until expiration and cannot
be revoked or limited without using other means. Whatever resource they grant access to will be completely exposed to anyone with
access to the bewit credentials which act as bearer credentials for that particular resource. While bewit usage is limited to GET
requests only and therefore cannot be used to perform transactions or change server state, it can still be used to expose private
and sensitive information.

### Host Header Forgery

Hawk validates the incoming request MAC against the incoming HTTP Host header. However, unless the optional `host` and `port`
options are used with `server.authenticate()`, a malicious client can mint new host names pointing to the server's IP address and
use that to craft an attack by sending a valid request that's meant for another hostname than the one used by the server. Server
implementors must manually verify that the host header received matches their expectation (or use the options mentioned above).

# Frequently Asked Questions

### Where is the protocol specification?

If you are looking for some prose explaining how all this works, **this is it**. **Hawk** is being developed as an open source
project instead of a standard. In other words, the [code](https://github.com/hueniverse/hawk/tree/master/lib) is the specification. Not sure about
something? Open an issue!

### Is it done?

Yes. The protocol is finished. This module might change with enhancements and bug fixes but the protocol itself is locked.

### Where can I find **Hawk** implementations in other languages?

**Hawk**'s only reference implementation is provided in JavaScript as a node.js module. However, it has been ported to other languages.
The full list is maintained [here](https://github.com/hueniverse/hawk/issues?labels=port&state=closed). Please add an issue if you are
working on another port. A cross-platform test-suite is in the works.

### Why isn't the algorithm part of the challenge or dynamically negotiated?

The algorithm used is closely related to the key issued as different algorithms require different key sizes (and other
requirements). While some keys can be used for multiple algorithm, the protocol is designed to closely bind the key and algorithm
together as part of the issued credentials.

### Why is Host and Content-Type the only headers covered by the request MAC?

It is really hard to include other headers. Headers can be changed by proxies and other intermediaries and there is no
well-established way to normalize them. Many platforms change the case of header field names and values. The only
straight-forward solution is to include the headers in some blob (say, base64 encoded JSON) and include that with the request,
an approach taken by JWT and other such formats. However, that design violates the HTTP header boundaries, repeats information,
and introduces other security issues because firewalls will not be aware of these "hidden" headers. In addition, any information
repeated must be compared to the duplicated information in the header and therefore only moves the problem elsewhere.

### Why not just use HTTP Digest?

Digest requires pre-negotiation to establish a nonce. This means you can't just make a request - you must first send
a protocol handshake to the server. This pattern has become unacceptable for most web services, especially mobile
where extra round-trip are costly.

### Why bother with all this nonce and timestamp business?

**Hawk** is an attempt to find a reasonable, practical compromise between security and usability. OAuth 1.0 got timestamp
and nonces halfway right but failed when it came to scalability and consistent developer experience. **Hawk** addresses
it by requiring the client to sync its clock, but provides it with tools to accomplish it.

In general, replay protection is a matter of application-specific threat model. It is less of an issue on a TLS-protected
system where the clients are implemented using best practices and are under the control of the server. Instead of dropping
replay protection, **Hawk** offers a required time window and an optional nonce verification. Together, it provides developers
with the ability to decide how to enforce their security policy without impacting the client's implementation.

### What are `app` and `dlg` in the authorization header and normalized mac string?

The original motivation for **Hawk** was to replace the OAuth 1.0 use cases. This included both a simple client-server mode which
this module is specifically designed for, and a delegated access mode which is being developed separately in
[Oz](https://github.com/hueniverse/oz). In addition to the **Hawk** use cases, Oz requires another attribute: the application id `app`.
This provides binding between the credentials and the application in a way that prevents an attacker from tricking an application
to use credentials issued to someone else. It also has an optional 'delegated-by' attribute `dlg` which is the application id of the
application the credentials were directly issued to. The goal of these two additions is to allow Oz to utilize **Hawk** directly,
but with the additional security of delegated credentials.

### What is the purpose of the static strings used in each normalized MAC input?

When calculating a hash or MAC, a static prefix (tag) is added. The prefix is used to prevent MAC values from being
used or reused for a purpose other than what they were created for (i.e. prevents switching MAC values between a request,
response, and a bewit use cases). It also protects against exploits created after a potential change in how the protocol
creates the normalized string. For example, if a future version would switch the order of nonce and timestamp, it
can create an exploit opportunity for cases where the nonce is similar in format to a timestamp.

### Does **Hawk** have anything to do with OAuth?

Short answer: no.

**Hawk** was originally proposed as the OAuth MAC Token specification. However, the OAuth working group in its consistent
incompetence failed to produce a final, usable solution to address one of the most popular use cases of OAuth 1.0 - using it
to authenticate simple client-server transactions (i.e. two-legged). As you can guess, the OAuth working group is still hard
at work to produce more garbage.

**Hawk** provides a simple HTTP authentication scheme for making client-server requests. It does not address the OAuth use case
of delegating access to a third party. If you are looking for an OAuth alternative, check out [Oz](https://github.com/hueniverse/oz).

# Implementations

- [Logibit Hawk in F#/.Net](https://github.com/logibit/logibit.hawk/)
- [Tent Hawk in Ruby](https://github.com/tent/hawk-ruby)
- [Wealdtech in Java](https://github.com/wealdtech/hawk)
- [Kumar's Mohawk in Python](https://github.com/kumar303/mohawk/)
- [Hiyosi in Go](https://github.com/hiyosi/hawk)

# Acknowledgements

**Hawk** is a derivative work of the [HTTP MAC Authentication Scheme](http://tools.ietf.org/html/draft-hammer-oauth-v2-mac-token-05) proposal
co-authored by Ben Adida, Adam Barth, and Eran Hammer, which in turn was based on the OAuth 1.0 community specification.

Special thanks to Ben Laurie for his always insightful feedback and advice.

The **Hawk** logo was created by [Chris Carrasco](http://chriscarrasco.com).
