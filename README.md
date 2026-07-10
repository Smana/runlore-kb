# RunLore knowledge catalog (OKF)

This repo is RunLore's [OKF](https://github.com/GoogleCloudPlatform/knowledge-catalog) knowledge
catalog: a tree of markdown files that the agent **reads** (cached + BM25-indexed) to ground
investigations, and **writes** to (via reviewed PRs) as it learns.

## How an entry works

- **One entry = one `.md` file** = YAML frontmatter + a markdown body.
- **The filename is a slug** — short, lowercase, hyphenated, descriptive
  (e.g. `payments-db-oom.md`). It's just the entry's identity; it is *not* indexed and *not* a
  frontmatter field. The Curator names learned entries `slugify(title).md`.
- **Put entries anywhere** — the repo root is fine, and subfolders (`playbooks/`, `incidents/`, …)
  work too; the whole tree is indexed recursively.
- **Reserved filenames** (skipped by the indexer, for humans): `index.md` (a listing) and `log.md`
  (the changelog). Dot-files are skipped too.
- **What gets searched:** `title` + `description` + `tags` + body (BM25). Write a precise title and
  tags so `kb_search` finds the entry — that, not the filename, is what matters.

## Frontmatter

```yaml
---
type: Playbook              # Playbook | Incident | Postmortem | …
title: HelmRelease upgrade failure
description: A Helm chart bump leaves the release Ready=False.
resource: helmrelease://*   # REQUIRED for Incident (namespace/name or scheme://…); optional for Playbook/Concept
tags: [flux, helmrelease, upgrade]
---
## Symptom
…
## Investigate
…
## Resolution
…
```

Unknown frontmatter keys (e.g. `okf_version`) are ignored; the body is everything after the
frontmatter.

### Rules the CI enforces

Every entry is checked on each PR by `lore validate-kb` (see `.github/workflows/validate-kb.yml`),
which runs RunLore's own validator so the rules never drift from what the agent enforces at index
time. `type`, `title` and `description` are always required. An **`Incident`** entry additionally
**must** carry a `resource:` (it is anchored to a concrete affected object, so recall can match it)
and the body sections `## Symptom`, `## Cause`, `## Resolution` (an `## Investigate` evidence
section is recommended). `Playbook`/`Concept` entries are abstract knowledge and stay free-form. A
structural error fails the merge; `index.md`, `log.md` and this `README.md` are skipped.

## How RunLore uses it

- **Read:** git-sync clones this repo and re-pulls on an interval, re-indexing automatically; the
  `kb_search` tool queries the index during investigations.
- **Write (curation):** a confident, novel finding becomes a **PR** drafting a new entry; an uncertain
  one becomes an **issue**. Merging a PR flows the new knowledge straight back into what the agent
  searches. Curation is the quality gate — review entries like code.

See `index.md` for the catalog listing and `log.md` for the change history.
