---
name: PlayerWON rewarded-video opportunity
description: Authorize with PlayerWON, request a rewarded-video ad opportunity for a player, play it, and report the playback lifecycle so the player's reward is granted.
api: PlayerWON API
base_url: https://game.simulmedia-apis.com
method: generated
source: https://github.com/simulmedia/playerwon-sdk
operations:
  - POST /token
  - POST /session
  - POST /opportunity
  - PUT /start/{receipt}
  - PUT /progress/{receipt}
  - PUT /complete/{receipt}
  - PUT /abort/{receipt}
---

# PlayerWON rewarded-video opportunity

Use this skill to serve a PlayerWON rewarded-video ad inside a free-to-play game and
grant the player their reward. All calls go to `https://game.simulmedia-apis.com`.

## Auth (do this on the game server, never in the client)
1. `POST /token` with `Content-Type: application/x-www-form-urlencoded` and body
   `client_id`, `client_secret`, `grant_type=client_credentials`, `tid=<game title id>`
   (optionally `idfa`). Read `access_token` from the JSON response.
2. Hand the token to the game client, which sends `Authorization: Bearer <access_token>`
   on every subsequent call. Never embed `client_secret` in the shipped client.

## Serve an opportunity
3. `POST /session` (Bearer) to open a session for the title.
4. `POST /opportunity` (Bearer) with a JSON `ClientDetails` body: `country`, `plat`,
   `lang`, `idfa`, plus the consent flags `coppa`, `gdpr`, `gdpr_consent`, `lt`. Set
   these correctly — they gate targeting and compliance. The response is an
   `Opportunity` carrying a `Receipt`, `CreativeURL`/`VideoURL`, and `Length`.
5. Begin playback and `PUT /start/{receipt}` (Bearer).
6. While the video plays, `PUT /progress/{receipt}?p=<percent>` (Bearer) to report progress.
7. On finish, `PUT /complete/{receipt}` (Bearer) — this is what grants the reward.
8. If the player cancels early, `PUT /abort/{receipt}?reason=<Cancel|Other>&t=<seconds>`
   (Bearer) instead of completing.

## Rules
- The `Receipt` from step 4 is the key for steps 5–8; carry it through the whole flow.
- Treat any non-2xx response as a failure and stop the flow (no reward on failure).
- Re-fetch a token when it expires (`expires_in` seconds from the /token response).
