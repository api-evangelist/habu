---
name: habu-provision-a-data-connection
description: Land a dataset into the Habu Clean Room platform by creating a data connection from a data source and mapping its fields to identifier types.
api: Habu Clean Room API
base_url: https://api.habu.com/v1/
docs: https://developers.liveramp.com/clean-room-api
generated: '2026-08-12'
method: generated
source: openapi/habu-clean-room-api-openapi.yml, data-model/habu-data-model.yml, conventions/habu-conventions.yml
operations:
  - getAllDataSources
  - getAllDataTypes
  - getAllDataSourceParameters
  - getAllCredentialSources
  - getAllOrganizationCredentials
  - createDataConnection
  - getDataConnectionById
  - getAllFieldConfigurations
  - getAllIdentifierTypes
  - mapFieldConfigurations
  - retryDataConnectionJob
---

# Provision a data connection

A clean room can only join data that has been landed as a **data connection** and mapped.
This is the setup flow that precedes everything else.

## Hard constraint: credentials are console-only

You **cannot create a credential through this API**. LiveRamp deliberately does not expose
credential creation externally "to prevent potential security issues with transmitting raw
credential values." Configure the credential in the Clean Room console first, then read it
back here with `getAllOrganizationCredentials`. An agent that tries to create one is
solving the wrong problem.

## Steps

1. **Authenticate** — see `habu-run-a-cleanroom-question` step 2. Same client-credentials
   flow, same 12-hour token, same 2-tokens-per-day ceiling.

2. **Pick the source.** `getAllDataSources` (`GET /data-sources`) lists the source systems
   available to your customer integration. Take the `dataSourceId`.

3. **Pick the data type.** `getAllDataTypes`
   (`GET /data-sources/{dataSourceId}/data-types`).

4. **Read the required parameters.** `getAllDataSourceParameters`
   (`GET /data-sources/{dataSourceId}/data-types/{dataTypeId}/data-source-parameters`).
   These are the fields the source demands — do not guess them.

5. **Resolve the credential.** `getAllCredentialSources` tells you which credential kinds
   the source accepts; `getAllOrganizationCredentials` returns the credentials already
   configured in the console. Take the `organizationCredentialId`.

6. **Create the connection.** `createDataConnection` (`POST /data-connections`) with the
   source, data type, parameters and credential. **No idempotency key exists** — if the
   call times out, list with `getAllDataConnections` before retrying, or you will create a
   duplicate connection.

7. **Wait, then read the discovered fields.** `getDataConnectionById` shows the job status;
   `getAllFieldConfigurations`
   (`GET /data-connections/{dataConnectionId}/field-configurations`) returns the columns
   the platform found.

8. **Map the fields.** `getAllIdentifierTypes` lists the identifier types you may assign;
   `mapFieldConfigurations`
   (`POST /data-connections/{dataConnectionId}/field-configurations`) commits the mapping.
   This is the step that makes the data joinable in a clean room — an unmapped connection
   is inert.

9. **If the job failed**, `retryDataConnectionJob` re-runs it rather than recreating the
   connection.

## Notes

- Only 11 of the 151 operations on this API accept `limit`/`offset`. Most list calls
  return everything; do not assume a page envelope, and do not look for `has_more`.
- The error envelope and the `x-request-id` correlation header are described in
  `conventions/habu-conventions.yml`.
