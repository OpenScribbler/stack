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
Library in ACIF form, signs them with my own security key, and pushes
them to a git repository I own. On a second machine I run one bootstrap command
pointing at that repository. Syllago clones it, verifies that every Content
Item was signed by my key, and installs the skill and the rule into Claude Code
and Codex in each tool's native format, at the paths capmon says each tool
reads. If anything in the apply fails, nothing is left half-installed. If I
want it gone, one command restores both tools to exactly what they were.

A content change pushed by anyone without my key never installs on any
machine without my confirmation, and syllago says why it stopped.

## User Stories

1. As an End User, I want to add an existing Claude Code skill to my Library, so that I can manage it from one place.
2. As an End User, I want to add an existing rule file to my Library, so that it can be rendered for every tool I use.
3. As an End User, I want added content stored in ACIF form, so that the same item has the same identity whichever tool it came from.
4. As an End User, I want each Content Item to carry its ACIF `body_hash`, and its `metadata_hash` when it has a `publisher_section` (frontmatter or sidecar values), so that any change to its body or to the tool, model and MCP grants in its frontmatter is detectable.
5. As an End User, I want adding the same skill twice from two different tools to yield one Library item, so that I don't get duplicates.
6. As an End User, I want `syllago init <repo-url>` to set up my Library as a GitHub repository I own, where MOAT's Registry Action (a GitHub Actions workflow) can sign it, so that my config has history and a remote.
7. As an End User, I want to create a security-key signing key (an SSH `sk-` key) once, so that my machines can tell my content apart from anyone else's.
8. As an End User, I want syllago to refuse to sign with a key that isn't an `sk-` key type unless I pass an explicit flag, so that I don't weaken my own setup by accident.
9. As a Self-Publisher, I want every Content Item I commit to be signed with my author key, so that a stolen push token can't produce content my machines install without my confirmation.
10. As a Self-Publisher, I want the signature to cover the ACIF attestation input digest, which binds `body_hash` and `metadata_hash`, so that a signed item's body and its frontmatter grants can't change without breaking the signature.
11. As a Self-Publisher, I want the author-key transparency-log entry to identify me only by my public key, so that signing never publishes my email address.
12. As a Self-Publisher, I want to declare which keys are allowed to sign for my repository, so that adding a second security key later is a config change rather than a rewrite.
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
33. As an End User, I want bootstrap, the first time it sees a repository, to ask me to confirm its allowed signing keys and to pin both that list and the registry's signing profile, so that trust starts from my decision (TOFU).
34. As an End User, I want `syllago sync` to pull, verify and apply, so that keeping a machine current is one command.
35. As an End User, I want sync to refuse an item whose signature doesn't verify and name the item and the reason, so that tampered content never reaches my tools.
36. As an End User, I want sync to refuse an item signed by a key not in my allowed list, so that a stolen push token can't add a signer without my confirmation.
37. As an End User, I want sync and bootstrap to refuse any item from my sync repository that isn't Dual-Attested under my author key from the `source_uri` pinned at bootstrap, so that a stolen push token can't add an item that only the Registry Action signed.
38. As an End User, I want a change to the allowed-signers list itself to require a signature from a currently allowed key and a higher sequence number, so that an attacker can't add their own key with a plain push or replay an old change to restore a removed key.
39. As an End User, I want sync to apply nothing when any item fails verification, so that a partly trusted update never lands.
40. As an End User, I want every command to support `--json`, so that I can script it and check results in CI.
41. As an End User, I want every error to name what failed, why, and what to do next, so that I can fix problems without reading source code.
42. As an End User, I want the skeleton to work on Linux, macOS and native Windows, so that every machine I own is covered.
43. As an End User, I want Windows paths, line endings and file locking handled from the start, so that a Windows machine never produces a different hash for the same content.
44. As an End User, I want syllago installed as a single static binary with signed checksums, so that I can verify what I'm running.
45. As a stack maintainer, I want one end-to-end test that runs this whole story against pinned versions of the ACIF suite, capmon feed and MOAT vectors, so that a change in any project that breaks the stack fails in one place.
46. As a stack maintainer, I want the ACIF crate to pass ACIF's conformance vectors for skills and rules, so that syllago's ACIF is the spec's ACIF.
47. As a stack maintainer, I want the MOAT author-key profile to ship with its own test vectors, so that any other Conforming Client can implement it.
48. As a stack maintainer, I want the Claude Code and Codex adapters to carry golden fixtures pinned to a tool version, so that a capmon change shows up as a fixture diff in a pull request.
49. As a stack maintainer, I want a capmon feed change that alters a path the skeleton uses to fail the end-to-end test until the fixture is regenerated, so that drift is never silent.
50. As a spec author, I want each spec change the skeleton needs to land through its project's change process before the code that depends on it, so that the specs stay the source of truth.

