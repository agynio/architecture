# Changes

Pending deltas between the desired state described in `/product` or `/architecture` and current reality.

## Lifecycle

1. A `/product` or `/architecture` document is created or updated to describe a new desired state.
2. A change file is created here recording the delta.
3. When reality matches the desired state, the change file is deleted.

Git history preserves the full record of past changes.

## File Naming

```
YYYY-MM-DD-<slug>.md
```

Date is when the desired state was defined, not when work starts or ends.

## File Structure

```markdown
# <Title>

## Target

Link to the `/product` or `/architecture` document this change relates to.

## Delta

What specifically differs between current state and desired state.

## Acceptance Signal

How to know the delta is closed.

## Notes

Optional context, related issues, dependencies.
```

## Rules

- A change file must reference a `/product` or `/architecture` document. If no spec exists, write the spec first.
- Change files are not modified to track progress. They either exist (delta open) or are deleted (delta closed).
- No implementation details. Changes describe what is missing, not how to build it.
