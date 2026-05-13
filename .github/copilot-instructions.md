# Copilot Instructions for Microsoft Learn

These instructions define a unified style and process standard for authoring and maintaining learn.microsoft.com documentation with GitHub Copilot or other AI assistance.

## Learn-wide Instructions

Below are instructions that apply to all Microsoft Learn documentation authored with AI assistance. The Learn product team will update this periodically as needed. Each repository SHOULD NOT update this to avoid being overwritten, but update the repository-specific instructions below as needed.

### AI Usage & Disclosure

All Markdown content created or substantially modified with AI assistance must include an `ai-usage` front matter entry:

- `ai-usage: ai-generated` – AI produced the initial draft with minimal human authorship
- `ai-usage: ai-assisted` – Human-directed, reviewed, and edited with AI support
- Omit only for purely human-authored legacy content

If missing, **add it**. However, do not add or update the ai-usage tag if the changes proposed are confined solely to:

- Links (link text and/or URLs)
- Single words or short phrases, such as entries in table cells
- Less than 5% of the article's word count

### Writing Style

Follow [Microsoft Writing Style Guide](https://learn.microsoft.com/style-guide/welcome/) with these specifics:

#### Voice and Tone

- Active voice, second person addressing reader directly
- Conversational tone with contractions
- Present tense for instructions/descriptions
- Imperative mood for instructions ("Call the method" not "You should call the method")
- Use "might" instead of "may" for possibility
- Avoid "we"/"our" referring to documentation authors

#### Structure and Format

- Sentence case headings (no gerunds in titles)
- Be concise, break up long sentences
- Oxford comma in lists
- Number all ordered list items as "1." (not sequential numbering like "1.", "2.", "3.", etc.)
- Complete sentences with proper punctuation in all list items
- Avoid "etc." or "and so on" - provide complete lists or use "for example"
- No consecutive headings without content between them

#### Formatting Conventions

- **Bold** for UI elements
- `Code style` for file names, folders, custom types, non-localizable text
- Raw URLs in angle brackets
- Use relative links for files in this repo
- Remove `https://learn.microsoft.com/en-us` from learn.microsoft.com links

## Repository-Specific Instructions

Below are instructions specific to this repository. These may be updated by repository maintainers as needed.

<!--- Add additional repository level instructions below. Do NOT update this line or above. --->

### Repository structure

This repository contains Azure Cosmos DB and Azure DocumentDB documentation published to learn.microsoft.com. Content is organized into five published docsets:

| Docset | Source folder | Description |
| --- | --- | --- |
| `azure-nosql` | `azure/` | Azure Cosmos DB service docs (how-tos, concepts, SDKs, migration) and Azure DocumentDB docs |
| `cosmos-db` | `cosmos-db/` | Standalone query language reference, indexing, and throughput concepts |
| `documentdb` | `documentdb/` | DocumentDB query reference (commands, operators) |
| `documentdb-generated` | `documentdb-generated/` | **Auto-generated API reference — DO NOT EDIT.** Protected by GitOps policy. |
| `nosql` | `nosql/` | Public contributor docset |

Key subdirectories:

- `azure/cosmos-db/` — ~200+ service articles including SDKs, gen-ai, shell, and sub-API docs (cassandra, gremlin, mongodb, postgresql, table)
- `cosmos-db/query/` — ~145 query function/clause reference docs
- `azure/documentdb/` — Azure DocumentDB (MongoDB-compatible) docs
- `documentdb-generated/` — Synced weekly from upstream via `.github/workflows/sync.yml`. Never edit directly.

### Content completeness

- Procedures must include all steps - check for missing prerequisites or follow-up actions
- Code samples must be complete and runnable - no placeholder comments like `// add code here`
- Ensure account creation steps reference the correct Azure portal experience
- Every article must end with either a "Related content" or "Next step" section (not both)
- "Related content" must be an H2 with a flat list of 3-5 hyperlinks and no other text or formatting:

```markdown
## Related content

- [Learn about vector indexing and search](vector-search-overview.md)
- [Learn about full text search](full-text-search-faq.yml)
- [Learn about hybrid search](hybrid-search.md)
```

- "Next step" must be an H2 with a single formatted button:

```markdown
## Next step

> [!div class="nextstepaction"]
> [Get started with change data capture](get-started-change-data-capture.md)
```

### Alert usage

- Limit alerts to one or two per article
- Never place multiple notes consecutively
- Use the correct alert type:
  - `[!NOTE]` - Information the user should notice even if skimming
  - `[!TIP]` - Optional information to help a user be more successful
  - `[!IMPORTANT]` - Essential information required for user success
  - `[!WARNING]` - Dangerous certain consequences of an action

### Markdown Formatting

#### Headings

- Each file must have exactly one H1 heading
- Do not skip heading levels (H2 to H4 without H3)

#### Code blocks

- Always specify the language identifier for syntax highlighting
- Use `azurecli` for Azure CLI commands
- Use `bash` for generic command line instructions

#### Tables

- Use exactly `| --- |` for column separators, not `|--------|`
- For alignment: `| :-- |` (left), `| :--: |` (center), `| --: |` (right)
- Empty first column headers are allowed when first column values are bolded

#### File endings

- All Markdown files must end with a trailing newline

### Writing Style

#### Word choice

| Use | Instead of |
| --- | --- |
| use | utilize |
| remove | eliminate |
| because | since (when meaning "because") |
| to | in order to |

#### Input-agnostic verbs

Use verbs that work for all input methods (mouse, keyboard, touch):

| Use | Instead of |
| --- | --- |
| Select | Click |
| Choose | Tap |
| Enter | Type |

#### Bias-free language

- Use people-first language: "users who are blind" not "blind users"
- Use gender-neutral terms: "sales representative" not "salesman"
- Show diverse perspectives in examples

### Pull request and issue naming

When creating pull requests or issues, follow this naming convention.

#### PR title format

```text
<Service prefix> | <Short description>
```

Choose the service prefix based on which files the PR changes:

| Prefix | When to use |
| --- | --- |
| `Cosmos DB` | Changes to `azure/cosmos-db/` or `cosmos-db/` files |
| `DocumentDB` | Changes to `azure/documentdb/` or `documentdb/` files |
| `NoSQL` | Changes that span both services, touch `nosql/` files, or affect cross-cutting configuration |

Examples:

- `Cosmos DB | Update query language TOC`
- `DocumentDB | Fix broken links`
- `NoSQL | Enable public contributions`

#### PR title suffixes

Append a bracketed suffix when the PR has a special status:

| Suffix | Meaning |
| --- | --- |
| `[WIP]` | Work in progress, not ready for review |
| `[BULK]` | Bulk mechanical change across many files |
| `[DO NOT MERGE]` | Experimental or test PR that must not be merged |

Examples:

- `Cosmos DB | Update branding to drop "for NoSQL" in articles [BULK]`
- `DocumentDB | Test manual content sync [DO NOT MERGE]`

#### Issue title format

AI-generated issues must include an emoji prefix. The presence of the emoji prefix indicates AI authorship, so there's no need for a "robot" emoji. Choose an emoji that reflects the nature of the issue rather than a generic robot emoji. Use a `[TODO]` suffix for future items or placeholder issues. Do not include a service prefix in issue titles.

Examples:

- `🔧 Add MongoDB delete examples to Python quickstart`
- `📊 AI Response Quality pilot — Cosmos DB retrievability improvements`
- `🟡 Items to fix — Partially incorrect`
- `🔗 Fix broken cross-reference links in partition key articles`
- `📝 Document RU charge behavior for cross-partition queries`

### Review Style

When providing feedback:

- Be specific and actionable - explain what to change and why
- Acknowledge good patterns when you see them
- Ask clarifying questions when intent is unclear
- Prioritize terminology issues and technical accuracy
- Focus on improvements that help readers succeed

### Content review checklist

When performing a pull request review, evaluate every changed file against this checklist. After your review, include the completed checklist in your review summary so the contributor can see exactly what was checked, what passed, and what needs attention. Mark each item as passing (`- [x]`), failing (`- [ ]`), or not applicable (`- N/A`).

```markdown
#### 📋 Content review checklist

- [ ] **Structure and completeness**
  - [x] Article has exactly one H1 heading
  - [x] Heading levels are not skipped (no H2 → H4 without H3)
  - [ ] Article ends with a "Related content" or "Next step" section (not both)
  - [x] Procedures include all steps with prerequisites and follow-up actions
  - N/A No consecutive headings without content between them
- [x] **Code samples**
  - [x] All code blocks specify a language identifier
  - [x] Code samples are complete and runnable
  - [x] No placeholder comments like `// add code here`
- [x] **Formatting and style**
  - [x] UI elements are **bold**
  - [x] File names and code references use `code style`
  - [x] Alerts are used correctly and sparingly (max 1-2 per article)
  - [x] Tables use `| --- |` separators
- [ ] **Technical accuracy**
  - [x] Product names follow terminology guidelines
  - [ ] Links are valid and use relative paths where applicable
  - [x] Feature flags and preview status are current
```

Adjust the checklist based on the content reviewed. Omit categories that don't apply (for example, skip "Code samples" for articles without code). Add specific findings as inline notes next to any failing items.
