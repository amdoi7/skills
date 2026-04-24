# Philosophy: Good Taste & Local Shape

> "Sometimes you can see a problem in a different way and rewrite it so that the special case goes away."

Use this reference when the code is locally ugly: too many branches, too much nesting, awkward special cases, or control flow that hides the real rule.

## Core Idea

Good taste is not decorative style. It is removing special handling by choosing a better representation.

## Rules

- Eliminate special cases instead of piling branches around them
- Prefer early returns over nested pyramids
- Let data shape simplify control flow
- If a function is hard to explain, the shape is probably wrong
- If a name is hard to choose, the responsibility is probably mixed

## Example: Remove the Special Case

```go
// awkward
func removeNode(head *Node, target *Node) *Node {
    if head == target {
        return head.Next
    }

    prev := head
    for prev.Next != nil {
        if prev.Next == target {
            prev.Next = prev.Next.Next
            return head
        }
        prev = prev.Next
    }
    return head
}
```

```go
// cleaner
func removeNode(head **Node, target *Node) {
    for *head != target {
        head = &(*head).Next
    }
    *head = target.Next
}
```

The second version does not "handle head removal" separately. It removes the distinction.

## Example: Collapse Nesting

```go
// awkward
func process(data *Data) error {
    if data != nil {
        if data.IsValid() {
            if err := save(data); err != nil {
                return err
            }
            return nil
        }
        return ErrInvalidData
    }
    return ErrNilData
}
```

```go
// clearer
func process(data *Data) error {
    if data == nil {
        return ErrNilData
    }
    if !data.IsValid() {
        return ErrInvalidData
    }
    return save(data)
}
```

## Local Smells

| Smell | Usually means |
| --- | --- |
| more than one "special path" in one function | representation is wrong |
| 3+ nested condition levels | conditions should be inverted or split |
| variable names like `tmp2`, `handler2`, `dataX` | the code lost its concepts |
| comments explaining control flow line by line | the flow is too awkward |

## Review Questions

- What special case is forcing this branch
- Can a better data shape remove that distinction
- Can an early return flatten this function
- Is the real rule obvious from the function shape
- Would a new helper hide complexity or just move it elsewhere
