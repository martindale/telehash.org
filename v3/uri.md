# URI handling for Out-Of-Band / Invite requests

This defines a URI format that any endpoint can use to encode temporary or permanent connectivity information for sending in an out-of-band medium.  Once a URI is resolved to an endpoint, that information (keys and paths) should be stored and used in place of the URI, a successful resolution only needs to be performed once.

A URI takes the form of: `protocol://user@canonical/session?csid=base32#token`

* `protocol` - should be defined or customized by the app so that it can install it's own handlers, defaults to `mesh` 
* `user@` - optional, only used to indicate a human-recognizable name that the URI is associated to within the app, see [name formatting](#friendly) for rules
* `canonical` - required, valid host name format (defaults to ip), may include optional `:port`
* `/session` - optional, opaque string of up to 64 [unreserved characters](https://tools.ietf.org/html/rfc3986#section-2.3) used by the recipient to match to a particular session
* `?csid=base32` - optional, when a canonical host name is used that has no mechanism for retrieving its public keys, they may be included as query string args
* `#token` - optional, opaque string of up to 1024 [unreserved characters](https://tools.ietf.org/html/rfc3986#section-2.3) used by the recipient to track a specific request

If supported, routers should always return a URI without a `#token` for the endpoint as a [handshake message](e3x/handshake.md), endpoints can then add a `#token` before sharing the full URI.  Endpoints may also generate their own URI to themsevles if they have an accessible hostname and port available.

Processing:

1. validate canonical
2. do we know canonical already
3. can we resolve the canonical
4. issue a `peer` request to the canonical including the `"uri":"..."`, which is also passed in the `connect` from the router
5. process any response handshake as the permanent resolution for this URI

## Token Reserved Characters

Upon encountering any reserved character (not part of the set of `ALPHA / DIGIT / "-" / "." / "_" / "~"`) when processing the token string, the remainder of the token must be ignored and not considered part of the token or URI.  It is not an error to have additional data at the end, that data must be processed by the application and is ignored by all URI handling.

## Canonical Host Keys

In order to send a handshake to the canonical host name given in a URI, its public keys must be known.  Each Cipher Set may be optionally included in the query string, and/or the keys may be shared via other standard lookup mechanisms to make the URIs shorter and easier.

### DNS

SRV records always resolve to a hashname-prefixed host, with TXT records returning all of the keys.

* `_mesh._udp.example.com. 86400 IN SRV 0 5 42424 uvabrvfqacyvgcu8kbrrmk9apjbvgvn2wjechqr3vf9c1zm3hv7g.example.com.` (also optionally support `_tcp` and `_http`, etc)
* `uvabrvfqacyvgcu8kbrrmk9apjbvgvn2wjechqr3vf9c1zm3hv7g IN A 1.2.3.4`
* `uvabrvfqacyvgcu8kbrrmk9apjbvgvn2wjechqr3vf9c1zm3hv7g IN TXT "1a=base32"`
* `uvabrvfqacyvgcu8kbrrmk9apjbvgvn2wjechqr3vf9c1zm3hv7g IN TXT "2a=base32"`
* `uvabrvfqacyvgcu8kbrrmk9apjbvgvn2wjechqr3vf9c1zm3hv7g IN TXT "2a2=base32"`
* `uvabrvfqacyvgcu8kbrrmk9apjbvgvn2wjechqr3vf9c1zm3hv7g IN TXT "3a=base32"`

If a key's base32 encoding is larger than 250 characters, it is broken into multiple TXT records with the CSID being numerically increased so that it can be easily reassembled.

No other DNS record type is supported, only SRV records resulting in one or more A and TXT records.

### HTTP well-known

```
GET http://example.com/.well-known/mesh.json
{
  "keys":{...},
  "paths":{...}
}

### dotPublic

Routers may also use a [dotPublic](https://github.com/quartzjer/dotPublic) hostname instead of a registered domain name, which will internally resolve using public keys.

### Local Network

A hostname that is a local IPv4 or IPv6 network address may be sent a discovery request directly on that network.

## Connecting

With keys available, if the resolution of the canonical does not also result in `path` information and only an IP and Port are known, handshakes should be sent to that address in every transport supported that can use an IP and Port, including UDP, TCP, TLS, and HTTP(S).

### Handshake

Any handshake that is generated as a direct result of a URI that has a `/session` should always include the `"session":"..."` in the inner packet of the handshake so that the recipient can use it for additional validation before deciding to respond.

### Peer/Connect Request

When a new [peer](channels/peer.md) request is made to a router resulting from the processing a URI, it should include the `"session":"..."` and/or `"token":"..."` if they were specified in the original URI.  If a subsequent [connect](channels/connect.md) is generated by the router, only the `"token":"..."` should be included in the connect request to the final endpoint so that it can validate the token to help decide if it should respond.

<a name="friendly" />
## Friendly `user@` Name

The optional `user` part of the URI is only used for visual human-recognizable purposes, it should never be sent or used except as a user-experience display and any subsequent request-matching is always performed with only the `session` and `token` values.

The user string must begin with 1 to 8 lower-case ascii alpha characters only (`a-z`) to serve as a simple visual indicator in a raw string encoded URI.  It may be followed by a single period (`.`) and 2 to 52 valid base32 characters (same base32 encoding as [hashnames](hashname/)), which when decoded contain a sequence of 1 to 32 UTF-8 characters to be shown or used in place of the preceeding ascii alpha characters in displays that support Unicode and for when special characters or punctuation are required.

Applications should never represent this friendly name as the only method of user trust. When displaying a `user` value the app should always be able to show full details about the URI and the context in which it was received to minimize spoofing.