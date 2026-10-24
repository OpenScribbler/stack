# Spec: walking skeleton

The first vertical slice through the whole stack. One skill and one rule travel
from authoring on one machine to a verified apply on a second machine, and back
out again by rollback. Every seam named in the stack README is crossed once.

Vocabulary follows each project's lexicon: syllago's for the client (Library,
Content Item, Provider, Loadout, Snapshot, Install), and MOAT's for trust
(Publisher, Self-Publisher, End User, Conforming Client, Signing Identity, Dual
Attestation). Where v2 changes a term's meaning, the change is stated below.

## Problem Statement

I keep AI coding tool configuration (rules, skills, settings) on several
machines and across several tools. Today I copy it by hand or with dotfile
tooling that knows nothing about the tools' formats. Each tool stores the same
idea in a different file, at a different path, in a format that changes every
few weeks. When I change a rule on one machine, the other machines and the
other tools don't see it. When something goes wrong after an apply, I can't
cleanly undo it. And anything that syncs config to every machine I own is also
a way to run code on every machine I own: if someone gets my GitHub token,
they can push a hook that runs everywhere.

The projects meant to fix this — ACIF, capmon, MOAT and syllago — were built
separately and now disagree with each other about hashing, install paths and
signing. No single path runs through all four, so nothing proves they work
together.

## Solution

I author a skill and a rule once, on my desktop. Syllago stores them in my
Library in ACIF form, signs them with my own hardware-backed key, and pushes
them to a git repository I own. On a second machine I run one bootstrap command
pointing at that repository. Syllago clones it, verifies that every Content
Item was signed by my key, and installs the skill and the rule into Claude Code
and Codex in each tool's native format, at the paths capmon says each tool
reads. If anything in the apply fails, nothing is left half-installed. If I
want it gone, one command restores both tools to exactly what they were.

A content change pushed by anyone without my key is refused on every machine,
with an error that says why.

## User Stories

1. As an End User, I want to add an existing Claude Code skill to my Library, so that I can manage it from one place.
2. As an End User, I want to add an existing rule file to my Library, so that it can be rendered for every tool I use.
3. As an End User, I want added content stored in ACIF form, so that the same item has the same identity whichever tool it came from.
4. As an End User, I want each Content Item to carry its ACIF `body_hash`, so that any change to it is detectable.
5. As an End User, I want adding the same skill twice from two different tools to yield one Library item, so that I don't get duplicates.
6. As an End User, I want `syllago init <repo-url>` to set up my Library as a git repository I own, so that my config has history and a remote.
7. As an End User, I want to create a hardware-backed signing key once, so that my machines can tell my content apart from anyone else's.
8. As an End User, I want syllago to refuse to sign with a key that isn't hardware-backed unless I pass an explicit flag, so that I don't weaken my own setup by accident.
9. As a Self-Publisher, I want every Content Item I commit to be signed with my author key, so that a stolen push token can't produce trusted content.
10. As a Self-Publisher, I want the signature to cover the ACIF `body_hash`, so that the signature and the identity are the same thing.
11. As a Self-Publisher, I want the transparency-log entry to carry my key fingerprint rather than my email address, so that signing doesn't publish my identity.
12. As a Self-Publisher, I want to declare which keys are allowed to sign for my repository, so that adding a second hardware key later is a config change rather than a rewrite.
13. As an End User, I want to create a loadout that names my skill and my rule and targets Claude Code and Codex, so that one command configures both tools.
14. As an End User, I want `syllago apply --preview` to show me every file and settings key it would write, so that I can check before anything changes.
15. As an End User, I want `syllago apply` to write the skill and the rule into Claude Code in its native format, so that Claude Code uses them.
16. As an End User, I want the same apply to write them into Codex in its native format, so that Codex uses them too.
17. As an End User, I want the rule written to `AGENTS.md` once when every targeted tool reads it, so that no tool loads it twice.
18. As an End User, I want syllago to know which tools also read another tool's directories, so that a skill isn't loaded twice by a tool that reads both `.claude/skills` and `.agents/skills`.
19. As an End User, I want install paths to come from the capmon feed, so that a tool moving its config directory doesn't silently break my setup.
20. As an End User, I want the capmon feed verified before syllago trusts it, so that a tampered feed can't redirect where my content is written.
21. As an End User, I want an apply to snapshot every file it will touch before writing anything, so that a failure can be undone.
22. As an End User, I want a failed apply to roll back everything it did across both tools, so that I'm never left with Claude Code configured and Codex not.
23. As an End User, I want a rollback that can't complete to keep its snapshot and tell me exactly what to restore by hand, so that I never lose the original files.
24. As an End User, I want `syllago remove <loadout>` to restore both tools to their pre-apply state, so that trying something is safe.
25. As an End User, I want remove to delete only what the apply placed, so that files I created myself are never touched.
26. As an End User, I want syllago to record what it owns after each apply, so that drift detection and remove know their scope.
27. As an End User, I want `syllago status` to report drift when I've hand-edited a file syllago placed, so that I know the tool and the Library disagree.
28. As an End User, I want an apply over a drifted file to stop and name the file, so that my edit is never lost silently.
29. As an End User, I want `syllago capture` to bring a hand edit back into my Library, so that a fix I made in place becomes the new truth.
30. As an End User, I want `--overwrite` to replace a drifted file deliberately, so that I can choose the Library's version when I mean to.
31. As an End User, I want `syllago push` to commit and push my signed Library changes, so that other machines can get them.
32. As an End User, I want `syllago bootstrap <repo-url>` on a fresh machine to clone, verify and apply in one command, so that a new machine or container is configured immediately.
33. As an End User, I want bootstrap to ask me to confirm the allowed signing keys the first time it sees a repository, so that trust starts from my decision (TOFU).
34. As an End User, I want `syllago sync` to pull, verify and apply, so that keeping a machine current is one command.
35. As an End User, I want sync to refuse an item whose signature doesn't verify and name the item and the reason, so that tampered content never reaches my tools.
36. As an End User, I want sync to refuse an item signed by a key not in my allowed list, so that a stolen push token can't add a signer.
37. As an End User, I want a change to the allowed-signers list itself to require a signature from a currently allowed key, so that an attacker can't add their own key with a plain push.
38. As an End User, I want sync to apply nothing when any item fails verification, so that a partly trusted update never lands.
39. As an End User, I want every command to support `--json`, so that I can script it and check results in CI.
40. As an End User, I want every error to name what failed, why, and what to do next, so that I can fix problems without reading source code.
41. As an End User, I want the skeleton to work on Linux, macOS and native Windows, so that every machine I own is covered.
42. As an End User, I want Windows paths, line endings and file locking handled from the start, so that a Windows machine never produces a different `body_hash` for the same content.
43. As an End User, I want syllago installed as a single static binary with signed checksums, so that I can verify what I'm running.
44. As a stack maintainer, I want one end-to-end test that runs this whole story against pinned versions of the ACIF suite, capmon feed and MOAT vectors, so that a change in any project that breaks the stack fails in one place.
45. As a stack maintainer, I want the ACIF crate to pass ACIF's conformance vectors for skills and rules, so that syllago's ACIF is the spec's ACIF.
46. As a stack maintainer, I want the MOAT author-key profile to ship with its own test vectors, so that any other Conforming Client can implement it.
47. As a stack maintainer, I want the Claude Code and Codex adapters to carry golden fixtures pinned to a tool version, so that a capmon change shows up as a fixture diff in a pull request.
48. As a stack maintainer, I want a capmon feed change that alters a path the skeleton uses to fail the end-to-end test until the fixture is regenerated, so that drift is never silent.
49. As a spec author, I want each spec change the skeleton needs to land through its project's change process before the code that depends on it, so that the specs stay the source of truth.

