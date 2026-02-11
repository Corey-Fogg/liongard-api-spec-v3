# Webhooks

Webhooks are first-class for vendors.

- Subscriptions: `/v3/webhooks/subscriptions`
- Each subscription declares one or more object scopes:
  - `object`: alerts/assets/environments/identities/inspectors
  - `events`: created/updated/deleted
  - optional `where`
  - optional `expand`
  - `delivery.mode`: full | diff

Delivery:
- full: sends full object payload (respecting expand)
- diff: sends changed fields and identifiers; optional snapshot via `includeSnapshot`
