---
name: PlayerWON server-side token brokering
description: Stand up the server-side half of a PlayerWON integration — exchange the game's client credentials for a short-lived access token and hand it to the game client — without ever shipping the client secret in the game binary.
api: PlayerWON API
base_url: https://game.simulmedia-apis.com
method: generated
source: https://github.com/Simulmedia/playerwon-sdk (DevGameServer/README.MD, DevGameServer/cmd/gameserver/gameserver.go)
operations:
  - POST /token
  - POST /sdk/telemetry
---

# PlayerWON server-side token brokering

PlayerWON authenticates with OAuth2 client credentials. The credentials belong to the
**game server**, never to the shipped game client. This skill covers that exchange and
the failure modes, and is the prerequisite for
`simulmedia-playerwon-rewarded-video.md`, which spends the token.

## What you need from PlayerWON
Three values, issued by PlayerWON — there are no test or sandbox values, so these are
production credentials from the first call:

- `CLIENT_ID`
- `CLIENT_SECRET`
- `GAMETITLE_ID` — sent to the API as `tid`

## The exchange
`POST https://game.simulmedia-apis.com/token`
with `Content-Type: application/x-www-form-urlencoded` and the body fields:

| field | required | value |
|---|---|---|
| `client_id` | yes | your client id |
| `client_secret` | yes | your client secret |
| `grant_type` | yes | `client_credentials` |
| `tid` | yes | your game title id |
| `idfa` | no | the player's advertising identifier, when you have one |

The response is JSON: `access_token`, `token_type`, `expires_in`. Return **only**
`access_token` to the game client. Re-run the exchange when `expires_in` elapses.

## Failure modes (observed on the live API)
- **400** `{"message":"client_id: no value"}` — a required form field was omitted. The
  message names the missing field, but there is no machine-readable error code, so
  match on the field-name prefix and treat the text as unstable.
- **401** `{"message":"Unauthorized"}` — returned by the resource endpoints when the
  Bearer token is missing, expired, or issued for a different title.
- **404** `{"message":"Not Found"}` — the path is not part of the API.

Treat any non-2xx as a hard failure. Do not retry a 400; fix the request. There is no
published rate limit and no `Retry-After` header, so back off conservatively on
repeated failures. The only correlation id available is Cloudflare's `cf-ray`
response header — log it.

## Before you have a game server
PlayerWON ships a reference implementation, the Dev Game Server, in
`DevGameServer/` of the SDK repository. Set `CLIENT_ID`, `CLIENT_SECRET`,
`GAMETITLE_ID` and optionally `PORT`, run it, and it exposes
`POST http://localhost:{PORT}/login/{user_id}` returning the access token string —
`{user_id}` is passed upstream as `idfa`. It calls **production**; it is a
convenience harness, not a sandbox.

## Rules
- Never embed `client_secret` in the shipped client. The whole point of this split is
  that the secret stays server-side.
- One token is scoped implicitly to one game title (`tid`). There are no OAuth scopes.
- Optional: the SDK reports `init`, `auth` and `error` messages to
  `POST /sdk/telemetry` (`{type, sdk_id, message, ...}`). It is unauthenticated in the
  first-party implementation and can be disabled by the host game.