## Implementation Decisions

### Spec changes, landed first

- **ACIF `attestation_hash`**
  ([ACIF #12](https://github.com/OpenScribbler/agent-content-interchange-format/pull/12),
  Registry Specification §6). The value an external system signs is the
  attestation input digest, which ACIF derives from the item's own hashes:

  ```text
  preimage = "acif-attestation-input-v1\n" BH "\n" (MH or "null") "\n"
  attestation_hash = sha256 : lowercase-hex(SHA-256(preimage))
  ```

  `BH` and `MH` are `body_hash` and `metadata_hash`, each written as
  `algorithm:value`. An item with no `publisher_section` has no
  `metadata_hash`, and the preimage carries `null` in its place. Signing
  `body_hash` alone would leave the frontmatter's tool, model and MCP grants
  outside the signature, because `body_hash` strips frontmatter and only
  `metadata_hash` covers it. A consumer recomputes both
  hashes from the bytes it fetched, never from a record.
- **ACIF install conflicts** (the same PR). The two the apply engine depends
  on are resolved before the engine is built:
  - A hook's contributions to a shared file are keyed by item. Uninstalling
    one item leaves another item's identical entries in place.
  - A blocking hook whose event renders degraded, or whose provider ACIF
    records no event names for, is refused at install. An operator can
    override the refusal for one item and one provider.

  Neither touches skills or rules directly, but the ownership record and the
  rollback model are designed against the resolved text.
- **MOAT signs the ACIF digest**
  ([MOAT #11](https://github.com/OpenScribbler/moat/pull/11)). A manifest
  entry marked `content[].acif` has `sha256:` plus the attestation input
  digest as its Content Hash. A client hashes the bytes it fetched, never what
  it renders for a provider, and installs only the bytes the digest covers. A
  single-file item installs its entry file alone, and an item whose
  frontmatter carries a key ACIF observation did not record is refused. The
  Library stores items in ACIF canonical form, so for syllago the MOAT Content
  Hash and ACIF's `attestation_hash` are the same value.
- **MOAT author-key profile** (the same PR). A publisher attestation can be an
  OpenSSH signature (SSHSIG, namespace `file`, SHA-512) by an `ssh-ed25519`,
  `sk-ssh-ed25519@openssh.com` or `sk-ecdsa-sha2-nistp256@openssh.com` key. It
  is logged in Rekor as a comment-free `rekord` entry, found by the item
  message's hash and selected by key fingerprint, and it is recorded as the
  second attestation of the Dual-Attested tier. A repository lists its keys in
  `.moat/allowed_signers`, with a sequence number:
  - A list change goes through without confirmation only when a trusted key
    signs a message binding the repository's `source_uri`, the new sequence,
    and both list digests.
  - A changed list whose sequence does not increase is refused outright, with
    no confirmation offered.
  - A first sight, an unsigned change, or a switch to keyless signing needs
    the End User's confirmation. So does an update to an installed item whose
    `source_uri` changed, when a trusted list exists for the recorded
    `source_uri`.
  - The End User can remove a repository's trusted list. That is the recovery
    path after a key is stolen.

  The profile ships with test vectors TV-AK-01 to TV-AK-12. MOAT's lexicon
  gains a key-fingerprint form of Signing Identity.
- **MOAT signer subject format** (the same PR). The CI profile's subject is
  the Fulcio certificate's SAN URI, whatever Fulcio's issuer configuration
  derives it from. The spec and the npm sub-spec now agree.
- **capmon structured install paths**
  ([capmon #77](https://github.com/OpenScribbler/capmon/pull/77)). Each
  `claude-code` and `codex` content-type node in the feed carries
  `install_entry_points`, a list of rows in the ACIF install-targets schema,
  and `schema_version` moved to 3. The prose `mechanism` fields stay, but
  they're no longer the only record.

### Modules

The Rust workspace in the syllago repository, beside the Go code until v1.0:

- **ACIF crate.** Parses and canonicalizes skills and rules, computes
  `body_hash`, `metadata_hash` and the attestation input digest, and renders back to a provider through install-target records.
  Has no knowledge of syllago, git or signing. Public interface: canonicalize
  native content into an item, hash an item, render an item for an install
  target.
- **MOAT crate.** Verifies Dual Attestation for items whose second attestation
  comes from the author-key profile, and checks signer allow-lists. Signs with
  the author key. One Sigstore verification core is shared with the feed
  verifier. Public interface: sign the item message for an item's Content
  Hash (which for syllago's canonical items equals the attestation input
  digest), verify an item against an allow-list, and verify a signed
  allow-list change.
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
- Sync verifies the sync repository's signed manifest before any of its
  items. The repository is the user's own MOAT Self-Publisher registry: it
  publishes a manifest with `self_published: true`. MOAT registry manifests
  are signed keyless in CI, so the sync repository runs MOAT's Registry
  Action. The manifest signature proves only the index's integrity. For each
  item, the Registry Action also writes a per-item Rekor entry
  (`content[].rekor_log_index`), the registry attestation that the
  author-key signature joins to make the item Dual-Attested. Sync verifies the
  manifest bundle, then each item's registry entry and author-key entry.
- The Registry Action regenerates and re-signs the manifest on a schedule as
  well as on push, so its `expires` never lapses for a repository with
  infrequent pushes.
- Sync and bootstrap refuse any item from the sync repository that is not
  Dual-Attested under the author-key profile from the `source_uri` pinned at
  bootstrap. That covers an entry with no `signing_profile`, a keyless one,
  and one under a different `source_uri`. This is syllago's own policy: MOAT
  alone installs an entry with no `signing_profile` as a Signed item, and a
  new keyless item under a different `source_uri` without asking.
- Sync verifies every item before applying any item. One failure applies
  nothing.
- A change to the allowed-signers file is itself signed by a key it already
  allows, and raises the file's sequence number. A first sight of a
  repository requires explicit End User confirmation of its signers.
- `syllago add` refuses an item whose frontmatter MOAT would refuse at
  install: one that carries a key ACIF observation does not record, or a
  scalar that reads differently under YAML 1.1 than under the YAML 1.2 core
  schema.
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
  repository stands in for the remote, and a pinned capmon feed fixture
  stands in for the live feed. A test signer plays the Registry Action: it
  emits the manifest, its bundle and each item's registry Rekor entry under a
  test trust root. A test author key stands in for production keys, and a
  Rekor stand-in under the same root supports search by data hash, which
  author-key lookup needs. One test drives the full story: add, sign, push,
  bootstrap on the second machine, verify, apply, check both tools' files,
  remove, and check that both tools are byte-identical to before.
- **Failure paths go through the same seam:**
  - an item signed by an unlisted key
  - an item from the sync repository with no `signing_profile`, a keyless
    one, or one under a different `source_uri`, each refused with nothing
    applied
  - an unsigned allow-list change, which sync holds for End User confirmation,
    applying nothing from that repository until then
  - a changed allow list whose sequence is not greater, refused outright with
    no confirmation offered
  - a tampered item body, which changes `body_hash`
  - an edited frontmatter grant, which changes only `metadata_hash`
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
  slices. MOAT signs ACIF items only for skills, rules, agents and commands
  until the sidecar-only types are normative there.
- OpenCode, pi, Cursor and Antigravity.
- Project-scoped loadouts. The skeleton is user-global only.
- Several active loadouts and conflict refusal between them.
- The systemd and launchd timer for sync.
- Secret references.
- The TUI.
- Third-party registries, third-party content, and the team track. The
  user's own sync repository is in scope as a MOAT Self-Publisher registry.
- capmon automation beyond publishing structured paths for two providers.
- Removing or changing the Go v0.15 code.

## Further Notes

- The MOAT author-key profile is the largest spec change here, and it partly
  reverses a v0.4 decision (the removal of the SSH profile). It reuses the Dual
  Attestation tier instead of adding a tier, which keeps the v0.4
  registry-first model intact. The Self-Publisher role already describes a
  person who is both Publisher and Registry Operator, which is exactly the solo
  sync repository.
- MOAT's profile allows `ssh-ed25519` and the two `sk-` key types, and
  carries no authenticator attestation. A key type proves nothing about
  hardware binding, so MOAT forbids a client from describing a key, signature
  or item as hardware-backed. Syllago's refusal to sign with a non-`sk-` key
  (story 8) is its own signing policy on the author's machine. It never
  becomes a claim shown to anyone verifying. Which `sk-` type to default to on
  Linux, macOS and Windows is the first implementation decision in the MOAT
  crate.
- The sync repository's registry signatures are Fulcio certificates whose
  SAN names the repository's owner and name, in a public log.
- The v0.15 lexicon defines Install as symlink or copy. v2 always writes
  rendered files and records ownership, because providers rewrite their own
  files and symlinks break on Windows.
- See the stack README for the ownership table and the decisions behind this
  slice.
