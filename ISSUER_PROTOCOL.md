# TrustTokenV1 Issuer Protocol

This document documents the cryptographic protocol for the "TrustTokenV1" experimental version of Trust Token. An issuer needs to support maintaining a set of keys and a key commitment endpoint, as well as implementing the Issue and Redeem cryptographic functions to sign and validate Trust Tokens. Experimental versions of Trust Token are not intended to be backwards-compatible with each other and will undergo rapid design/implementation changes during the experiment timeframe.

This document uses TLS presentation language (https://tools.ietf.org/html/rfc8446#section-3) for structures and serialization.

## Public Issuer Interfaces

This section describes the public issuer interfaces that an issuer will need to support.

### Issuer Key Commitments

A Trust Token issuer should have an endpoint at a publicly accessible secure URL (HTTPS) that serves the current key commitments used in the Trust Token protocol. Requests to this endpoint should result in a JSON response of the following format:


```
Key commitment result
{
  "protocol_version": <protocol version, TrustTokenV1 for this>,
  "id": <key commitment identifier, as a monotonically increasing integer>
  "batchsize": <batch size>,
  "srrkey": <base-64 encoded SRRVerificationKey, in 32-byte RFC8032 encoding>,
  <keyID>: { "Y": <base64-encoded TrustTokenPublicKey>,
                     "expiry": <key expiry, encoded as a string representation of
                                an integer timestamp in microseconds since the Unix
                                epoch> },
  <keyID>: { "Y": <base64-encoded public key>,
                     "expiry": <key expiry, encoded as a string representation of
                                an integer timestamp in microseconds since the Unix
                                epoch> }, ...
}
```

The `id` field is used as a unique identifier to identify this key commitment and to allow comparing the freshness of key commitments (larger values indicate a newer key commitment).

### Issuing Tokens

To support the issuance of tokens made via Trust Token calls by the client, server paths that support token issuance should parse the "Sec-Trust-Token" header as a base64 binary blob. This decoded binary blob should be interpreted as a **Trust Token Issuance Request** and passed into the _Issue_ crypto protocol along with the Issuance metadata based on the rest of the request. The result of _Issue_ (a **Trust Token Issuance Response**) should be base64 encoded and returned via the "Sec-Trust-Token" header in the HTTP response.


#### Issuance Metadata

As part of an issuance, two forms of metadata can be embedded into the token. Public metadata is embedded via the choice of key which is used as part of the issuance process (amongst the up to 3 keys configured in the key commitment) and is passed into the _Issue_ method as the keypair selection. Private metadata is embedded via a cryptographically hidden bit in the signed token itself and is passed into the _Issue_ method as the private metadata boolean.


### Redeeming Tokens

To support the redemption of tokens by the client, server paths that support token redemption should parse the "Sec-Trust-Token" header as a base64 binary blob. This decoded binary blob should be interpreted as a **Trust Token Redemption Request** and passed into the _Redeem_ crypto protocol. The result of _Redeem_ (a **Trust Token Redemption Response**) should be base64 encoded and returned via the "Sec-Trust-Token" header in the HTTP response. Additionally, the issuer should keep track of redeemed tokens to prevent token reuse from malicious clients.


#### Redemption Metadata

At redemption time, the metadata encoded in the Trust Token will be embedded in the Signed Redemption Record. The public metadata is embedded via the key ID of the corresponding key used to sign the Trust Token. The private metadata bit is embedded as a one-bit boolean that can be set per redeemer logic and which can be decoded by partners downstream. One potential means of encoding the private metadata is to hash a shared secret and the token hash and take the lowest bit XOR between that and the metadata value (H(shared\_secret||token-hash) ^ private metadata value), allowing only partners with the shared secret to get the true value of the private metadata bit. Other potential schemes involve an additional endpoint which can be used to read the private metadata value based on the Signed Redemption Record (SRR) or token hash.


## Trust Token Crypto Protocol

The Trust Token crypto protocol is based on the PMBTokens design in https://eprint.iacr.org/2020/072 (appendix H) using P-384. The necessary keys and function mappings are described below.


### Keys

The Trust Token protocol primarily requires two kinds of keypairs:

*   SRRSigningKey/SRRVerificationKey - A Ed25519 public keypair for signing and verifying the integrity of the Signed Redemption Record response.
*   TrustTokenSecretKey/TrustTokenPublicKey - A Trust Token keypair used to sign and verify Trust Tokens.

#### SRRSigningKey/SRRVerificationKey

These keys should be generated and stored as standard Ed25519 keys, with the public/verification key being included in the Key Commitment result as a 32-byte RFC8032 encoding.


#### TrustTokenSecretKey/TrustTokenPublicKey

These are keys used in Trust Token consisting of elliptic curve scalars and points. All scalars and points are sized based on the curve choice (P-384). Up to 3 keys may be configured in parallel in the key commitment.

Each keypair consist of the following:


```
opaque ECPoint<1..2^16-1>; // X9.62 Uncompressed point.
opaque Scalar<Ns>; // big-endian bytestring

struct {
  // Corresponding to the FALSE private metadata bit (x0, y0).
  Scalar x0;
  Scalar y0;
  // Corresponding to the TRUE private metadata bit (x1, y1).
  Scalar x1;
  Scalar y1;
  // Corresponding to the token validity (x˜, y˜).
  Scalar xs;
  Scalar ys;
} TrustTokenSecretKey;

struct {
  ECPoint pub0; // Corresponding to the FALSE private metadata bit.
  ECPoint pub1; // Corresponding to the TRUE private metadata bit.
  ECPoint pubs; // Corresponding to the token validity check.
} TrustTokenPublicKey;
```

### Serialization/Hashing

For the PMBTokens functions, the following serialization schemes and hashes are used internally using draft 07 of the hash-to-curve specification (https://tools.ietf.org/html/draft-irtf-cfrg-hash-to-curve-07):


`Ht(t)` is defined to be P384_XMD:SHA-512_SSWU_RO_ with a big-endian bytestring input of `t` and a dst of "PMBTokens Experiment V1 HashT".

`Hs(T, s)` is defined to be P384_XMD:SHA-512_SSWU_RO_ with a bytestring input of `T` with the following content and a dst of "PMBTokens Experiment V1 HashS".

```
struct {
  ECPoint t;
  opaque s[Nn];
} T;
```

The hash-to-curve document does not define hash to scalars, so `Hc(x)` is defined to be the output of the [hash_to_field](https://tools.ietf.org/html/draft-irtf-cfrg-hash-to-curve-07#section-5.2) function with the following parameters:

* `DST` is "PMBTokens Experiment V1 HashC"
* `F`, `p`, and `m` are defined according to the finite field `GF(r)`, where `r` is the order of P-384. Note this is a different modulus from `hash_to_field` as used in P384_XMD:SHA-512_SSWU_RO_.
* `L` is 72, derived based on P384_XMD:SHA-512_SSWU_RO_'s security parameter `k` (192), and `p` defined above.
* `expand_message` uses the corresponding function from P384_XMD:SHA-512_SSWU_RO_.

When used in the DLEQ proof, `Hc` takes the following input.

```
struct {
  uint8 label[6] = "DLEQ2\0";
  ECPoint X;
  ECPoint T;
  ECPoint S;
  ECPoint W;
  ECPoint K0;
  ECPoint K1;
} DLEQInput; // Used for DLEQ proof.
```

When used in the DLEQOR proof, `Hc` takes the following input.

```
struct {
  uint8 label[8] = "DLEQOR2\0";
  ECPoint X0;
  ECPoint X1;
  ECPoint T;
  ECPoint S;
  ECPoint W;
  ECPoint K00;
  ECPoint K01;
  ECPoint K10;
  ECPoint K11;
} DLEQORInput; // Used for DLEQOR proof.
```

When used in the DLEQ batching step, `Hc` is called once for each `e_i` output, with the following input. The `index` field contains `i`.

```
struct {
  uint8 label[11] = "DLEQ Batch\0";
  ECPoint pubs;
  ECPoint pub0;
  ECPoint pub1;
  ECPoint T'0;
  ECPoint S'0;
  ECPoint Wp0;
  ECPoint Wsp0;
  ...
  ECPoint T'n;
  ECPoint S'n;
  ECPoint Wpn;
  ECPoint Wspn;
  uint16 index;
} DLEQBatchInput; // Used for Batch proof.
```

### Issue Function

The _Issue_ function corresponds to the **AT.Sig** stage of the PMBTokens protocol.

Inputs:

*   count (the number of tokens being requested, from the IssueRequest)
*   nonces (the list of nonces to sign)
*   secretKey (the secret key that should be used to sign this request, determined by the public metadata)
*   publicKey (the public key corresponding to the secretKey)
*   keyID (the ID corresponding to this key)
*   privateMetadata (the boolean value corresponding to the private metadata value)
*   toIssue (number of tokens to issue)

Outputs:

*   issued (the number of tokens being issued)
*   keyID (the key used to sign these tokens)
*   signed (the list of signed nonces)
*   cs, us, vs, c0, u0, v0, c1, u1, v1 (the list of scalars corresponding to the zero-knowledge proof)

```
Issue:
  issued = min(count, toIssue)
  xb,yb = secretKey.x1, secretKey.y1
  if privateMetadata == 0:
    xb,yb = secretKey.x0, secretKey.y0
  signed = []
  T,S,W,Ws = [], [], [], []
  for i in 0..issued:
    T' = nonces[i]
    s ←$ {0, 1}λ
    S' = Hs(T', s)
    Wp = xb*T' + yb*S'
    Wsp = secretKey.xs*T' + secretKey.ys*S'
    T,S,W,Ws += T', S', Wp, Wsp
    signed += SignedNonce{s, Wp, Wsp}
  proof = DLEQbatched.P((publicKey,T,S,W,Ws),(secretKey.xs, secretKey.ys, xb, yb))
  return (issued, keyID, signed, proof)

DLEQbatched.P((X,T,S,W,Ws),(xs, ys, xb, yb)):
  e0,...,em = Hc(X,T,S,W,Ws)
  Tt = Sum(ej*Tj) // over j from 0 to m
  St = Sum(ej*Sj) // over j from 0 to m
  Wt = Sum(ej*Wj) // over j from 0 to m
  Wst = Sum(ej*Wsj) // over j from 0 to m
  dleqProof = DLEQ2.P((X,T,S,Ws), (xs,ys)) // cs,us,vs
  dleqorProof = DLEQOR2.P((X,T,S,W), (xb,yb)) // c0,c1,u0,u1,v0,v1
  return dleqProof + dleqorProof
```

Input Serialization:

The Trust Token Issuance Request contains an `IssueRequest` structure defined below.

```
struct {
  uint16 count;
  ECPoint nonces[count];
} IssueRequest;
```

Output Serialization:

The Trust Token Issuance Response contains an `IssueResponse` structure defined below.

```
struct {
  opaque s<Nn>; // big-endian bytestring
  ECPoint Wp;
  ECPoint Wsp;
} SignedNonce;

struct {
  Scalar cs;
  Scalar us;
  Scalar vs;
  Scalar c0;
  Scalar c1;
  Scalar u0;
  Scalar u1;
  Scalar v0;
  Scalar v1;
} DLEQProof;

struct {
  uint16 issued;
  uint32 key_id = keyID;
  SignedNonce signed[issued];
  opaque proof<1..2^16-1>; // Length-prefixed form of DLEQProof.
} IssueResponse;
```

### Redeem Function

The _Redeem_ function corresponds to the **AT.VerValid** and **AT.ReadBit** stage of the PMBTokens protocol.

Inputs:

*   token (The trust token to redeem)
*   client\_data (the client data sent as part of the redemption request to include in the SRR)
*   redemptionTime (the redemption time from the client)
*   secretKey (the secret key that should be used to sign this request, determined by the public metadata)
*   lifetime (lifetime for the SRR)
*   keys (dictionary from known key IDs to secret/public keys)
*   srrKey (SRRSigningKey)

Outputs:

*   srr (the Signed Redemption Record, a binding of the client data and metadata to this redemption)
*   signature (the signature over the SRR)

```
Redeem:
  secretKey, publicKey = keys[token.key_id]
  T = Ht(token.nonce)
  if token.Ws != secretKey.xs*T+secretKey.ys*token.S:
    return 0 // Invalid validity verification.
  W0 = secretKey.x0*T+secretKey.y0*token.S
  W1 = secretKey.x1*T+secretKey.y1*token.S
  // These equalities are timing-sensitive as they could be a timing side-channel.
  isW0 = constantTimeEquality(tokenW, W0)
  isW1 = constantTimeEquality(tokenW, W1)
  if !(isW0 ^ isW1):
    return 0 // Internal Error
  privateMetadata = isW1
  tokenHash = SHA256("TrustTokenV0 TokenHash"||token)
  // Issuer-specific logic for encoding the private metadata bit.
  encodedPrivate = EncodePrivateMetadata(privateMetadata)
  srr = ConstructCBOR({
    "metadata": { "public": token.key_id, "private": encodedPrivate},
    "token-hash": tokenHash,
    "client-data": client_data,
    "expiry-timestamp": redemptionTime + lifetime
  }) // Function to correctly encode a CBOR structure into a bytestring.
  signature = Ed25519Sign(srrKey, srr)
  return (srr, signature)
```

Input Serialization:

The Trust Token Redemption Request contains a `RedemptionRequest` structure as defined below.

```
struct {
  uint32 key_id;
  opaque nonce<nonce_size>;
  ECPoint S;
  ECPoint W;
  ECPoint Ws;
} Token;

struct {
  opaque token<1..2^16-1>; // Bytestring containing a serialized Token struct.
  opaque client_data<1..2^16-1>;
  uint64 redemption_time;
} RedeemRequest;
```


Output Serialization:

The Trust Token Redemption Response contains a `RedemptionResponse` structure as defined below.

```
struct {
  opaque srr<1..2^16-1>;
  opaque signature<1..2^16-1>;
} RedeemResponse;
```