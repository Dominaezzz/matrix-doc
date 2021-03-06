# Grammars for identifiers in the Matrix protocol

## Background

Matrix uses client- or server-generated identifiers in a number of
places. Historically the grammars for these have been underspecified, which
leads to confusion about what is or is not a valid identifier with the
possibility of incompatability between implementations.

This proposal presents tightly-specified grammars for a number of
identifiers.

## Common Identifiers

[Spec](https://matrix.org/docs/spec/appendices.html#common-identifier-format)

Proposal:

> `localpart` may not include `:`. When parsing a Common Identifier, it should
>  be split at the *leftmost* `:`.

Rationale: server names may contain multiple `:`s (think IPv6 literals), so the
first colon is the only sane place to split them. This is a Known Thing, but I
don't think we spell it out anywhere in the spec.

## User IDs

User IDs are
[well-specified](https://matrix.org/docs/spec/appendices.html#user-identifiers),
however we should consider dropping `/` from the list of allowed characters,
because HTTP proxies might rewrite
`/_matrix/client/r0/profile/@foo%25bar:matrix.org/displayname` to
`/_matrix/client/r0/profile/@foo/bar:matrix.org/displayname`, messing things
up.

History: `/` was introduced with the intention of acting as a hierarchical
namespacing character, particularly with consideration to the gitter protocol
which uses it as a hierarchical separator. However, this was not as effective
as hoped because `@foo/bar:example.com` looks like the ID is partitioned into
`@foo` and `bar:example.com`.

Proposal:

> Remove `/` from the list of allowed characters in User IDs.

`/` will of course be maintained under the grammar of "historical user
IDs". Sorting out that mess is a longer-term project.

## Room IDs and Event IDs

[Issue](https://github.com/matrix-org/matrix-doc/issues/667)
[Spec](https://matrix.org/docs/spec/appendices.html#room-ids-and-event-ids)

These currently have similar formats, though it is likely that event ids will
be replaced with something else due to
[#1127](https://github.com/matrix-org/matrix-doc/issues/1127).

Currently they are both specified as ``?opaque_id:domain``, without clues as to
what the opaque_id should be.

Synapse uses: `[A-Za-z]{18}`.
[Dendrite](https://github.com/matrix-org/dendrite/blob/b71d922/src/github.com/matrix-org/dendrite/clientapi/routing/createroom.go#L125)
uses (I think) `[A-Za-z0-9]{16}` via
[json.go](https://github.com/matrix-org/util/blob/master/json.go#L185). However,
some server implementations/forks are known to generate event IDs (and possibly
room IDs) using a wide alphabet, which means that there exist rooms that
include unusual event IDs.

Proposal:

> The opaque_id part must not be empty, and must consist entirely of the
> characters `[0-9a-zA-Z.=_-]`.
>
> The total length (including sigil and domain) must not exceed 255 characters.
>
> This is only enforced for v2 rooms - servers and clients wishing to support
> v1 rooms should be more tolerant.


## Key IDs (for federation, e2e, and identity servers)

These are always of the form `<algorithm>:<tok>`. 

Valid algorithms are defined at
https://matrix.org/docs/spec/client_server/unstable.html#key-algorithms, though
we should define the alphabet for future algorithms.

Proposal:

> Future algorithm identifiers will be assigned from the alphabet `[a-z0-9_.]`
> and will be at most 31 characters in length.

For federation keys,
[Synapse](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/config/key.py#L159)
generates key ids as `ed25519:a_[A-Za-z]{4}`, though an HS admin can configure
them manually to be anything without whitespace.

Key IDs end up in an Authorization header which looks like `X-Matrix
origin=origin.example.com,key="keyId",sig="ABCDEF..."`. The Synapse
implementation splits on `,` and `=` without regard to quoting so this
currently precludes the use of `,` or `=` in a key ID.

For e2e, device keys have a `tok` corresponding to the device id, whilst
one-time keys are generated by libolm, which uses a base64-encoded 32-bit int, ie
`[A-Za-z0-9+/]{6}`.

A key ID needs to be unique over the lifetime of the server (for federation) or
the device (for e2e). However, they are used fairly widely, so making them long
is unattractive as they could significantly increase the amount of data being
transmitted. Let's limit the 'tok' part of the key to 31 characters too.

Proposal:

> Key IDs use the following BNF grammar:
>
> ```
> key_id         = algorithm ":" tok
>
> algorithm      = 1*31 alg_chars
>
> tok            = 1*31 tok_chars
>
> alg_chars      = %x61-7a / %30-39 / "_" / "."
>                     ; a-z 0-9 _ .
>
> tok_chars      = ALPHA / DIGIT / "." / "=" / "_" / "-"
>                     ; A-Z a-z 0-9 . = _ -
> ```
>

Note that enforcing this grammar will mean:

* Making sure that synapse handles "=" characters in key IDs (easy).

* Making libolm not put + and / characters in key IDs (easy enough, but there
  will be a bunch of malformed unique keys out there in the wild. Possibly they
  would just get thrown away. Servers may need to continue to tolerate `+` and
  `/` in e2e keys for a while.)

## Opaque IDs

[Issue](https://github.com/matrix-org/matrix-doc/issues/666)

This is a class of identifier types where nobody is really meant to parse any
part of the ID - they are just unique identifiers (with varying scopes of
uniqueness). See below for discussion on what is currently in use.

I propose to specify the almost the same grammar for all of these, for
simplicity and consistency.

Proposal:

> Opaque IDs must be strings consisting entirely of the characters
> `[0-9a-zA-Z.=_-]`. Their length must not exceed 255 characters and they must
> not be empty.

For almost all of the current implementations I have looked at (listed below),
the grammar above is a superset of the generated identifiers, and a subset of
the understood identifiers. There should therefore be no
backwards-compatibility problems with its introduction.

The exception is transaction IDs generated by some clients. I think that we'll
just have to fix those clients and accept that old versions may not work with
future servers.

### Call IDs

[Spec](https://matrix.org/docs/spec/client_server/unstable.html#m-call-invite)

These are only used within the body of `m.call.*` events, as far as I am
aware. They should be unique within the lifetime of a room. (Some
implementations currently treat them as globally unique, but that is considered
an implementation bug.)

[matrix-js-sdk](https://github.com/matrix-org/matrix-js-sdk/blob/4d310cd4618db4e98a8e6b5eb812480102ee4dee/src/webrtc/call.js#L72) uses `c[0-9.]{32}`.
[matrix-android-sdk](https://github.com/matrix-org/matrix-android-sdk/blob/5c6f785e53632e7b6fb3f3859a90c3d85b040e7f/matrix-sdk/src/main/java/org/matrix/androidsdk/call/MXWebRtcCall.java#L221) uses `c[0-9]{13}`.

Additional proposal:

> Call IDs should be long enough to make clashes unlikely.

### Media IDs

[Spec](https://matrix.org/docs/spec/client_server/r0.3.0.html#id67)

These are generated by the server on upload, and then embedded in `mxc://` URIs
and used in the C-S API and the S-S API.

They must be URI-safe to be sensibly embedded in `mxc://` URIs.

[Synapse](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/rest/media/v1/media_repository.py#L153)
uses `[A-Za-z]{24}`, though it also uses `[0-9A-Za-z_-]{27}` for
[URL
previews](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/rest/media/v1/preview_url_resource.py#L285).

[matrix-media-repo](https://github.com/turt2live/matrix-media-repo/blob/539f25ee75ba6cdbb0410314b29978f4b8b1d7fe/src/github.com/turt2live/matrix-media-repo/controllers/upload_controller/upload_controller.go#L50)
uses `[A-Za-z0-9]{32}`, via [random.go](https://github.com/turt2live/matrix-media-repo/blob/539f25ee75ba6cdbb0410314b29978f4b8b1d7fe/src/github.com/turt2live/matrix-media-repo/util/random.go#L18-L27).

### Filter IDs

[Spec](https://matrix.org/docs/spec/client_server/unstable.html#post-matrix-client-r0-user-userid-filter)

These are generated by the server and then used in the CS API. They are only
required to be unique for a given user. `{` is already forbidden by the spec.

[Synapse](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/storage/filtering.py#L70-L73)
uses a stringified int.

### Auth Session IDs

[Spec](https://matrix.org/docs/spec/client_server/r0.3.0.html#user-interactive-authentication-api)

These are generated by the server during auth, and then used in the CS
API. However, they need to be unique for a given server.

[Synapse](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/handlers/auth.py#L494) uses `[A-Za-z]{24}`.

### Transaction IDs (for federation)

[Spec](https://matrix.org/docs/spec/server_server/unstable.html#put-matrix-federation-v1-send-txnid)

Generated by sending server. Needs to be unique for a given pair of servers.

[Synapse](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/federation/transaction_queue.py#L593) uses a stringified int and accepts pretty much anything.

### Transaction IDs (for C-S API)

[Spec](https://matrix.org/docs/spec/client_server/unstable.html#put-matrix-client-r0-rooms-roomid-send-eventtype-txnid)

These are generated by the client. They only need to be unique within the
context of a single access_token/device.

Synapse doesn't appear to do any sanity-checking here currently.

[matrix-js-sdk](https://github.com/matrix-org/matrix-js-sdk/blob/c6b500bc09994ab5924ef8aab9bd10fc7ded5dae/src/base-apis.js#L123)
uses `m[0-9]{13}.[0-9]{1,}`.
[matrix-android-sdk](https://github.com/matrix-org/matrix-android-sdk/blob/088414fb187cae341690c3a01493b87d97f0169f/matrix-sdk/src/main/java/org/matrix/androidsdk/rest/model/Event.java#L503)
uses a room ID plus a timestamp, hence kinda could be anything, but certainly
will include a `!`.

### Device IDs

[Spec](https://matrix.org/docs/spec/client_server/unstable.html#relationship-between-access-tokens-and-devices)

These are normally generated by the server on login. It's possible for clients
to present their own device_ids, but we're not aware of this feature being
widely used.

They are used between users and across federation for E2E and to-device
messages. They need to be unique for a particular user. They also appear in key
IDs and must therefore be a subset of that grammar.

[Synapse](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/handlers/device.py#L89)
generates device IDs with `[A-Z]{10}`. It appears to do little sanity-checking
of client-generated device IDs currently.

Additional proposal:

> Device IDs must not exceed 31 characters in length.

### Message IDs

These are used in the server-server API for 
[Send-to-device messaging](https://matrix.org/docs/spec/server_server/unstable.html#send-to-device-messaging).

Synapse uses `[A-Za-z]{16}`, and accepts anything that fits in a postgres TEXT
field. Ref: [devicemessage.py](https://github.com/matrix-org/synapse/blob/74854a97191191b08101821753c2672efc2a65fd/synapse/handlers/devicemessage.py#L102).


## Room Aliases

These are a complex topic and are discussed in [MSC
1608](https://github.com/matrix-org/matrix-doc/issues/1608).
