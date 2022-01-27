# Verification solution

## OpenID Connect and OAuth 2.0

The verification solution uses [Identity server](https://identityserver4.readthedocs.io/en/latest/) to implement the OpenID connect and OAuth 2.0 specifications.

## Configured clients

The only client that is intended to use the verification solution is the Smittestopp exposure notification app.
As a result, this client is configured through [appsettings.json](https://github.com/folkehelseinstituttet/Fhi.Smittestopp.Verification/blob/7521b71d84593b09a4f4e4c28dfcb1ee790dfca0/Fhi.Smittestopp.Verification.Server/appsettings.json#L21) with the client id `smittestopp`.
For the dev-, test- and QA-environments, a client with id `test-spa-client` is also included to faciliate easier isolated testing of the verification solution.

## Verified access to notify contacts

To be able to notify you contacts through the Smittestopp app, you must fulfill the following 3 criterias

- Have the ability to identify yourself via logging in with [ID-porten](https://eid.difi.no/nb/id-porten)
- Be at least 16 years old
- Have NOT already performed the verification flow 3 times the last 24 hours
- - These values are for production, and may vary for other environments

Each time you perform the verification flow with all criterias fulfilled your verification counts towards the verification limit.

The verification limit is tracked by the verification solution itself (through the VerificationRecords DB-table which stores a hash of the ID-porten pseudonym of the user logged in for 24 hours).

If you fullfill all 3 criterias (identification,age and verification limit), the JWT access token issued to you will contain the `role`-claim set to `upload-approved` and `covid19_blokeret`-claim set to `false`

To avoid replay attacks, one can store the `jti`-claim for the valid period of the claim (given by the `exp`-claim).

Note: As of february 2022, the hard requirement of a verifiable positive test in MSIS to get the `role` -claim set to `upload-approved` has been dropped

## Request to skip MSIS lookup

A user can request to skip looking for test results in MSIS. This is done by requesting an additional scope: `no-msis`

The `covid19_status` claim will then be set to `negativ` and the MSIS lookup of covid-19 test results will be omitted in its entirety (e.g no calls to MSIS are being made), but the user will still get the `role`-claim set to `upload-approved` and `covid19_blokeret`-claim set to `false` to be able to generate an anonymous token and notify contacts, as long as the verification limit has not yet been reached for the user requesting verification.

## Compatibility with Smitte|Stop (DK)

Since the norwegian Smittestopp is based on the danish Smitte|Stop, the decision was made to use JWTs compatible with the existing DK-implementation.

To request a DK-compatible JWT access token from the Smittestopp Verification, one should request the scope `smittestop`.

A user which *does not have* a positive COVID-19 test within the last 14 days, or *is below* 16 years of age, will then have the following claims

```
covid19_status=negativ
```

If there is an unexpected error during the processing of a token request, the following claim will be given provided,
which then should trigger a message to the user in app informing them of the error, and asking them to try again later.

```
covid19_status=ukend
```

A user which *does have* a positive COVID-19 test within the last 14 days, *is at least* 16 years old,
and has not performed the verification flow more than the allowed number of times will have the following claims.

```
covid19_status=positiv
covid19_smitte_start=yyyy-MM-dd // sampling date of COVID-19 test
covid19_blokeret=false
```



A user which *does have* a negative COVID-19 test OR requested to skip the MSIS lookup, *is at least* 16 years old,
and has not performed the verification flow more than the allowed number of times will have the following claims.

```
covid19_status=negativ
covid19_blokeret=false
```


If a user satisfies the requirements for age, but has been blocked due to too many attempts at performing the verification flow,
the blocking claim changes value, and some additional claims are added informing the app of the duration and number of attempts configured for the blocking limit.

```
covid19_blokeret=true
covid19_limit_duration=24
covid19_limit_count=3
```

Thus, to authorize a user to upload their diagnosis keys using only the DK-compatible claims,
one must require that the `covid19_status` claim is present with a value `positiv` and that the `covid19_blokeret` claim is present with a value `false`.

As of february 2022, users get to upload their tokens to retrieve an anonymous token even if they have a `covid19_status `claim with a value of `negativ`

## Anonymous tokens

When anonymous tokens are enabled, an additional claim is introduced in the JWT access token: `covid19_anonymous_token=v1_enabled`

This claims is intended to communicate to the client (Smittestopp app), that anonymous tokens are available,
and should therefore exchange the JWT for an anonymous token when uploading diagnosis keys.
[This is described further here](anonymous-tokens.md)

## Internal dependencies

The verification solution has the following internal dependencies

- [ID-porten](https://www.digdir.no/digitale-felleslosninger/id-porten/864): External identity provider used to authenticate users and retrieve national identifier to perform the verification for
- [MSIS](https://www.fhi.no/en/hn/health-registries/msis/): Used to check that both the infection status requirement and the age requirement is fullfilled for a provided national identifier

## Environments

- Azure-dev
  - Url: https://dev-smittestopp-verification.azurewebsites.net/.well-known/openid-configuration/
  - ID-porten [ver1](https://docs.digdir.no/oidc_func_wellknown.html) environment ([test users](https://docs.digdir.no/idporten_testbrukere.html))
  - [Fixed test cases for MSIS check and block limit](https://github.com/folkehelseinstituttet/Fhi.Smittestopp.Verification/blob/7521b71d84593b09a4f4e4c28dfcb1ee790dfca0/Fhi.Smittestopp.Verification.Server/appsettings.AzDev.json#L71)
- NHN-test
  - Url: https://test-vl-op.ss2np.fhi.no/.well-known/openid-configuration/
  - ID-porten [ver1](https://docs.digdir.no/oidc_func_wellknown.html) environment ([test users](https://docs.digdir.no/idporten_testbrukere.html))
  - MSIS-check against test data
  - Block limit increased, block duration decreased
- NHN-qa
  - Url: https://qa-vl-op.ss2np.fhi.no/.well-known/openid-configuration/
  - ID-porten [ver1](https://docs.digdir.no/oidc_func_wellknown.html) environment ([test users](https://docs.digdir.no/idporten_testbrukere.html))
  - MSIS-check against test data
  - Block limit and duration as prod
- NHN-prod
  - Url: https://vl-op.ss2.fhi.no/.well-known/openid-configuration/
  - ID-porten [production](https://docs.digdir.no/oidc_func_wellknown.html) environment
  - MSIS-check against production data
  - Block limit max 3 attempts per 24 hours
