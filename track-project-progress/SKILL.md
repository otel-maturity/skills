---
name: track-project-progress
version: 0.0.1
description: Track a CNCF project's OpenTelemetry maturity evolution by querying GitHub issues and pull requests listed in TRACKING.md. Produces or updates EVOLUTION.md with items ordered by date (newest first).
argument-hint: "<project-name>"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Track Project Progress

You query GitHub issues and pull requests referenced in a project's `TRACKING.md` file and produce an `EVOLUTION.md` that records each item's date and description, ordered newest first.

## Required argument

The user provides the `<project-name>`. The TRACKING.md must already exist at `results/<project-name>/TRACKING.md` (or `.otel-eval/<project-name>/TRACKING.md`).

## TRACKING.md format

TRACKING.md is a Markdown file that lists GitHub issue and pull-request URLs to monitor, one per line, optionally followed by a short context note:

```markdown
# Tracking

- https://github.com/org/repo/issues/123
- https://github.com/org/repo/pull/456 - landed in v1.2
- https://github.com/org/repo/issues/789
```

Lines that do not start with `- https://github.com/` are treated as comments/headings and are ignored.

## Process

### Step 1: Read TRACKING.md

Read the TRACKING.md file provided in the prompt. Extract every GitHub URL that matches one of these patterns:

- Issue:       `https://github.com/<owner>/<repo>/issues/<number>`
- Pull request: `https://github.com/<owner>/<repo>/pull/<number>`

If no URLs are found, write a short message to the user explaining that TRACKING.md contains no queryable GitHub URLs, then stop.

### Step 2: Query GitHub for each item

For each extracted URL, determine the owner, repo, type (`issue` or `pr`), and number. Then fetch the item's metadata using the `gh` CLI:

**For an issue:**
```bash
gh api repos/<owner>/<repo>/issues/<number> \
  --jq '{type: "issue", number: .number, title: .title, state: .state, created_at: .created_at, closed_at: .closed_at, updated_at: .updated_at, url: .html_url, labels: [.labels[].name]}'
```

**For a pull request:**
```bash
gh api repos/<owner>/<repo>/pulls/<number> \
  --jq '{type: "pr", number: .number, title: .title, state: .state, created_at: .created_at, closed_at: .closed_at, merged_at: .merged_at, updated_at: .updated_at, url: .html_url, labels: [.labels[].name]}'
```

If a `gh` command fails (e.g. rate-limited, private repo, 404), skip that item and note the failure in the output section.

#### Determining the representative date for sorting

Use the following priority to pick the single date that best represents when an activity happened:

1. `merged_at` — for merged pull requests (most meaningful event)
2. `closed_at` — for closed issues or unmerged closed PRs
3. `created_at` — for open items

Format the date as `YYYY-MM-DD`.

### Step 3: Build the sorted list

Sort all successfully queried items by their representative date, **newest first**. Items with the same date are ordered: PRs before issues, then by number descending.

### Step 4: Write EVOLUTION.md

Determine the output directory — use the same directory that contains TRACKING.md (e.g. `results/<project-name>/`).

Write (or overwrite) `EVOLUTION.md` in that directory with the following structure:

```markdown
# Evolution

*Last updated: <today's date YYYY-MM-DD>*

A chronological record of GitHub issues and pull requests that track the project's
OpenTelemetry maturity progress, newest first.

---

## <YYYY-MM-DD> — <type>: [#<number> <title>](<url>)

**State:** <open|closed|merged>  
**Labels:** <label1>, <label2> *(omit this line if no labels)*  
**Description:** <one-sentence summary of what this item is about, derived from the title and any context note in TRACKING.md>

---

## <next item>
...
```

Rules:
- Each item gets its own `##` heading using its representative date.
- The `**Description:**` line is a single sentence you compose from the item's title. Keep it factual and concise (under 20 words).
- If the item had a context note in TRACKING.md (text after the URL), append it in parentheses after the description.
- If `gh` failed for an item, include a placeholder entry:
  ```markdown
  ## <URL>
  **Error:** Could not fetch data — <brief reason>
  ```

### Step 5: Verify

After writing, read the first 20 lines of EVOLUTION.md back to confirm it was written correctly. Report the count of items successfully fetched and the path of the file.

## Output

When complete, print:
```
Evolution written: <path-to-EVOLUTION.md>
Items tracked: <N> (<M> issues, <K> pull requests)

Open in editor:
  open <path-to-EVOLUTION.md>
```
