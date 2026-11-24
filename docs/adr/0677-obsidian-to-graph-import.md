---
type: adr
area: cli
claim: 0d3671430c1f49c0e5f13e5e7c39709a71005004
name: 0677-obsidian-to-graph-import
status: proposed
tags:
  - load
title: The obsidian-to-graph import contract
REFERENCES:
  - "[[adr/0157-fs-to-graph-import]]"
  - "[[adr/0376-git-to-graph-import]]"
  - "[[adr/0377-import-resource-bounds]]"
  - "[[adr/0406-markdown-vault-snapshot-codec]]"
  - "[[adr/0429-cli-extension-registry]]"
  - "[[adr/0430-cli-extension-wire-protocol]]"
  - "[[adr/0431-cli-extension-distribution]]"
  - "[[adr/0447-extension-single-wire]]"
  - "[[adr/0602-fs-import-per-batch-commit]]"
  - "[[adr/0647-vault-identity-fold-composes-to-nfc]]"
---

# ADR-0677: The obsidian-to-graph import contract

## Context

A folder of markdown notes — an Obsidian vault, or a wiki an agent maintains
as plain `.md` files with `[[links]]` — is a body of knowledge people want to
hand to an agent as one queryable, portable artifact: a single store file
with a fixed graph shape. Neither existing piece does this job:

- The markdown snapshot codec ([ADR-0406](./0406-markdown-vault-snapshot-codec.md)) makes a vault the live database
  *in place*. It assumes a layout (top-level folder is the label, the
  filename is the key), reads links only from frontmatter, requires a `type`
  key and reserves `index.md` and `log.md`. A wiki with no frontmatter does
  not fit, and the result is not a portable single file.
- The first-party importers (`fs-to-graph`, [ADR-0157](./0157-fs-to-graph-import.md); `git-to-graph`,
  [ADR-0376](./0376-git-to-graph-import.md)) are extension packages ([ADR-0431](./0431-cli-extension-distribution.md) §11): child processes that
  drive the host over the extension wire ([ADR-0430](./0430-cli-extension-wire-protocol.md), [ADR-0447](./0447-extension-single-wire.md)). The host owns the
  store and the policy. Neither understands markdown.

## Decision

A third first-party importer converts a folder of markdown notes into a graph
with a fixed schema, written to an ordinary store.

### Command shape

A package `obsidian-to-graph`, mounted as `relate run obsidian-to-graph [DIR]
[flags]` ([ADR-0429](./0429-cli-extension-registry.md)); importers live under `run`, with no new verb
([ADR-0431](./0431-cli-extension-distribution.md) §11). `DIR` defaults to the current directory and `--database`
to `:memory:`. The package's own flags:

- `--no-body` stores no note bodies.
- `--no-prune` keeps notes, tags and edges this run did not see.
- `--ignore-patterns PAT` (repeatable) prunes entries by exact name, as in
  [ADR-0157](./0157-fs-to-graph-import.md).
- `--label-by folder|none` (default `folder`) sets whether the top-level
  folder adds a label.

Only `.md` files are read; `.obsidian/`, `.trash/`, `.relate/` and any path
component starting `.git` are skipped. The child carries no `Database`: it
works through the wire within the host's caps and per-batch commit policy ([ADR-0377](./0377-import-resource-bounds.md), [ADR-0602](./0602-fs-import-per-batch-commit.md)).

### Graph schema

- **`Note`** nodes, one per file, with `path`, `name` (the filename without
  `.md`), `hash` (SHA-256 of the bytes), optional `body` (the text after the
  frontmatter fence, or the whole file) and the frontmatter keys. Every note
  carries `Note`. Under `--label-by folder`, a note below a top-level folder
  also carries that folder's label, sanitized as the codec sanitizes labels.
- **`path`** is vault-relative, `/`-separated and NFC-normalized
  ([ADR-0647](./0647-vault-identity-fold-composes-to-nfc.md)); it is the identity key.
