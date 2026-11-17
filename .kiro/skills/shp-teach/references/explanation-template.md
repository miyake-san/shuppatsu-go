# Explanation template

Use one block per decision. Keep each block short; link instead of quoting at length.

```markdown
### <n>. <decision in a few words>

**Where:** `path/to/file.go:<line>` (`FuncName`)
**Class:** Language rule | Idiom | Pattern | Platform guidance | Project convention | Trade-off

- **What:** what the code does here, in one or two sentences.
- **Why:** the reason, with a source link. If it is a pattern, name it and link the `shp-patterns-*` skill.
- **Alternative:** the most plausible other way to write it, and what it would cost or gain.
- **Wrong choice when:** the conditions under which this decision should be reversed.
```

## Closing block

```markdown
### Check your understanding
1. <question that requires applying the idea, not recalling it>
2. <optional: "what breaks if we change X to Y?">

**Go deeper:** <one or two links or `go doc` commands worth running>
```

## Confidence labels

Append one of these when relevant:

- *(verified: <source>)* when you read the source in this session.
- *(general knowledge, not verified against current docs)* when web access was unavailable.
- *(project choice, no external source)* when the reason is a local convention.
