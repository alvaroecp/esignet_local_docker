# esignet_local_docker

## Overview

A local installation of [eSignet](https://github.com/mosip/esignet) 2.0.0, the OpenID Connect identity provider of MOSIP, with Docker Compose on Windows 10 and WSL 2. It is meant for testing relying parties (RPs) on a single computer, not for production: its issuer is `http://localhost:8080`, without TLS.

This repository only documents how it was installed and what was observed of its API. It contains no eSignet code.

Relying party tested with this setup (one RP, made of a backend and a frontend):

- [rp_backend_dart_frog](https://github.com/alvaroecp/rp_backend_dart_frog) (backend)
- [rp_frontend_windows_flutter](https://github.com/alvaroecp/rp_frontend_windows_flutter) (frontend)

Components, as they run in Docker:

| Component | Container | Image | Host port | What it does |
| --- | --- | --- | --- | --- |
| Login interface | `esignet-esignet-ui-1` | `mosipid/oidc-ui:2.0.0` | 3000 | The web page where the user authenticates |
| eSignet service | `esignet-esignet-1` | `mosipid/esignet:2.0.0` | 8080 | The OpenID Connect server (the issuer) |
| Mock identity system | `esignet-mock-identity-system-1` | `mosipid/mock-identity-system:0.14.0` | 8082 | A mock identity registry, with records of demo people |
| Database | `esignet-database-1` | `postgres:bookworm` | 5455 | PostgreSQL (port 5432 inside the container) |

Environment where it was tested:

| Item | Value |
| --- | --- |
| Windows | 10 Pro 22H2 (build 10.0.19045.6466) |
| WSL | Package 3.0.3.0, WSL 2, kernel 6.18.54.1 |
| Linux distribution | Ubuntu 26.04.1 LTS |
| Docker | 29.1.3, the engine inside WSL (from Ubuntu's packages), not Docker Desktop |
| Docker Compose | 2.40.3 |
| Memory | 15 GiB assigned to WSL; about 2.5 GiB in use with the four containers running (that figure covers all of WSL, not only these containers) |

**Status.** In the tests it worked and was used with one RP (a backend and a frontend). The tests were limited and not exhaustive, and everything here comes from a single machine: please read it as observations, not as guarantees.

## Installation

Requirements:

- The environment above: WSL 2 with a Linux distribution, and a Docker engine with Compose v2 (`docker compose`). Other setups were not tried.
- According to the official guide: at least 4 GB of free RAM (8 GB recommended) and about 2 GB of disk for the images.
- Free ports: 3000, 5455, 8080 and 8082.

The steps follow the [official Docker Compose guide](https://github.com/mosip/esignet/blob/master/docker-compose/README.md), which also covers troubleshooting.

1. Clone the repository and enter the Compose folder. In this setup the clone lives in `~/esignet`, and the commands of the next sections assume that path:

```bash
cd ~
git clone --depth 1 https://github.com/mosip/esignet.git
cd esignet/docker-compose
```

2. Optionally, preview the ports and images before starting anything:

```bash
docker compose -p esignet config | grep -E "published|image:"
```

For example, the previous command listed the four images and the four host ports that Compose would use, each port right after its image:

```
    image: postgres:bookworm
        published: "5455"
    image: mosipid/esignet:2.0.0
        published: "8080"
    image: mosipid/oidc-ui:2.0.0
        published: "3000"
    image: mosipid/mock-identity-system:0.14.0
        published: "8082"
```

3. Start it. `-p esignet` names the Compose project: repeat it in every later Compose command.

```bash
docker compose -p esignet up -d
```

4. Check that it is up. `ps` should list four services, with the database `healthy`:

```bash
docker compose -p esignet ps
```

**Warning.** It is not known what would happen with `docker compose down` (or with `-v`), and it was preferred not to try it. Apparently the database and the mock identity system keep their data (registered clients, people and keys) inside their containers, so it might be lost; for that reason, `down` is best avoided. The official configuration seems designed to be recreated every time, not to be stopped and started.

## Deployment notes

What was followed, what was adjusted and how long it took.

| Question | What happened |
| --- | --- |
| What was followed | The [official Docker Compose guide](https://github.com/mosip/esignet/blob/master/docker-compose/README.md) of the eSignet repository: clone it, enter `docker-compose` and run `docker compose -p esignet up -d` (see Installation) |
| Version of the compose files | The cloned repository was at commit `db52842`, dated 2026-09-25, as shown by `git log` in the local clone |
| What was adjusted in the compose file | Nothing. The file and its environment variables were used as they come in the cloned repository |
| Versions | The ones the compose file defines (see the table of components in the Overview) |
| Ports | Unchanged: 3000, 5455, 8080 and 8082. Port 8080 was at first held by a container of an earlier, unrelated project, which was stopped before starting eSignet |
| Memory | No memory setting was added to the compose file. WSL had 15 GiB assigned, and about 2.5 GiB were in use with the four containers running |
| How long it took | A few minutes, from memory; no time was recorded. The first start downloads the images, so it probably depends on the connection |
| Problems during the installation | None came up during the installation. The problems appeared later, when the PC was switched off and on (see Daily operation) |
| Other adjustments | One, made later and outside the compose file: a change to a start script inside the mock identity container (see Daily operation) |

## Daily operation

In the tests, the PC was switched off and on. The services came back after a few adjustments, described below, and the containers kept their data.

After restarting the PC, start Docker if it is not running (for example, `sudo service docker start`). Then start the services one by one, in this order (the one that worked in the tests), waiting for each one to be up before starting the next:

```bash
cd ~/esignet/docker-compose
docker compose -p esignet start database
docker compose -p esignet start mock-identity-system
docker compose -p esignet start esignet
docker compose -p esignet up -d --no-deps --force-recreate esignet-ui
docker compose -p esignet ps
```

The login interface is recreated instead of started because, in the tests, it started and stopped again by itself; the cause was not confirmed. It keeps no data, so recreating it seemed harmless.

A problem seen in the tests:

**The mock identity system failed from its third start**, with `mv: cannot overwrite 'hsm-client/client'` in its log (`docker logs esignet-mock-identity-system-1`). Its start script seems to reinstall the HSM client on every start. What worked was to change one line of that script inside the stopped container, so that the installation runs only if the `hsm-client` folder does not exist yet. It was tried with the image `mosipid/mock-identity-system:0.14.0`; `diff` should show a single changed line:

```bash
docker cp esignet-mock-identity-system-1:/home/mosip/configure_start.sh .
cp configure_start.sh configure_start.sh.original
sed -i 's|if \[ "\$active_profile_env" != "local" \]; then|if [ "$active_profile_env" != "local" ] \&\& [ ! -d "$DIR_NAME" ]; then|' configure_start.sh
diff configure_start.sh.original configure_start.sh
docker cp configure_start.sh esignet-mock-identity-system-1:/home/mosip/configure_start.sh
```

The change is made inside the container, so it would be lost if the container were recreated.

## API

eSignet publishes its metadata at `http://localhost:8080/.well-known/openid-configuration`. In this installation, it lists these endpoints:

| Purpose | URL |
| --- | --- |
| Issuer | `http://localhost:8080` |
| Authorization | `http://localhost:8080/oauth2/authorize` |
| Token | `http://localhost:8080/oauth2/token` |
| UserInfo | `http://localhost:8080/oauth2/userinfo` |
| Keys (JWKS) | `http://localhost:8080/oauth2/jwks` |
| Introspection | `http://localhost:8080/oauth2/introspect` |
| Pushed authorization requests (PAR) | `http://localhost:8080/oauth2/par` |

Other endpoints used in this setup: `GET http://localhost:8080/health`, `GET http://localhost:8082/v1/mock-identity-system/actuator/health`, `POST http://localhost:8080/client-mgmt/client` (client registration) and `POST http://localhost:8082/v1/mock-identity-system/identity` (people). The last two are described in the sections below.

What the discovery document announces:

| Topic | Value |
| --- | --- |
| Response and grant types | Only `code` and `authorization_code` are listed; no refresh token grant is announced |
| Client authentication | `private_key_jwt` only |
| Algorithms of the client assertion | PS256, ES256, ES256K and EdDSA |
| PKCE | `S256` |
| `iss` parameter in the callback | Supported |
| Scopes | `openid`, `profile`, `email` and `phone` |
| Claims | `name`, `address`, `gender`, `birthdate`, `picture`, `email` and `phone_number`. The `claims` parameter is supported |
| Authentication contexts (`acr`) | `mosip:idp:acr:biometrics`, `mosip:idp:acr:generated-code`, `mosip:idp:acr:knowledge` and `mosip:idp:acr:password` |
| Subject type | `pairwise` |
| Signing of `id_token` and `userinfo` | PS256 |
| Encryption of `id_token` and `userinfo` | JWE is listed as supported (RSA-OAEP and RSA-OAEP-256, with AES-CBC-HMAC and AES-GCM) |
| DPoP | PS256, ES256, ES256K and EdDSA |
| Sign-out | No `end_session_endpoint` appears in the document, so a global sign-out is apparently not offered |

Some behaviors observed in the tests are listed below. The tests were not exhaustive, so some observations may be inaccurate:

| Topic | Observation |
| --- | --- |
| `kid` in the assertion | Apparently required: without it, eSignet answered `401 invalid_client` ("Invalid client assertion") |
| `aud` of the assertion | In the tests it had to be the issuer (`http://localhost:8080`); the token endpoint URL was rejected. The official guide describes the token endpoint URL as the default and the issuer as the strict mode; which mode this installation uses was not checked |
| Algorithm of the assertion | PS256 and RS256 were both accepted, although the discovery document does not list RS256 |
| Token lifetime | About 120 seconds in the tests; no refresh token was seen |
| `id_token` | Signed with PS256. In the tests it carried `acr`, `auth_time`, `nonce`, `sub` and `jti`, and no `at_hash` |
| `access_token` | In the tests, a JWT signed with PS256 with the same key as the `id_token`, with header `typ: at+jwt` |
| `userinfo` | In the tests it was a signed JWT, but signed with RS256 using the mock identity system's key, which the JWKS publishes as PS256. Its `iss` is `http://localhost:8088/v1/esignet`, not the issuer (port 8088 is used inside the eSignet container and is not published to the host). This is probably a quirk of the mock identity system, not of eSignet; it was not verified |
| `sub` | The same value came back on every login of the same person with the same client. The discovery document says `pairwise`; whether it differs between clients was not compared |
| `picture` | In the tests it arrived as an inline image (`data:image/jpeg;base64,…`), not as a URL |
| Registered clients | No way to delete them was found |

## Key pair of a client

A client authenticates with `private_key_jwt`: the RP signs a JWT with its private key, and eSignet checks it with the public key registered for that client.

The commands below create an RSA 2048 key pair in a PEM file (`client-private.pem`) and, from it, write two JWK files: `client-public.jwk`, the only one that goes into the registration, and `client-private.jwk`, for the RP. They need `openssl` and `python3`. The format is the one used in the tests; other key types or sizes were not tried.

<details>
<summary>Commands (bash)</summary>

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out client-private.pem

python3 - << 'EOF'
import base64, hashlib, json, re, subprocess

def b64u(data):
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()

text = subprocess.check_output(
    ["openssl", "rsa", "-in", "client-private.pem", "-noout", "-text"], text=True)

def number(label):
    m = re.search(r"^" + label + r":\s*\n((?:[ \t]+[0-9a-f:]+\n)+)", text, re.M)
    value = int(re.sub(r"[^0-9a-f]", "", m.group(1)), 16)
    return b64u(value.to_bytes((value.bit_length() + 7) // 8, "big"))

n, e = number("modulus"), "AQAB"
thumbprint = json.dumps({"e": e, "kty": "RSA", "n": n}, separators=(",", ":"))
kid = b64u(hashlib.sha256(thumbprint.encode()).digest())

public = {"kty": "RSA", "n": n, "e": e, "kid": kid, "use": "sig"}
private = {"kty": "RSA", "n": n, "e": e, "d": number("privateExponent"),
           "p": number("prime1"), "q": number("prime2"), "dp": number("exponent1"),
           "dq": number("exponent2"), "qi": number("coefficient"), "kid": kid, "use": "sig"}
json.dump(public, open("client-public.jwk", "w"))
json.dump(private, open("client-private.jwk", "w"))
EOF

chmod 600 client-private.pem client-private.jwk
```

</details>

Notes:

- The `kid` is the SHA-256 thumbprint of the public key (RFC 7638). In the tests, the registration and the RP's client assertions used the same `kid`.
- `client-private.pem` holds the whole pair, because the public key can be derived from the private one. The two JWK files are two views of that same key, not two different pairs: `client-public.jwk` has only the public part (`n`, `e`), so it is the one that can be sent to eSignet, while `client-private.jwk` also has the private components.
- The public exponent is 65537 (`AQAB` in base64url), OpenSSL's default.
- The commands overwrite `client-private.pem`, `client-private.jwk` and `client-public.jwk` if they already exist in the current folder: run them in an empty folder.
- Keep the private files out of Git and out of shared folders. If the private key is lost, the client apparently becomes unusable (see the next section).

## Client registration

It registers an RP in eSignet. It is a `POST http://localhost:8080/client-mgmt/client` with `Content-Type: application/json`. In the tests that were run it did not ask for credentials.

The body has `requestTime` (an ISO 8601 date in UTC, with milliseconds) and a `request` object. The fields follow the example in eSignet's Postman collection:

| Field | What it is | Value used in the tests |
| --- | --- | --- |
| `clientId` | The client's identifier | One per RP, for example `demo-rp-app` |
| `clientName`, `clientNameLangMap` | The name, and the name per language | `Demo RP`, in `eng` |
| `relyingPartyId` | The entity the client belongs to | `decl-ou-1`, taken from the Postman example |
| `logoUri` | A logo | A placeholder address |
| `redirectUris` | The return addresses | One or several, absolute |
| `userClaims` | Claims the client may ask for | `name`, `email`, `picture`, `phone_number`, `address` |
| `authContextRefs` | Authentication contexts (`acr`) | `generated-code`, `password` and `biometrics` |
| `publicKey` | The public key only, as a JWK | RSA, with `kid` and `use: sig` |
| `grantTypes` | Grant types | `authorization_code` |
| `clientAuthMethods` | Client authentication | `private_key_jwt` |
| `additionalConfig` | Per-client options | PKCE required; PAR and DPoP off |

In the tests, a successful registration returned a body like the one below, with `response.status` equal to `ACTIVE`. No `errors` field appeared in it:

```json
{"responseTime":"2026-11-08T18:44:57.969Z","response":{"clientId":"demo-rp-example","status":"ACTIVE"}}
```

The example assumes that `client-public.jwk` already exists (see the previous section). Run as written, it returned the answer above.

<details>
<summary>Example with curl (bash)</summary>

```bash
curl -s -X POST http://localhost:8080/client-mgmt/client \
  -H "Content-Type: application/json" \
  -d @- << EOF
{
  "requestTime": "$(date -u +%Y-%m-%dT%H:%M:%S.000Z)",
  "request": {
    "clientId": "demo-rp-example",
    "clientName": "Demo RP",
    "clientNameLangMap": { "eng": "Demo RP" },
    "relyingPartyId": "decl-ou-1",
    "logoUri": "https://example.com/logo.png",
    "redirectUris": ["http://localhost:5000/callback"],
    "userClaims": ["name", "email", "picture", "phone_number", "address"],
    "authContextRefs": [
      "mosip:idp:acr:generated-code",
      "mosip:idp:acr:password",
      "mosip:idp:acr:biometrics"
    ],
    "publicKey": $(cat client-public.jwk),
    "grantTypes": ["authorization_code"],
    "clientAuthMethods": ["private_key_jwt"],
    "additionalConfig": {
      "require_pushed_authorization_requests": false,
      "dpop_bound_access_tokens": false,
      "require_pkce": true
    }
  }
}
EOF
```

Running it leaves a client registered, and apparently it cannot be deleted (see the notes below).

</details>

Notes:

- To use it with the RP linked in the Overview, change `redirectUris` to `["http://localhost:3005/callback"]` and `clientId` to the value you will use as `CLIENT_ID` in the RP's configuration.
- Only the public key was sent. Its `kid` appears to be the one the RP has to put in the header of its client assertion: without a `kid`, eSignet answered `401 invalid_client`.
- When a private key was lost, the client became unusable and no way to recover it was found. The way out was to register another client with a different name.
- The Postman collection also has operations to read (`GET`), replace (`PUT`) and modify (`PATCH`) a client by its `client_id`. They were not tried. Apparently there is no way to delete or list clients: no such operation appears in the collection, and none was found in the tests.

## User registration

It creates a test person in the mock identity system, not in eSignet. It is a `POST http://localhost:8082/v1/mock-identity-system/identity` with `Content-Type: application/json`. In the tests that were run it did not ask for credentials either.

The body has `requestTime` and a `request` object. Fields with text per language are lists of `{ "language": "eng", "value": "…" }`. The fields follow the example in the Postman collection:

| Group | Fields |
| --- | --- |
| Identification | `individualId` (the UIN, unique identification number, used to sign in), `pin`, `password` |
| Contact | `email`, `phone` |
| Name | `fullName`, `nickName`, `preferredUsername`, `givenName`, `middleName`, `familyName` |
| Personal data | `gender` (`Male` or `Female`), `dateOfBirth` (format `YYYY/MM/DD`) |
| Address | `streetAddress`, `locality`, `region`, `country`, `postalCode` |
| Preferences | `preferredLang`, `locale`, `zoneInfo` |
| Photo | `encodedPhoto`, an image as text: `data:image/jpeg;base64,…` |

In the tests it answered `HTTP 200` even when it failed, so it is better to check `errors`: the person existed when `errors` was empty and `response.status` said `mock identity data created successfully`. A successful answer looked like this:

```json
{"responseTime":"2026-11-08T19:10:42.780Z","response":{"status":"mock identity data created successfully"},"errors":[]}
```

The example follows the body of the Postman collection, with test values. The `individualId` is made up, with 10 digits as the login screen asks, and the photo is a 2×2-pixel PNG from the Postman example. Run as written, it returned the answer above.

<details>
<summary>Example with curl (bash)</summary>

```bash
curl -s -X POST http://localhost:8082/v1/mock-identity-system/identity \
  -H "Content-Type: application/json" \
  -d @- << EOF
{
  "requestTime": "$(date -u +%Y-%m-%dT%H:%M:%S.000Z)",
  "request": {
    "individualId": "1234567891",
    "pin": "111111",
    "password": "Mosip@123",
    "email": "demo.user@example.com",
    "phone": "+51987654321",
    "fullName": [{ "language": "eng", "value": "Alvaro Demo" }],
    "nickName": [{ "language": "eng", "value": "Alvaro Demo" }],
    "preferredUsername": [{ "language": "eng", "value": "Alvaro Demo" }],
    "givenName": [{ "language": "eng", "value": "Alvaro" }],
    "familyName": [{ "language": "eng", "value": "Demo" }],
    "gender": [{ "language": "eng", "value": "Male" }],
    "dateOfBirth": "1990/01/01",
    "streetAddress": [{ "language": "eng", "value": "Av. Prueba 123" }],
    "locality": [{ "language": "eng", "value": "Lima" }],
    "region": [{ "language": "eng", "value": "Lima" }],
    "country": [{ "language": "eng", "value": "Peru" }],
    "postalCode": "15001",
    "preferredLang": "eng",
    "locale": "en",
    "zoneInfo": "America/Lima",
    "encodedPhoto": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAIAAAACCAIAAAD91JpzAAAAEElEQVR4nGM4YWQERAwQCgAjNgSxRnwWMwAAAABJRU5ErkJggg=="
  }
}
EOF
```

</details>

Notes:

- `middleName` can apparently be omitted, as the example does: it was accepted without it.
- The postal code seems to be required: when it was missing or not valid, the server answered `invalid_postalcode`.
- eSignet's login screen rejected an 8-digit UIN with the message "UIN/VID must be between 10 and 16 digits".
- The photo was reduced to 256 px on its longest side, presumably to keep small the `userinfo`, which is a JWT and carries it. Other sizes were not tried.
- The data is probably stored in plain text in the demo database (not verified): do not use real personal data.
- With `scope` `openid profile email`, the profile showed `name`, `email` and `picture`, but not the phone number or the address.

These fields come from eSignet's Postman collection and from the tests that were run. Not every combination was tried, and it is not known which fields are mandatory beyond the ones noted above.
