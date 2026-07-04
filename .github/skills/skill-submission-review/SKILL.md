---
name: skill-submission-review
description: Review PRs that add or update skills under Skills/ in pnp/sharepoint-skills. Validates folder structure, kebab-case naming, SKILL.md frontmatter, README template compliance, and assets/sample.json metadata so manual review effort is minimized. Use when reviewing a new skill submission or checking a skill folder for repo conventions. Triggers on "review PR", "review this skill", "validate skill submission", or PR review requests for skill folders.
---

# Skill Submission Review

Review PRs that add or change skill folders under `Skills/` for compliance with the standards in [CONTRIBUTING.md](../../../CONTRIBUTING.md). Every rule below comes from that document — if this skill and CONTRIBUTING.md ever disagree, CONTRIBUTING.md wins.

## Workflow

1. Checkout the PR: `gh pr checkout <number>`
2. Identify the skill folder(s) touched by the PR: `git diff --name-only origin/main... | grep '^Skills/' | cut -d/ -f2 | sort -u`
3. Run every check below for each touched skill, collecting comments (file, line, issue)
4. Submit all comments in a single review API call (GitHub limitation — see [Submitting Reviews](#submitting-reviews))

Let `<skill-name>` be the outer folder name under `Skills/` in the checks that follow.

## Checks

### 1. Folder structure

The expected layout is:

```
Skills/<skill-name>/
├── README.md                      (required)
├── assets/
│   ├── sample.json                (required)
│   └── preview.png                (required)
├── <skill-name>/                  (required — upload-ready inner package)
│   ├── SKILL.md                   (required)
│   └── ...extra runtime files     (optional, e.g. persona documents)
└── demo/                          (optional)
    ├── README.md
    └── sample-files/
```

- **Outer folder** is directly under `Skills/` — not nested deeper, not at repo root
- **Inner package folder** exists at `Skills/<skill-name>/<skill-name>/` and its name matches the outer folder exactly
- **`SKILL.md` lives inside the inner package**, not in the outer folder
- **Repo docs stay in the outer folder** — `README.md`, `assets/`, and `demo/` must NOT be inside the inner package (it is uploaded to SharePoint as-is)
- Extra runtime files the skill needs (reference docs, personas) sit beside `SKILL.md` inside the inner package
- The PR must not modify unrelated skill folders or repo-level files without explanation

### 2. Naming convention

- Folder name is **kebab-case**: lowercase letters, digits, and dashes only (`organize-library`, not `OrganizeLibrary`, `organize_library`, or `Organize Library`). Regex: `^[a-z0-9]+(-[a-z0-9]+)*$`
- Folder name **exactly matches** the `name` field in `SKILL.md` frontmatter (character for character)
- Name is specific — flag generic names like `tool`, `helper`, `processor`, `utility`
- Name does not collide with an existing folder under `Skills/`

### 3. SKILL.md

- Starts with YAML frontmatter delimited by `---` lines containing:
  - `name`: matches the folder name (checked above)
  - `description`: states what the skill does, when to use it, and ideally trigger phrases — this is how the skill gets discovered
- Body contains actual instructions (a heading plus numbered steps or structured guidance), not just a placeholder
- Skill is **focused** (one capability) and **self-contained** — flag instructions that depend on files outside the inner package or on external services the runtime can't reach

### 4. README.md (outer folder)

Compare against the template in CONTRIBUTING.md. Required elements, in order:

- **Title + short description** of what the skill does and when to use it
- **Preview image**: `![preview](./assets/preview.png)`
- **"What you get" section** with a bullet list of outputs/outcomes
- **"SharePoint Skill" credits table** with solution name and actual author name(s) with GitHub (and optionally LinkedIn) links — not "Microsoft" or placeholder text like "Your Name"
- **"Version history" table** with version, date, and comments; the author implied there should match the PR author
- **Disclaimer** section with the standard AS-IS warranty text in bold
- **Visitor-stats image** as the last element, pointing at this skill's path:
  ```html
  <img src="https://m365-visitor-stats.azurewebsites.net/sharepoint-skills/skills/<skill-name>" />
  ```
  Verify `<skill-name>` in the URL matches the actual folder name
- No trailing `---` at the end of the file

### 5. assets/sample.json

Compare against the template in CONTRIBUTING.md. It is a JSON **array** with one object. Verify:

| Field | Expected value |
|---|---|
| `name` | `pnp-sharepoint-skills-<skill-name>` |
| `source` | `pnp` |
| `title` | Human-readable skill title |
| `shortDescription` | Non-empty, matches what the skill actually does |
| `url` | `https://github.com/pnp/sharepoint-skills/tree/main/Skills/<skill-name>` (actual folder name, not the template's `summarize-page`) |
| `longDescription` | Array of one or more real paragraphs |
| `creationDateTime` / `updateDateTime` | Valid `YYYY-MM-DD` dates, not in the future, not left as template values |
| `products` | Includes `SharePoint` |
| `metadata` | Includes `SAMPLE-TYPE` = `SharePoint-AI-Skill` |
| `thumbnails[0].url` | `https://github.com/pnp/sharepoint-skills/raw/main/Skills/<skill-name>/assets/preview.png` |
| `authors` | Real GitHub account(s) — cross-check `gitHubAccount` against the PR author (`gh pr view <PR> --json author`); flag placeholders like `your-handle` |

Also verify the file parses as valid JSON: `python3 -m json.tool Skills/<skill-name>/assets/sample.json`.

### 6. assets/preview.png

- File exists at `Skills/<skill-name>/assets/preview.png`
- Recommended 1280×720 (16:9) PNG — check with `file` or `python3 -c "from struct import unpack; ..."` if needed; a different size is a suggestion, not a blocker
- Should show the skill's actual output, not a logo or placeholder (judge from the image if viewable)

### 7. demo/ (only if present)

- Lives in the **outer** folder, not inside the inner upload package
- Contains a `README.md` with setup steps and a `sample-files/` folder with the content

## Severity

- **Request changes** for: wrong folder structure, naming violations, missing required files, missing/incorrect `name`/`description` frontmatter, template placeholder values left in `sample.json` or README, wrong URLs/paths in `sample.json` or visitor-stats image, invalid JSON
- **Comment (non-blocking)** for: preview image size, wording suggestions, optional demo content, minor formatting

## Submitting Reviews

GitHub API cannot add comments to a pending review after creation. Collect all comments first, then submit in one call.

Get the commit SHA:

```bash
gh pr view <PR> --json headRefOid
```

Create `review.json`:

```json
{
  "commit_id": "<sha>",
  "event": "REQUEST_CHANGES",
  "body": "Summary of findings with a checklist of what passed and what needs fixing",
  "comments": [
    {"path": "Skills/<skill-name>/assets/sample.json", "line": 10, "body": "Issue description"}
  ]
}
```

Use `"event": "COMMENT"` (or `"APPROVE"` if permitted) when all blocking checks pass.

Submit:

```bash
gh api repos/pnp/sharepoint-skills/pulls/<PR>/reviews --method POST --input review.json
```

In the review body, include the pre-submission checklist from CONTRIBUTING.md with each item marked pass/fail so the contributor sees exactly what is left.