## Implementation Decisions

### Spec changes, landed first

- **ACIF `attestation_hash`.** Defined as the input MOAT signs, which is the
  item's `body_hash`. This retires MOAT's separate directory hash for ACIF
  content.
- **ACIF install conflicts.** The two the apply engine depends on are resolved
  before the engine is built. Hook contributions to shared files are keyed by
  item, not by event wiring. A blocking hook whose event would render degraded
  is refused at install, with an operator override. Neither touches skills or
  rules directly, but the ownership record and the rollback model are designed
  against the resolved text.
- **MOAT adopts `body_hash`.** MOAT's attestation payload carries the ACIF
  `body_hash` for ACIF content items.
- **MOAT author-key profile.** A new publisher signing profile signed by a
  hardware-backed key, not by CI keyless signing. It is recorded as the second
  attestation of the Dual Attestation tier. It changes MOAT's lexicon: Keyless
  Signing stops being the only normative mechanism, and Signing Identity gains
  a key-fingerprint form beside the OIDC form. The profile ships with test
  vectors.
- **MOAT signer subject format.** The spec and the npm sub-spec disagree on the
  subject form for the CI profile. One form is chosen and both texts updated,
  because the client's verifier is written once.
- **capmon structured install paths.** The feed publishes install paths for
  Claude Code and Codex as structured records in the ACIF install-targets
  schema. The prose `mechanism` fields stay, but they're no longer the only
  record.

### Modules

The Rust workspace in the syllago repository, beside the Go code until v1.0:

- **ACIF crate.** Parses and canonicalizes skills and rules, computes
  `body_hash`, and renders back to a provider through install-target records.
  Has no knowledge of syllago, git or signing. Public interface: canonicalize
  native content into an item, hash an item, render an item for an install
  target.
- **MOAT crate.** Verifies Dual Attestation for items whose second attestation
  comes from the author-key profile, and checks signer allow-lists. Signs with
  the author key. One Sigstore verification core is shared with the feed
  verifier. Public interface: sign a `body_hash`, verify an item against an
  allow-list, and verify a signed allow-list change.
