# REVISION.md

This document defines the workflow for writing a Revision History section for the current HEAD.
A Revision History section records what changed in each released or pending version of a project.
It lives in the project `README.md` (or `CHANGELOG.md` when present) under a `## Version History` heading.

## Source of Truth by Language

Choose the change source by the project's primary language:

- **Rust, Python, F#:** derive entries from **git commits**. These ecosystems bump versions inside manifest files (`Cargo.toml`, `pyproject.toml`, `*.fsproj`), so commit history between version bumps is the reliable source.
- **Go, Bash:** derive entries from **git tags**. These ecosystems have no versioned manifest, so annotated tags mark releases; commits between tags describe each release.

If a repo mixes languages, use the rule for the deliverable being versioned (e.g., the `.ipk`/binary's language).

---

## Rust / Python / F#: Commit-Based History

### 1. Find version bumps

- **Rust:** `git log --follow --oneline -- <crate>/Cargo.toml` and diff each commit for `version = ` changes.
- **Python:** same over `pyproject.toml` (`version = `) or `setup.cfg`.
- **F#:** same over the `.fsproj` (`<Version>`).

This yields the boundary commit for each version.

### 2. Collect commits for the current version

```
git log --format='%h %s' <prev-version-commit>..HEAD
```

### 3. Write the entry

Label the section `### <version> (current HEAD)` when the version exists only on HEAD and is not yet tagged/released.

Group commits by theme (e.g., schema changes, enforcement, analytics, docs). For each group or notable commit:

- Lead with a one-line summary derived from the commit subject.
- Expand with details from the commit body when it exists (`git log -1 --format='%b' <hash>`).
- Reference the commit hash in parentheses, e.g. ``(`abc1234`)``.
- Merge trivial commits (typo fixes, test-only tweaks) into the related feature line; drop pure noise.

### 4. Version bumps with no content

When a version bump commit carries no changes of its own, write "Version bump only" and attribute the following feature commits to the *next* version section.

---

## Go / Bash: Tag-Based History

### 1. List release boundaries

```
git tag --sort=-creatordate        # newest first
git describe --tags --abbrev=0     # latest tag reachable from HEAD
```

### 2. Collect commits for the current version

```
git log --format='%h %s' <latest-tag>..HEAD     # unreleased changes on HEAD
git log --format='%h %s' <tag-n-1>..<tag-n>     # a released version
```

If HEAD has no commits past the latest tag, mark the top entry as the tagged release; otherwise label it `<next-version> (current HEAD)` or `Unreleased`.

### 3. Write the entry

Same grouping and attribution rules as step 3 above. Reference tag names for released versions (e.g., `v0.3.0`) instead of hashes in the section heading.

---

## Output Format

Use this structure:

```markdown
## Version History

---

### 0.1.11 (current HEAD)

- **<Theme>**: <one-line summary> (`<hash>`).
  - <sub-detail derived from commit body>.
- <one-line summary> (`<hash>`).

### 0.1.10

- **<Theme>**: <summary> (`<hash>`).
```

Rules:

- Newest version first.
- Keep the existing file's heading style (`###` per version).
- Use [STYLE.md](STYLE.md): active voice, present tense, one idea per sentence, technical names exempt.
- Bold the theme label; put commit hashes in backtick-wrapped parentheses.
- Do not invent version numbers that never existed; if bumps skip a number (e.g., 0.1.4 → 0.1.6), note the skipped version as folded into the next bump.
- Cite real commit hashes and tag names only. Verify each with `git log -1` / `git tag` before writing.

---

## Verification Checklist

Before finishing:

1. Every hash/tag cited resolves (`git cat-file -t <ref>`).
2. Every version in the section maps to a real manifest bump or tag.
3. No commit between the boundary and HEAD is unaccounted for (`git log <boundary>..HEAD` count vs. covered lines; trivial merges may be folded).
4. Skipped version numbers are explained.
5. The entry passes [STYLE.md](STYLE.md) rules.
