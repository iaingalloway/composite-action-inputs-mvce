# MCVE: Node action post hook sees wrong input when called through nested composite actions

Minimal reproduction of a GHA runner bug where a node action's post step receives
the wrong `INPUT_*` environment — the input of a parent composite action rather than
the input the action itself was called with.

## Tracked upstream

- [actions/runner#3514](https://github.com/actions/runner/issues/3514) — Wrong environment passed to node post when called by composite called by composite action
- [actions/runner#2030](https://github.com/actions/runner/issues/2030) — Composite: Nested actions post steps have the wrong context

## Structure

```text
workflow
└── outer-composite        (input: value=foo)
    └── [sets value=bar]
    └── inner-composite    (input: value=bar)
        └── handle-value   (node action, input: value=bar)
            ├── main.js
            └── post.js
```

## Observed behaviour

```text
[main] value=bar   ← correct: the value computed in outer-composite
[post] value=foo   ← wrong:   the value the workflow passed to outer-composite
```

## Expected behaviour

```text
[main] value=bar
[post] value=bar
```

The post step should see the same `INPUT_VALUE` that main received. Instead, the
runner re-evaluates the input expression chain at post time in a degraded context,
and falls back to the nearest ancestor composite's input — which is the workflow-level
`value: foo`.

## Workaround

Persist inputs explicitly in main using `saveState`, then read from `STATE_*` in post
rather than relying on `INPUT_*`. See
[github/codeql-action#2557](https://github.com/github/codeql-action/pull/2557) for a
real-world example of this fix.

## Real-world impact

This bug affects any node action that:

1. Has a post step that reads its own inputs, **and**
2. Is called (directly or transitively) through a composite action that itself receives
   those values via an input

Known affected actions include `github/codeql-action` (worked around in v3.30.0) and
`devcontainers/ci` (push in post step uses wrong image tags).

A real-world example can be seen in [this workflow](https://github.com/iaingalloway/devcontainers/blob/3878f223925c5341df8d9b7aff2c57078b063fbc/.github/workflows/build-devcontainers.yaml) which uses devcontainers/ci to build and push devcontainer images. Images are pushed during the post step, and the action receives the wrong input.
