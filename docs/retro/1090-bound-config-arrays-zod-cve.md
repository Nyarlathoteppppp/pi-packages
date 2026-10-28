---
issue: 1090
issue_title: "upgrade zod dependency - versions below 4.6.6 are affected by CVE-2023-54404"
---

# Retro: #1090 — upgrade zod dependency - versions below 4.6.6 are affected by CVE-2023-54404

## Stage: Planning (2026-10-28T03:28:52Z)

### Session summary

I planned the change for a third-party report asking for zod 4.6.6 or later.
The plan is cross-package: it raises `zod` to `^4.6.7` in `pi-permission-system` and `pi-permission-model-judge`, and bounds all six config arrays at 1,000 entries with `.max()`.
I also filed [#1092] for `pi-permission-model-judge` reading an untrusted project's config.

### Observations

- **Reading zod's source changed the direction.**
  The compiled zod 4.6.7 `$ZodArray` showed the upstream fix only pre-checks `.max()` / `.length()` / `.size()`.
  An unbounded array still records one issue per element, so the reporter's "may not be completely fixed" comment was right.
  On the operator's direction ("propose a bound for the array"), the plan bounds every array, not just the floor.
- **Exposure.**
  `pi-permission-system` parses only operator-owned configs (the project scope is trust-gated).
  `pi-permission-model-judge` parses an untrusted project's config, which is the one attacker-controlled zod input; that trust gap is [#1092].
- **Operator decisions.**
  - All six arrays, one shared cap of 1,000.
  - Breaking (`fix(<pkg>)!:`), chosen over non-breaking hardening.
  - Floor at the latest published version, `^4.6.7`, not `^4.6.6`.
  - File [#1092].
- **Measured with the published tarballs:**
  - 1,001 invalid entries gives 1,002 issues on 4.4.3 and 1 on 4.6.7 with `.max(1000)`.
  - 1,000,000 invalid entries on an unbounded array gives 1,000,000 issues in 672 ms.
  - The spike worktree passed every package's tests on 4.6.7, and both `gen:schema` outputs were unchanged.
- **The Tidy-First assessor caught a vacuous assertion.**
  At a union site (`declaredEffectsSchema`) the top-level issue count is 1 under every zod version.
  The plan's tests flatten `invalid_union` `errors` and assert exactly one `too_big` leaf and no element-indexed leaf (measured: 1,003 leaves on 4.4.3, 2 on 4.6.7).
- **Rejected options.**
  - `pnpm dedupe`: it churned 689 lockfile lines in the spike.
  - Capping `z.record` maps: zod 4.6.7 has no pre-walk cap for records (`capSize` covers only array, map, and set), and every record is read from trusted input only.
  - A shared `boundedArray` helper.
- Roadmap-fit: neither package has an open improvement phase, so [#1092] got no disposition.

[#1092]: https://github.com/gotgenes/pi-packages/issues/1092
