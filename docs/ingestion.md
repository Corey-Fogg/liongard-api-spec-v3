# Dataprint ingestion (Push)

Ingestion is modeled as creating dataprints under an inspector.

Flow:
1) Configure mapping:
   - `PUT /v3/environments/{environmentId}/inspectors/{inspectorId}/dataprints/config`
2) Push data:
   - `POST /v3/environments/{environmentId}/inspectors/{inspectorId}/dataprints`

Config stores `metadata` (JSONPath mapping) and optional `views`.
Push accepts `data` (plus optional `configVersion`) and returns an ingestion result.
