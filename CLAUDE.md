# Working in this repository

## `_corpus/heldout/` — do not read, do not run anything against it

**This is the one instruction in this file that cannot be recovered from if you get it wrong.**

`_corpus/heldout/` holds a frozen set of real published papers whose entire purpose is to be
material **no detector was written knowing about**. Its protocol
(`reverse_engineer/HELDOUT.md`) states the spend rule directly:

| set | role | spent by |
|---|---|---|
| challenge-card fixtures | train | authored with the detector |
| the current corpus | validation | **acting on a fire** |
| a fresh frozen corpus | test — one unbiased number | **reading it** |

Reading is not a neutral act here. A fire rate measured on these papers asserts *"no detector was
written knowing this paper"*; reading one in order to author or justify a change is exactly how a
detector comes to know it. The protocol's own instruction is **"Never open them again except to
label a fire."**

So, when working anywhere in this repository:

- **Do not open** `_corpus/heldout/*.md`.
- **Do not run** a detector, a grep, or a scan across `_corpus/`.
- **Do not cite** a number derived from it as evidence for changing a detector.
- If a task seems to require it, stop and ask. It almost certainly does not.

`_corpus/` is gitignored, so `git status` will never show a change there and a clone will not carry
it. Neither fact protects it: **anything scanning the working tree sees it.** That is not
hypothetical — on 2026-07-31 an audit was launched against "the repository", three of its agents
reached `_corpus/heldout/`, and one ran a detector across the corpus and opened two of the papers.
The finding it produced was quarantined and not acted on, and the corpus in place was already spent,
so nothing measurable was lost. The next corpus is the one that reading destroys, and this file
exists because the fence was in a prompt instead of in the repository.

## Merging pull requests

This is a fork of `Aperivue/medsci-skills`, maintained by its owner for hematology and
transfusion-medicine work. **Ask the fork owner before merging any pull request.** A request to
"open the PR and merge it once checks pass" is that permission for that pull request only. Merge
only when CI is green on the current head and there is no merge conflict.

Use a **merge commit**, not a squash, for a pull request that brings in upstream changes: a squash
discards the upstream history and makes the next upstream merge conflict with itself. Any other
pull request may be merged either way; default to a merge commit.

A blanket "merge once checks pass" does not cover the following. Name the item to the fork owner
and wait for a yes to that item:

- a change to what a detector flags or clears, or to its message — that is what a user is told about
  their manuscript;
- a change to a medical or research claim, or to a reporting checklist — the fork owner reviews
  these as the clinical reader;
- a release, a version bump or a tag — releases belong to upstream, not to this fork;
- anything touching `_corpus/`.

## Working from a cloud session

A cloud session sees this repository and nothing else. The fork owner's own Claude rules, skills
and memory are on their machine, and so are their manuscript and project folders. Five rules
matter here, so they are stated.

**Talk to the fork owner in English.** What lands in the repository is English too: code,
comments, commit messages and pull-request text (`CONTRIBUTING.md` → *Language Policy*).

**Scan what CI does not.** `validate_skills.sh` scans files. A commit message and a pull-request
body are stored separately, stay public, and nothing checks them. Before `git commit`, and before
`gh pr create` or `gh pr edit`, write the text to a file and run:

```bash
python3 scripts/check_precedent.py pr_body.md   # exit 0 = clean, 3 = hit
```

The blocklist is hash-only on purpose. Never write an identifier itself into this repository, not
in a gate, a comment, or a sentence saying it is absent. To report that something is clean, name
the class ("no manuscript IDs"), not the value.

**Stop and ask before building a new wall.** Before adding a gate, detector, CI check or skill,
changing direction (positioning, target, distribution channel), or touching three or more gates
or skills in one batch, stop and brief the fork owner in plain English: what it costs, what it could
hide, and which real failure it answers. Bug fixes, corrections of a typo or a stated fact, and
mechanical steps of a plan the fork owner already approved go ahead without this.

**Review the final diff blind.** Before a pull request is merged, hand its final diff to a reviewer
that did not write it: a different model family if one is available, otherwise a fresh agent, and
then say that the review is not independent. Give it the diff, one line of intent per change, and
the paths it must not open (`_corpus/`). Do not give it your checklist, earlier findings or a note
that something was fixed. A review steered by a checklist finds what the checklist names; on
2026-10-05 a blind pass found two defects that a checklist-guided review had passed. Fix what it
finds, and say in the pull request how many rounds ran.

**Do not reconstruct missing local context.** If a task depends on a manuscript, a project folder,
a reviewer's comment or anything else that is not in this repository, say that it is missing and
ask. Do not infer it from file names or commit history.

## Everything else

`CONTRIBUTING.md` is the entry point: what to run before pushing (CI is the merge gate), the
worktree discipline, and what a change is expected to ship with.