- **Feed module.** Fetches the capmon feed, verifies its SLSA attestation
  fail-closed, and exposes install-target records per provider. It vendors no
  path tables.
- **Provider adapters.** One per provider: Claude Code and Codex in the
  skeleton. Each declares its tier, its cross-reads, and its tool-version-pinned
  golden fixtures. Adapters hold behavior only: paths come from the feed and
  formats from ACIF.
- **Apply engine.** Plans, previews, snapshots, applies across every targeted
  provider, records ownership, and rolls back on failure. Owns drift detection
  over owned files and owned keys.
- **Library and sync.** Owns the Library layout in the user's git repository,
  the loadout manifest, `init`, `push`, `sync`, `bootstrap`, `capture`, and the
  allowed-signers file with its TOFU confirmation.
- **CLI.** A thin layer over the modules above. Every command takes `--json`.

### Behavior

- Apply is all-or-nothing across every provider in one run.
- Ownership is recorded per placed file and per written settings key. Remove
  and drift detection read only that record.
- Sync verifies every item before applying any item. One failure applies
  nothing.
- A change to the allowed-signers file is itself signed by a key it already
  allows. A first sight of a repository requires explicit End User
  confirmation of its signers.
- A content type a provider can't take is skipped and reported. `--strict`
  turns skips into failures.
- The loadout manifest is redesigned for v2. It lists Content Items by ACIF
  identity and targets one or more providers. There is no v0.x compatibility.

## Testing Decisions

- **Good tests check behavior a user can observe:** files on disk, exit codes,
  `--json` output, what each tool would read. They never check internal data
  structures. A test that would survive a rewrite of the module it covers is
  the right test.
- **The primary seam is the CLI binary, end to end, across two simulated
  machines.** Each machine is an isolated home directory. A local bare git
  repository stands in for the remote, a pinned capmon feed fixture stands in
  for the live feed, and a test signing key and test trust root stand in for
  production keys and Rekor. One test drives the full story: add, sign, push,
  bootstrap on the second machine, verify, apply, check both tools' files,
  remove, and check that both tools are byte-identical to before.
- **Failure paths go through the same seam:**
  - an item signed by an unlisted key
  - an unsigned allow-list change
  - a tampered `body_hash`
  - a feed with a bad attestation
  - a failure injected mid-apply, with rollback checked byte for byte
  - a hand-edited owned file blocking apply until `capture` or `--overwrite`
- **The ACIF crate is tested at its existing seam:** the ACIF conformance
  runner and vectors for skills and rules. No new seam.
- **The MOAT crate is tested at MOAT's existing vector seam,** extended with
  the author-key profile's new vectors.
- **Provider adapters are tested by golden fixtures pinned to a tool version.**
  This is the one new seam below the CLI, because it is how a capmon change
  becomes a reviewable diff.
- **Prior art:**
  - syllago v0.15's loadout apply, remove and rollback tests, for the failure
    modes they cover
  - the snapshot tests, for same-second and duplicate-entry edge cases
  - the `install_moat` integration tests, for verification failure paths
  - ACIF's Python conformance runner
  - MOAT's `generate_test_vectors.py`
- **Every test runs sandboxed.** No test deletes a path outside its own
  temporary directory, and the injected-failure tests use a filesystem stub,
  never real deletion outside the sandbox.
- CI runs the end-to-end test on Linux, macOS and Windows.

## Out of Scope

- Hooks, MCP configuration, commands, agents and settings. They come in later
  slices, after the two ACIF install conflicts are resolved in the spec.
- OpenCode, pi, Cursor and Antigravity.
- Project-scoped loadouts. The skeleton is user-global only.
- Several active loadouts and conflict refusal between them.
- The systemd and launchd timer for sync.
- Secret references.
- The TUI.
- Registries, third-party content, and the team track.
- capmon automation beyond publishing structured paths for two providers.
- Removing or changing the Go v0.15 code.

## Further Notes

- The MOAT author-key profile is the largest spec change here, and it partly
  reverses a v0.4 decision (the removal of the SSH profile). It reuses the Dual
  Attestation tier instead of adding a tier, which keeps the v0.4
  registry-first model intact. The Self-Publisher role already describes a
  person who is both Publisher and Registry Operator, which is exactly the solo
  sync repository.
- "Hardware-backed" needs a concrete key type that works on Linux, macOS and
  Windows: for example a FIDO2 security key, or a platform authenticator
  through SSH's `sk-` key types. Picking it is the first implementation
  decision in the MOAT crate.
- The v0.15 lexicon defines Install as symlink or copy. v2 always writes
  rendered files and records ownership, because providers rewrite their own
  files and symlinks break on Windows.
- See the stack README for the ownership table and the decisions behind this
  slice.
