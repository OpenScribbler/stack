# OpenScribbler stack

Four projects form one supply chain for AI agent content: publish once, run in
any tool, verify where it came from, and stay correct as tools change. This
repository holds the decisions that bind more than one of them. A decision that
touches only one project lives in that project.

| Project | Layer | Owns |
|---|---|---|
| [ACIF](https://github.com/OpenScribbler/agent-content-interchange-format) | Format | What a content item is, its identity and hash, and how it renders into each tool |
| [capmon](https://github.com/OpenScribbler/capmon) | Ground truth | What each tool accepts and where it reads it from, tracked continuously |
| [MOAT](https://github.com/OpenScribbler/moat) | Trust | Signatures, provenance, trust tiers and revocation for content |
| [syllago](https://github.com/OpenScribbler/syllago) | Client | Behavior only: add, verify, render, apply, sync and roll back |

Syllago v2 is the first implementation of the whole stack. Its first user is
one person who keeps their AI tool configuration in sync across their own
machines.

## Concept ownership

Each shared concept has one owner. The others cite the owner and define
nothing of their own.

| Concept | Owner | Consequence |
|---|---|---|
| Content identity and hash | ACIF (`body_hash`, `metadata_hash`) | MOAT signs ACIF's attestation input digest over both hashes rather than its own directory hash, and ACIF registries publish that digest as the optional `attestation_hash` |
| Content-type names | ACIF | Syllago uses ACIF's kind tokens. capmon and MOAT map their own type names to them |
| Hook event vocabulary | ACIF | Syllago's own hooks spec (HIF) retires |
| Install-path schema | ACIF (install-targets) | ACIF's A.2 table stays normative until it is reconciled with capmon's rows, then becomes a conformance fixture |
| Install-path data | capmon | The feed publishes structured paths. Syllago vendors no path tables |
| Provider identifiers and capabilities | capmon | Syllago's `provider-formats` YAML retires |
| Content signatures and trust tiers | MOAT | Covers third-party content and the user's own content alike |
| Build-artifact provenance | GitHub SLSA | The capability feed keeps SLSA attestation. The client shares one Sigstore verifier between SLSA and MOAT |
| Settings (permissions, model, env) | Syllago, for now | A portable subset goes to ACIF once mappings across the launch providers exist |

## Seam contracts

**ACIF to MOAT.** One signed input. MOAT signs ACIF's attestation input
digest, which binds `body_hash` and `metadata_hash`, so it covers the
frontmatter grants that `body_hash` strips. MOAT hashes the fetched bytes and
applies none of ACIF's canonicalization rewrites, though `body_hash` still
strips frontmatter and normalizes line endings. The same skill still hashes
the same whichever tool it came from, because syllago signs the ACIF-canonical
Library copy. The digest is also the route by which MOAT can later attest
hooks and MCP configuration. MOAT limits the ACIF method to skills, rules,
agents and commands until then.

**ACIF to capmon.** ACIF defines the install-targets schema. capmon fills it
from the tools it watches and publishes it in the feed. capmon's prose
`mechanism` fields stop being the only record of paths. capmon's `conversion`
vocabulary converges on ACIF render-back's fidelity classes.

**MOAT to authors.** MOAT gains an author-key publisher profile. An SSH key,
an `sk-` security-key type where the author has one, signs. The signature sits
beside the registry's per-item Rekor entry as the second attestation of the
existing Dual-Attested tier. The author-key Rekor entry identifies the author
only by a public key with no comment, never by an email address. The
push-token guarantee comes from syllago's own policy on top of MOAT's profile:
syllago refuses any item from the sync repository that isn't Dual-Attested
under the author key from the `source_uri` pinned at bootstrap. A stolen push
token then can't produce content that a machine trusting the author's keys
installs without asking.

**capmon to syllago.** Syllago consumes the feed and verifies its SLSA
attestation fail-closed. Each provider adapter carries golden fixtures pinned
to a provider version, so a feed change regenerates them and the diff shows
what syllago would write differently.

## Syllago v2

- Rust, designed greenfield. The Go v0.15 code is a reference for provider edge
  cases and failure modes, not a translation source.
- Clean break from v0.x with no importer. Same name and repository, built
  beside the Go code and swapped at v1.0.
- Linux, macOS and native Windows from the first commit, all three in CI.
- CLI only, with `--json`. A TUI comes after the model is stable.
- ACIF is a separate crate in the Cargo workspace. It runs the ACIF conformance
  suite in CI and is published once it passes.

### Launch providers

Claude Code, Codex CLI, Antigravity CLI, OpenCode, pi and Cursor. Providers sit
in one of two tiers. Stable providers have verified paths and golden fixtures.
Experimental providers warn on apply until their paths are confirmed against a
real install. Antigravity starts experimental.

### Content

ACIF's six types (hook, skill, rule, command, agent, MCP configuration) plus
provider settings. Settings are owned per key: syllago writes only the keys it
owns and never touches the rest of the file, because the tools rewrite their
own settings files.

Where every targeted provider reads a shared location (`AGENTS.md`,
`.agents/skills`), the item is written once there. Each adapter declares what
its provider cross-reads, so no provider loads an item twice. Content a
provider can't take is skipped and reported. `--strict` turns skips into
failures.

### Loadouts

A loadout is a named set of content targeted at one or more providers. Several
can be active at once. When two would write the same file or the same settings
key, the apply is refused and names both. User-wide loadouts live in the sync
repository. A project loadout is a file committed in that project's own
repository.

A loadout in the sync repository is signed with the author key, like the items
it names. The signature covers the providers it targets, the
`attestation_hash` of each item, and a sequence number that rises with every
edit. Each machine records the last sequence it accepted for each loadout.
Sync refuses a loadout whose signature fails, whose signer isn't in the
trusted allowed-signers list, or whose manifest changed without a sequence
greater than the one recorded, and names the loadout and the reason.
Sync applies only the loadouts a machine has activated. The user activates a
loadout at bootstrap or with an explicit apply, after seeing its name,
sequence, providers and pinned items, and `syllago remove` deactivates it.
For an activated loadout, a stolen push token can't point it at other
providers or at an older signed version of an item, or roll it back past the
version the machine last accepted. Two limits remain. First activation rests
on the user's confirmation, which can't tell an older signed version from
the current one. And with no freshness signal, a token holder can hold a
machine at a stale version newer than its record until it syncs from a
current repository. The cost is that every machine that edits a loadout needs
the author key, and an `sk-` key asks for a touch on each edit.

### Apply

Apply is all-or-nothing across every provider in one run: snapshot, apply
everything, and roll the whole run back on any failure. Drift detection covers
only what syllago owns, meaning whole files it placed and keys it wrote. Drift
in owned content is refused and reported, and the user chooses `capture`
(take the edit into the library) or `--overwrite`.

### Sync

The source of truth is a GitHub repository the user owns, because MOAT's
Registry Action, which signs it as a registry, is a GitHub Actions workflow. Sync
runs one way: the library generates provider files, and hand edits come back
only through `capture`. Every machine runs syllago and pulls. `syllago sync`
pulls, verifies and applies, and syllago can install an optional systemd or
launchd timer for it. There is no daemon. `syllago bootstrap <repo>` sets up
an ephemeral environment in one command.

Every machine verifies MOAT author signatures before it applies a skill, rule,
agent or command, the content types MOAT signs.

### Secrets

The repository holds references only. Syllago renders each provider's native
environment reference (`${VAR}`, `${env:VAR}`, `{env:VAR}`, or Codex's
`*_env_var` keys), so the secret value never reaches disk. A provider without
interpolation gets the value resolved at apply time, in a file with 0600
permissions that is never captured back into the repository.

### Out of scope at launch

Third-party registries, ACIF's registry records, packs and projections,
publishing to registries, sandbox, telemetry, the rule splitter, the content
analyzer and `share`. They return with the team track. Two parts stay in. The
user's own sync repository is a MOAT Self-Publisher registry that publishes a
signed manifest. From ACIF's publisher and registry layers, the skeleton takes
`publisher_section` observation, `metadata_hash` and the Registry Specification
§6 digest. Release signing (cosign, checksums, SBOM) ships from day one.

## Delivery

Work proceeds in vertical slices. Each slice is one thing a user can do, cut
through every project it needs. A spec change lands through its project's
change process before the code that depends on it.

### Walking skeleton

One skill and one rule travel the whole path:

1. Authored on one machine and canonicalized as ACIF.
2. Signed with the MOAT author-key profile.
3. Rendered for Claude Code and Codex by adapters whose fixtures come from the
   capmon feed.
4. Pushed to the sync repository.
5. Verified and applied by a second machine set up with `syllago bootstrap`.
6. Rolled back cleanly.

One end-to-end test in syllago's CI covers this path. It pins ACIF's
conformance suite, capmon's feed and MOAT's test vectors to fixed versions, so
a change in any project that breaks the stack fails there.

### Spec work the skeleton needs

All of it has landed except a MOAT clarification that an Author Key may sign
messages outside the profile, which signed loadouts need.
[`specs/walking-skeleton.md`](specs/walking-skeleton.md) records what each
change settled.

- ACIF: settle the two install conflicts the apply engine depends on. Hook
  contributions are keyed by item, not by event wiring, and a blocking hook is
  refused when its render would fail open. The other seven are settled as
  slices reach them.
- ACIF: define `attestation_hash` as the digest over `body_hash` and
  `metadata_hash`.
- MOAT: sign that digest for ACIF items, add the author-key profile, and
  settle the signer subject format (the spec and the npm sub-spec disagreed).
- capmon: publish structured install paths against the ACIF schema.

### Later slices

Hooks and MCP configuration, settings, the remaining launch providers, full
capmon automation, the team track (registries and shared sync repositories),
and research into a hosted sync service. MOAT attestation for hooks and MCP
configuration, and a decision on how synced settings are protected, are both
needed before v2 syncs those types.

### Cleanup carried by the first slices

- Syllago's vendored `install-entry-points.yaml` is stale against ACIF (its Cursor
  hook rows point at `settings.json`).
- MOAT's spec says it has no adopters. Syllago implements v0.6, and the spec is
  at v0.8.
- ACIF's conformance status table is stale.
- Syllago's 15 `provider-formats` files differ from capmon's.
