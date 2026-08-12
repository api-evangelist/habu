---
name: habu-set-up-a-cleanroom-collaboration
description: Stand up a clean room on the Habu Clean Room API — create it, provision datasets, invite a partner, attach a question and set its permissions.
api: Habu Clean Room API
base_url: https://api.habu.com/v1/
docs: https://developers.liveramp.com/clean-room-api
generated: '2026-08-12'
method: generated
source: openapi/habu-clean-room-api-openapi.yml, data-model/habu-data-model.yml
operations:
  - getCleanroomTypes
  - createCleanroom
  - configureCleanroomDataset
  - getCleanroomDatasets
  - addCleanroomPartner
  - getAllCleanroomPartners
  - listPartnerInvitationsForInviter
  - getAllOrganizationUsers
  - addCleanroomUser
  - getCleanroomRoles
  - createQuestion
  - addCleanroomQuestion
  - configureCleanroomQuestionDatasets
  - configureCleanRoomQuestionPermissions
---

# Set up a clean room collaboration

The end-to-end setup: a clean room, the data in it, the partner you are collaborating
with, and the question they are allowed to run.

## Steps

1. **Authenticate** (client credentials → bearer, 12h token, 2 tokens/day).

2. **Choose the clean room type.** `getCleanroomTypes`
   (`GET /cleanrooms/cleanroom-types`) — the type fixes the underlying technology and
   cannot be swapped later.

3. **Create the clean room.** `createCleanroom` (`POST /cleanrooms`). Keep the returned
   `cleanroomId`; nearly every subsequent path takes it (70 of the 151 operations do).

4. **Provision datasets into it.** `configureCleanroomDataset`
   (`POST /cleanrooms/{cleanroomId}/datasets/configure`) binds an already-mapped data
   connection into the clean room. Verify with `getCleanroomDatasets`. If the connection
   is not mapped yet, run `habu-provision-a-data-connection` first.

5. **Invite the partner.** `addCleanroomPartner`
   (`POST /cleanrooms/{cleanroomId}/partners`). Track the invitation with
   `listPartnerInvitationsForInviter`, and confirm acceptance with
   `getAllCleanroomPartners`. Use `cancelPartnerInvitationById` or
   `cancelPartnerInvitationByEmail` to withdraw one.

6. **Add your own users.** `getAllOrganizationUsers` lists who is eligible;
   `getCleanroomRoles` lists the roles; `addCleanroomUser`
   (`POST /cleanrooms/{cleanroomId}/users`) grants access with a role. Authorization on
   this API is **by clean room role, not by OAuth scope** — the client-credentials flow
   declares an empty `scopes` object, so the role is the only access control that exists.

7. **Author the question.** `createQuestion` (`POST /question`) creates it in the
   organization library; `addCleanroomQuestion`
   (`POST /cleanrooms/{cleanroomId}/cleanroom-questions`) attaches it to this clean room.

8. **Bind the datasets to the question's macros.** `configureCleanroomQuestionDatasets`
   (`POST /cleanroom-questions/{cleanroomQuestionId}/datasets`). A question whose macros
   are unbound cannot run.

9. **Set who may run and see it.** `configureCleanRoomQuestionPermissions`. Optionally
   share results with the partner via the clean room question result-share operations.

The collaboration is now ready — hand off to `habu-run-a-cleanroom-question`.

## Watch out

- **No idempotency key on any of these POSTs.** Every creation step here can duplicate on
  retry. Read before you write.
- **No deprecation policy and no `Sunset`/`Deprecation` headers.** The only stability
  signal in the contract is the `Internal` tag, which marks 4 operations as outside the
  supported external contract. Avoid those.
- **The changelog is dormant** (one post, January 2025) while the OpenAPI is actively
  maintained, so contract changes will not be announced to you. Re-fetch
  https://storage.googleapis.com/lr-tech-docs-resources/Files/clean-room/api/liveramp-clean-room-api-specification.yml
  and diff it rather than relying on release notes.
