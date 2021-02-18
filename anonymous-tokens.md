# Anonymous tokens

As a additional privacy measure, an an alternate authorization method for uploading TEKs has been introduced.
This method exchanges the JWT access token for an anonymous token, which removes the possibility of the two backend solutions,
Fhi.Smittestopp.Backend and Fhi.Smittestopp.Verification, to link verification attempts to uploaded TEKs through the token used for authorization.
You can find more information about what anonymous tokens are and how they work in [the github repo wiki](https://github.com/HenrikWM/anonymous-tokens/wiki).
The [readme of the github repo](https://github.com/HenrikWM/anonymous-tokens) also contains sample code for how to use the package.

## New infection verification and TEK upload flow

The TEK upload flow using anonymous tokens starts in exactly the same way as the default flow [described here](./gaen-processes.md#verify-infection-and-notify-exposure),
but triggers a new behaviour in the client through a new feature flag in the JWT access token issued to the client after verification.
The changes in the flow is shown from step 1.2 onwards in the diagram below.

![Smittestopp components overview](diagrams/Smittestopp_notify_contacts_anonymous_tokens_en.svg)

### New verification solution behaviour

#### Feature flag

Enabling/disabling the anonymous token feature is controlled from the verification solution.
Internally, this feature is enabled/disabled through the appsetting  `common:anonymousTokens:enabled`.
If enabled, and the authenticated user is verified COVID-19 positive,
the JWT access token will include a new claim, `covid19_anonymous_token=v1_enabled`, which will activate the new behaviour in the client. The backend should still accept JWT tokens for authentication, regardless of this feature flag, to retain backwards compatibility with older versions of the app.

#### Anonymous token endpoints

Two new endpoints have been added to the verification solution, and are active if the feature flag is enabled.

The first endpoint is used for exchanging a JWT-access token for an anonymous token. Access to this endpoint is granted through the claim `role=upload-approved` (combines positive test found and not blocked).

```bash
curl --location --request POST 'https://localhost:5001/api/anonymoustokens' \
--header 'Authorization: Bearer <JWT access token>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "maskedPoint": "<base64-encoded randomized token seed (t)>"
}'
```

Which should then respond with a response as follows

```json
{
    "kid": "6215",
    "signedPoint": "<base64-encoded signed point (Q)>",
    "proofChallenge": "<base64-encoded proof challenge (c)>",
    "proofResponse": "<base64-encoded proof response (z)"
}
```

The properties of the response object is calculated roughly as follows

```c#
// using AnonymousTokens.Server.Protocol
// request : AnonymousTokenRequest
// _keyStore : IAnonymousTokensKeyStore
// _tokenGenerator = new TokenGenerator()
var signingKeyPair = await _keyStore.GetActiveSigningKeyPair();
var k = signingKeyPair.PrivateKey;
var K = signingKeyPair.PublicKey;
var P = signingKeyPair.EcParameters.Curve.DecodePoint(Hex.Decode(request.PAsHex));

var token = _tokenGenerator.GenerateToken(k, K.Q, signingKeyPair.EcParameters, P);
var Q = token.Q;
var c = token.c;
var z = token.z;

return new AnonymousTokenResponse(signingKeyPair.Kid, Q, c, z);
```

The private key used in this logic (`_keyStore.GetActiveSigningKeyPair()`) must be shared with the backend service that will accept the created tokens for authenticating and authorizing requests.

The second endpoint returns a list of valid anonymous token public keys, used to validate issued anonymous tokens.
This endpoint is inspired by the specification for [JSON Web Key Sets](https://tools.ietf.org/html/rfc7517).

```bash
curl --location --request GET 'https://localhost:5001/api/anonymoustokens/atks'
```

Which should return a response as follows

```json
{
    "keys": [
        {
            "kid": "6215",
            "kty": "EC",
            "crv": "P-256",
            "x": "AMEz4DeB0qSA1OIsSXDoI4bmjwZpOpta7ZrZU7opb8ao",
            "y": "ALnIKgbW+0MCTaMOme19AM7c3LCp8uyQj0g9nWK/YkgO"
        }
    ]
}
```

Please note that the "keys" element will contain multiple elements during key rollover,
and the correct element to validate an anonymous token should be retrieved by matching the "kid" property from the anonymous token response above.

### New backend behaviour

### Anonymous authentication scheme

A new authentication option has been added to the TEK upload API on the backend server in addition to the existing `Bearer` scheme using the JWTs.

The new authentication option uses the scheme `Anonymous` and assumes the following structure for the `Authentication` header

```http
Authentication: Anonymous <submitted point>.<token seed>.<kid>
```

The validation of the incoming tokens is performed by the backend roughly as follows

```c#
// using AnonymousTokens.Core.Services
// using AnonymousTokens.Server.Protocol
// _anonymousTokenKeySource : IAnonymousTokenKeySource
// _tokenVerifier = new TokenVerifier(seedStore : ISeedStore)
string[] anonymousToken = authHeader.Replace("Anonymous ", string.Empty).Split(".");
var submittedPoint = _anonymousTokenKeySource.ECParameters.Curve.DecodePoint(Convert.FromBase64String(anonymousToken[0]));
var tokenSeed = Convert.FromBase64String(anonymousToken[1]);
var keyId = anonymousToken[2];

var privateKey = _anonymousTokenKeySource.GetPrivateKey(keyId);

var isValid = await _tokenVerifier.VerifyTokenAsync(privateKey, _anonymousTokenKeySource.ECParameters.Curve, tokenSeed, submittedPoint);
```

The `ISeedStore` must be implemented to prevent token replay attacks, and `_anonymousTokenKeySource.GetPrivateKey(keyId)` must always return the same private key for a given keyId as the one used by the verification solution, to be able to correctly verify the token.

### New client behaviour

Given that the feature flag is present in the JWT access token returned to the client,
the client will performs some additional steps prior to uploading the TEKs to the backend.

1. Perform the necessary client side initialization of the anonymous token request.
    ```c#
    // using AnonymousTokens.Client.Protocol
    // _initiator = new Initiator()
    // _ecParameters = CustomNamedCurves.GetByOid(X9ObjectIdentifiers.Prime256v1)
    var init = _initiator.Initiate(_ecParameters.Curve);
    var state =  new AnonymousTokenState
    {
        t = init.t,
        r = init.r,
        P = init.P
    };
    var tokenRequest = new GenerateTokenRequestModel
    {
        MaskedPoint = Convert.ToBase64String(state.P.GetEncoded())
    };
    ```
1. After requesting an anonymous token using the request object above,
   and the JWT from verification,
   the client randomizes the anonymous token,
   using the tokenResponse returned from the verification server,
   as well as the public key used for creating the token,
   retrieved from the atks-endpoint.
   ```c#
    // using Org.BouncyCastle.Math
    // X9ECParameters ecParameters = CustomNamedCurves.GetByName("P-256"); // or "crv" from the matching public key
    var K = DecodePublicKey(atksResponse.Keys.Single(k => k.Kid == tokenResponse.kid))
    var Q = _ecParameters.Curve.DecodePoint(Convert.FromBase64String(tokenResponse.SignedPoint));
    var c = new BigInteger(Convert.FromBase64String(tokenResponse.ProofChallenge));
    var z = new BigInteger(Convert.FromBase64String(tokenResponse.ProofResponse));
    var token = _initiator.RandomiseToken(_ecParameters, K, state.P, Q, c, z, state.r);
    ```
1. The anonymous authentication header value can then be constructed as follows
    ```c#
    var t = state.t;
    var W = token;
    var keyId = tokenResponse.Kid;
    var encodedToken = Convert.ToBase64String(W.GetEncoded()) + "." + Convert.ToBase64String(t) + "." + keyId;
    var authHeaderValue = $"Anonymous {encodedToken}"
    ```

## Environment setup requirements

### Shared private key

The same private key is used to both issue anonymous tokens from the secure token server, as well as validating calls to the API protected by the anonymous tokens. As a result, the STS (Fhi.Smittestopp.Verification) and the API (Fhi.Smittestopp.Backend) must at all times agree on the private key used.

### Rotating shared private key

An issued anonymous token will remain valid as long as the private key used to issue it is still accepted by the API.
To avoid cases where tokens issued could be used far into the future, it is desired to limit the timespan each private key is valid for.
However, a too short timespan for each key is not desired either, as that would potentially enable reidentification of the anonymous token through the private key used to issue it.

To avoid having to constantly generate new private keys, and keeping these in sync between the STS and the API,
a method of generating short lived private keys from a shared long lived master key has been adopted.
Then the requirements are reduced to agreeing on the shared master key,
and how long each generated short lived private key should be valid for. (And optionally, defining a rollover period).

The first step then is to generate the private key interval number (also used as kid) for a given point in time, which should be done as follows.

```c#
// pointInTime = DateTimeOffset.UtcNow
// _config.KeyRotationInterval = TimeSpan.FromDays(3) // default value
var intervalNumber = pointInTime.ToUnixTimeSeconds() / Convert.ToInt64(_config.KeyRotationInterval.TotalSeconds)
```

Given the interval number, and the master private key bytes, the short lived private key should then be generated as follows.

```c#
// keyIntervalNumber = intervalNumber from above
// masterKeyBytes = master private key bytes shared between the two server applications
var keyByteSize = (int) Math.Ceiling(ecParameters.Curve.Order.BitCount / 8.0);
var counter = 0;

BigInteger privateKey;
do
{
    var privateKeyBytes = GeneratePrivateKeyBytes(keyByteSize, masterKeyBytes, keyIntervalNumber, counter);
    privateKey = new BigInteger(privateKeyBytes);
    counter++;
    if (counter > 1000)
    {
        throw new GeneratePrivateKeyException("Maximum number of iterations exceeded while generating private key within curve order.");
    }
} while (privateKey.CompareTo(ecParameters.Curve.Order) > 0); // Private key must be within curve order

return privateKey;
```

The while-loop included as a safeguard to ensure the generated key is within the curve order, as it otherwise could produce a weak private key.

The method GeneratePrivateKeyBytes is defined as follows

```c#
private static byte[] GeneratePrivateKeyBytes(int numBytes, byte[] masterKeyBytes, long keyIntervalNumber, int counter)
{
    var keyIntervalBytes = BitConverter.GetBytes(keyIntervalNumber);
    var counterBytes = BitConverter.GetBytes(counter);
    var ikm = masterKeyBytes;
    var salt = keyIntervalBytes.Concat(counterBytes).ToArray();
    var hkdf = new HkdfBytesGenerator(new Sha256Digest());
    var hParams = new HkdfParameters(ikm, salt, null);
    hkdf.Init(hParams);
    byte[] keyBytes = new byte[numBytes];
    hkdf.GenerateBytes(keyBytes, 0, numBytes);
    return keyBytes;
}
```

Once you have a private key, the public key can be calculated as follows:

```c#
// BigInteger privateKey = <calculated as above>
// X9ECParameters ecParameters = CustomNamedCurves.GetByName("P-256");
ECPoint publicKey = ecParameters.G.Multiply(privateKey).Normalize()
```