# Philosophy: Complexity Management & Deep Modules

> "Complexity is anything related to the structure of a software system that makes it hard to understand and modify."

Use this reference when the problem is not local code shape but structural drag: too much coordination, repeated knowledge, shallow abstractions, or costly change.

## The Three Main Symptoms

| Symptom | What it looks like |
| --- | --- |
| change amplification | one simple change touches many files or layers |
| cognitive load | you must load too much context before editing anything |
| unknown unknowns | you cannot tell what else might break |

## Design Rules

- Prefer deep modules: simple interface, substantial hidden complexity
- Keep implementation detail out of public interfaces
- Put common paths first; uncommon knobs should not dominate the API
- Pull complexity downward into the owning module
- Design twice before committing to a structural split

## Shallow vs Deep

```text
shallow module                 deep module
┌─────────────────────┐        ┌──────────────┐
│ many knobs          │        │ small API    │
│ many call steps     │        ├──────────────┤
│ little hidden power │        │ hidden rules │
└─────────────────────┘        │ and mechanics│
                               └──────────────┘
```

## Pass-Through Layer Smell

```go
// shallow pass-through
func (s *UserService) Get(ctx context.Context, id UserID) (*User, error) {
    return s.repo.Get(ctx, id)
}
```

A layer that mostly renames the next call is not an abstraction. It is extra coordination cost.

## Better Direction

```go
// deeper module
func (s *UserService) GetProfile(ctx context.Context, id UserID) (*Profile, error) {
    user, err := s.userRepo.Get(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err)
    }

    preferences, err := s.prefRepo.Get(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get preferences: %w", err)
    }

    return buildProfile(user, preferences), nil
}
```

Here the module owns a real decision and hides coordination.

## Information Hiding Test

If multiple packages must know the same storage rule, retry rule, DTO shape, or lifecycle sequence, information is leaking.

The fix is usually not another layer by name. The fix is moving ownership to one module.

## Review Questions

- Does this module hide complexity or export it
- Does the interface make the common path simple
- Is the same rule duplicated across packages
- Would deleting this layer make the design clearer
- Is the proposed split based on ownership or only on execution order