- **Frontmatter is optional.** Scalars and same-type lists are stored as they
  are, a mixed list as strings, and a nested map as dotted keys
  (`author.name`), since properties cannot hold maps. A key colliding with an
  importer property is stored under an `fm_` prefix; nothing is dropped.
- **Frontmatter `type` is only a property**, never a label: a wiki with no
  `type` keys would otherwise get unpredictable labels.
- **`TAGGED`** edges link a note to a **`Tag`** node, one per full tag name,
  case-folded, without `#`. Tags come from frontmatter `tags` and inline
  `#tag` (not in code, not a heading marker). Tags are edges only, never also
  a property: parallel state is a bug. A hierarchical tag `a/b/c` is one node
  plus `CHILD_OF` edges to each ancestor, so a subtree is a traversal.
- **`aliases`** stay a list property and take no part in link resolution:
  Obsidian resolves by filename or path, so a bare alias does not resolve
  there either.
- **`LINKS_TO`** has one edge per distinct source, target, alias, heading,
  block, embed flag and `via`, covering wikilinks, embeds and Markdown links
  to `.md` files. A wikilink in a frontmatter value carries `via` naming that
  key, so `author: "[[X]]"` keeps the codec's relationship-key information.
  Links in code are not links.
- **Dangling links become stub notes**: a name, a stub marker, no `path`.
- `index.md` and `log.md` are ordinary notes; the codec's reserved-view rule
  does not apply. Uniqueness constraints on note path and tag name and a
  fulltext index over note name and body are created.

### Identity, change detection and re-import

The key is the path, never the hash: identity must survive an edit, or every
edit would orphan the edges and any reference an agent cached. Each run
hashes every file, reads the stored path and hash pairs, writes the full node
row only for notes that are new, changed or lacking a requested body, and
stamps everything it saw with the run id.

**Edges are recomputed from the files every run.** A link's target depends
on the whole vault's name index, so a new or renamed note can change where an
unchanged note's links point. The hash gates only the large node-property
write, never the edges. A re-run over an unchanged vault is idempotent.
Importing with `--no-body` after a full import leaves stored bodies in place;
drop bodies with a fresh store.

### Pruning

Unless `--no-prune`, one sweep, run only after every write batch succeeded,
deletes what the run did not stamp: notes whose files are gone, unlinked
stubs, vanished edges, and unused tags. A partial run keeps its committed
batches ([ADR-0602](./0602-fs-import-per-batch-commit.md)) and never sweeps, so it cannot delete a note it did
not reach. `--no-prune` is how several vaults merge into one store.

### Link resolution, failure and bounds

A link resolves as Obsidian resolves it: a target with `/` matches by path
suffix, a bare target by case-folded filename, `.md` implied. Among several
matches the source note's folder wins, then the shortest path, then byte
order; no match makes a stub. The run reports counts of ambiguous links,
dangling links and notes whose frontmatter did not parse; none fails the
import. A frontmatter parse failure imports the note with no frontmatter
properties and its whole file as body. A file over the per-file cap imports
without its body and is counted ([ADR-0377](./0377-import-resource-bounds.md)). Rows stream in bounded batches;
memory scales with note count, not text.

## Alternatives considered

- **Extend the snapshot codec to import.** A layout-independent import would
  loosen its in-place, strict-key guarantees.
- **A `relate import --format obsidian` verb** would duplicate the extension
  mount and grow the closed verb list.
- **Hash in the identity key** orphans a node's edges on every edit; **tags
  as a list, maps as JSON** make the data opaque to queries.

## Consequences

- A folder of notes becomes one store file with one public-contract shape;
  a re-import is incremental in writes, not in reads. Dotted keys need
  quoted access. Pruning is on by default, so a shared store loses what an
  earlier run wrote for the same notes.
- Open for the implementation: whether the path constraint admits stubs, the
  per-file body cap, tag display spelling, and a folder name that sanitizes
  to nothing.
