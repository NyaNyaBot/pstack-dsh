# Local patches on top of upstream pstack-dsh

This fork exists for one reason: **DSH 0.2.1-rc.2 / rc.3 do not expose
`ctx.subagents.start(provider, request)`**, which pstack 0.2.5 calls from `pstack_spawn`.
Those builds expose `ctx.subagents.startActivation(spec)` instead — a one-shot child whose
result is delivered to the parent as a message. Without a shim, `pstack_spawn` fails with
`subagents.start is not a function`.

## What changed

| File | Change |
| --- | --- |
| `src/tools.ts` | adds `startPstackRun()` and routes the spawn tool through it |
| `lib/index.js` | same change in the built output — **this is what actually runs** |
| `package.json` | version `0.2.5-dshcompat.1` |

`startPstackRun()` prefers the official path and degrades only when it has to:

1. `subagents.start` exists → unchanged behaviour (future DSH versions take this branch).
2. else `subagents.startActivation` exists → starts a detached activation with
   `delivery: "parent"`, returns its `childId` as the run id.
3. neither → throws a clear error.

Background spawns are unaffected. A **foreground** spawn on a shimmed build returns
`kind: "foreground-detached"` with a `note`, because that build has no blocking run handle;
the child keeps running and reports back as a message.

## Why `lib/` is committed

`package.json` has no `prepare` script, so pnpm installing `github:...#tag` uses the committed
`lib/` as-is and never builds. Committing the patched `lib/index.js` is therefore required for
installs to pick the shim up.

If you want `lib/` to be a real build product again:

```sh
pnpm install
pnpm check          # typecheck + tests + tsdown build
git diff --stat     # lib/ should change to the freshly built output
```

## Install

```sh
dsh plugin --profile <name> add github:NyaNyaBot/pstack-dsh#v0.2.5-dshcompat.1
```

## Upgrading from upstream

```sh
git fetch upstream --tags
git checkout -b dsh-0.2.1-compat-vX.Y.Z vX.Y.Z
# re-apply the two hunks: src/tools.ts (startPstackRun + spawn branch) and rebuild lib
git commit -am "feat: carry DSH startActivation compat shim onto vX.Y.Z"
git tag vX.Y.Z-dshcompat.1 && git push origin dsh-0.2.1-compat-vX.Y.Z --tags
```

## Known limitations

- Role personas (`agents/poteto-agent.md`, …) are not injected as a system prompt on the
  `startActivation` path, so subagents behave more generically than on a build with `start()`.
- `src/tools.ts` and `lib/index.js` are kept in sync by hand unless `pnpm check` was run.
