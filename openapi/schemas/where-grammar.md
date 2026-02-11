# where expression

Purpose: filter list endpoints and scope webhook subscriptions.

Guidance:
- Use field names that exist on the target object
- Operators: =, !=, AND, OR, parentheses
- Values should be quoted strings when ambiguous
- Validate at request time (REST) and creation time (webhooks)
