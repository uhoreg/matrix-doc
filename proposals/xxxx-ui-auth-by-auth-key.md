# MSCxxxx: User-interactive authentication using an authentication key

Some Matrix security-sensitive endpoints require that the user re-authenticate
in order to ensure that the person performing the request is actually the user
who is logged in.  This is known as [User-Unteractive
Authentication](https://spec.matrix.org/unstable/client-server-api/#user-interactive-authentication-api)
(UI ACTH), and often involves the user re-entering their password.

However, it may be inconvenient for the user to enter their password,
especially on mobile devices, and some devices may have other mechanisms for
verifying the user's identity, such as fingerprint readers.

This proposal introduces a way for a client to perform UI Auth after it has
itself checked the user's identity.  This is based on public-key cryptography:
the server is given a public key and the client retains the corresponding
private key.  When the server requires UI auth, it will include a challenge in
its response.  The client will complete the challenge, and submit its response
in its UI Auth parameter.

## Proposal

A new `authentication_keys` parameter is added to the `POST /login` endpoint,
which is a map from key ID (prefixed with the key algorithm followed by a
colon, as is currently done with device keys) to a public key.  The format of
the public key is defined by the algorithm.  It may only contain one key per
key algorithm.

New endpoints are added:

- `POST /authentication_keys` is used to set the user's authentication keys.
  Requires UI Auth.  The request body has one required parameter,
  `authentication_keys`, which has the same format as the same parameter for
  `POST /login`.  Clients should only have one key per algorithm, so if a key
  is uploaded for an algorithm for which the user already has a key, the server
  should evict the old key.
- `DELETE /authentication_keys/{algorithm}/{keyId}`, where `keyId` is the
  unprefixed key ID, deletes the specified key.  Does not require UI Auth.

A new authentication type is defined for UI Auth: `m.login.authentication_key`.

When UI auth is needed and the client can authenticate using an authentication
key as its next step in a flow, the server will select one of the public keys
previously uploaded by the client (in general, it should select the key with
the strongest algorithm that it supports), generate a challenge according to
the algorithm, and include the algorithm name (in the `algorithm` property),
key ID (`key_id` property), and challenge (`challenge` property) in its
`params` response property.  The format of the `challenge` property is defined
by the key algorithm.  The client will complete the challenge and use submit
its response to complete the UI auth stage in the `response` property of the
`auth` property.


### Algorithms: curve25519-hkdf-sha256

We initially only define one algorithm: `curve25519-hkdf-sha256`, which uses
curve25519 public and private keys.  The public key format is the same as for
the `curve25519` key algorithm used for device keys.  The key ID is the public
key.

To construct the challenge, the server generates an ephemeral curve25519
keypair and sends the public part as the challenge, encoded in the same way as
the `curve25519` key algorithm used for device keys.

To complete the challenge, the client performs an ECDH using its private key
and the ephemeral public key to generate a shared secret.  It then uses the
shared secret as the input key material HKDF-SHA256, using the empty string as
salt, and the concatenation of: the client's authentication public key, a
"|", the ephemeral public key, another "|", and the session ID as the info
parameter.  256 bits (32 bytes) are generated, and encoded as unpadded base-64,
and the result in used as the response.

The server perform the same calculation (except using the client's public key
and the ephemeral private key for ECDH) and ensures that the result matches the
value submitted by the client.

### Example

Alice wants to log out some of her other devices, so she calls `POST
/delete_devices`, which requires UI Auth.  The server responds with:

```jsonc
{
  "flows": [
    {
      "stages": ["m.login.authentication_key"]
    },
    // other possible flows...
  ],
  "params": {
    "m.login.authentication_key": {
      "algorithm": "curve25519",
      "key_id": "a+curve25519+public+key",
      "challenge": "another+curve25519+key"
    }
  },
  "session": "a_session_id"
}
```

The client computes the ECDH shared secret, performs the HKDF with
"a+curve25519+public+key|another+curve25519+key|a_session_id" as the info
parameter, yielding "output_key", and re-performs
the `POST /delete_devices` request with:

```jsonc
{
  "auth": {
    "type": "m.login.authentication_key",
    "session": "a_session_id",
    "response": "output_key"
  },
  // other request parameters
}
```

## Potential issues

See security considerations

## Alternatives

None?

## Security considerations

Since this proposal allows clients to authenticate without interaction from the
user, this assumes that clients are trusted: clients can surreptitiously call
protected endpoints without the user noticing.

This proposal relies on the secrecy of the private keys; clients that implement
this must ensure that the private keys cannot be obtained by anything else,
such as other programs running on the same computer.  Clients should make use
of the device's secure enclave, wherever one is available, to secure the
private key.

Clients must ensure the user's identity before authenticating using this
method.  This can be done, for example, by using biometrics.  Less secure
methods could also be used, such as using a PIN, provided that additional steps
are taken to increase security, such as adding a timeout between PIN attempts,
and limiting the maximum number of attempts.

## Unstable prefix

Until this proposal is accepted, the following unstable identifiers should be
used in place of the stable identifiers:

- `org.matrix.mscxxxx.authentication_keys` rather than `authentication_keys` as
  the `POST /login` parameter
- `POST /unstable/org.matrix.mscxxxx/authentication_keys` rather than `POST
  /authentication_keys`
- `DELETE /unstable/org.matrix.mscxxxx/authentication_keys/{algorithm}/{keyId}`
  rather than `DELETE /authentication_keys/{algorithm}/{keyId}`
- `org.matrix.mscxxxx.login.authentication_key` rather than
  `m.login.authentication_key` as the authentication type

There is no need for an unstable version of the algorithm name since the
algorithm name is only ever used in the context of one of the other identifiers.

## Dependencies

None
