# Decisions

- Inspectors are environment-scoped under `/v3/environments/{environmentId}/inspectors`.
- Dataprints are not global; they only exist under an environment + inspector.
- Ingestion is modeled as dataprints + dataprints/config.
- Expand is dynamic per object (and sometimes per type) and is discovered via meta endpoints.
- Filtering uses a `where` expression for REST and webhooks.
- POST endpoints use 201.

(Keep this file updated when the contract changes.)
