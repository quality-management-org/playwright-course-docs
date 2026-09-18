# CLAUDE.md - Project context for Claude Code

## Project

Student-facing course materials for "Automation Testing tu Zero den Hero" (Playwright + TypeScript), aimed at manual testers with no prior TypeScript/Playwright experience. 15 sessions, 4 phases, 8 weeks. Session-by-session outline: `course-outline.md` at repo root.

Published via GitBook Git Sync: pushing to `main` goes live to students within about one minute.

**All document content is written in Vietnamese.** Only this file, skill files, and code identifiers follow English conventions described below.

## Repo structure (hard rules)

- `README.md`: homepage with a card grid (`<table data-view="cards">`) linking to lessons. Adding a lesson requires adding its card.
- `SUMMARY.md`: sidebar table of contents. GitBook ONLY renders pages listed here. Any add/remove/rename of a .md file must update SUMMARY.md.
- `.gitbook.yaml`: sync config, do not modify.
- `course-outline.md`: the 15-session outline (internal reference, written in Vietnamese). Keep it out of SUMMARY.md.
- `phase-N/buoi-XX-short-name.md`: one file per session. Filenames are kebab-case ASCII. Display titles (H1 and SUMMARY link text) are Vietnamese with full diacritics.
- Never rename existing lesson .md files: it breaks GitBook URLs already shared with students.

## Writing style (most important rules)

- Documentation-guide register: declarative sentences, neutral instructions ("Chay lenh sau", "Tao file X"). No colloquial/spoken language, no exclamations, no jokes, no emoji in body text.
- NEVER use the em dash character "—". Replace with commas, colons, parentheses, or split the sentence. Only the plain hyphen "-" is allowed (e.g. "45-60 phut").
- Address the reader as "ban", neutral tone.
- Keep technical terms in English (locator, assertion, fixture...). Prefer precise definitions; everyday analogies only for genuinely difficult concepts, at most once per concept, phrased formally.

## Standard lesson template (in this order)

1. YAML frontmatter: `description` (one sentence, double-quoted) and `icon` (Font Awesome name).
2. H1: `# Buoi X · Ten bai` (Vietnamese with diacritics).
3. Opening `{% hint style="info" %}` starting with "Sau buoi nay ban se:" listing outcomes.
4. Numbered sections `## 1.`, `## 2.`: theory with runnable code samples and tables.
5. "Loi thuong gap" section: 3-column table Trieu chung / Nguyen nhan / Cach xu ly, at least 3 rows.
6. Final section "Bai tap ve nha (45-60 phut)": numbered tasks, submission via Pull Request on branch `your-name/lesson-X`, ending with a "Checklist truoc khi nop:" block written as a Markdown task list (one `* [ ] ` item per line, blank line after the heading).

GitBook blocks: hint info for notes, success for tips, warning for warnings/anti-patterns. Command or line-by-line code explanations use 2-column tables (Lenh / Y nghia).

## Code samples and naming conventions

Naming language: all identifiers in example code and commands are English.

- Folder and file names in examples: English kebab-case (`lesson-01`, `homework-lesson-01.ts`, `login-page.ts`). Not Vietnamese-without-diacritics (`buoi-01`, `baitap-buoi1.ts`).
- Variables/functions: camelCase. Classes/interfaces: PascalCase. All English.
- Git branch names in submission instructions: `your-name/lesson-X` (concrete example: `linh/lesson-2`).
- Commit messages in examples: short English (e.g. `lesson 2: async await homework`).
- Exceptions that stay Vietnamese: comments inside code (they explain "why" to beginners) and test description strings in `test("...")`.

Sample quality:

- TypeScript, must actually run on the course practice sites: Saucedemo (account standard_user / secret_sauce, `data-test` attribute), DemoQA, The Internet, ReqRes, JSONPlaceholder, FakeStoreAPI.
- Locators prefer getByRole, getByLabel, getByPlaceholder, getByText.
- Never `page.waitForTimeout()` or XPath in samples, unless illustrating an anti-pattern with an explicit note.

Scope: the English naming rule applies to code inside lessons. The repo's own .md filenames keep the `buoi-XX-*.md` pattern (see Repo structure).

## Illustrative images

- Add a screenshot when a step depends on a UI the reader must recognize: VS Code menus and dialogs, the GitHub PR banner and form, Playwright HTML report, Trace Viewer, Codegen window, error messages shown in the editor. Do not add images for content a code block or table already shows completely (terminal output, file trees, code).
- Claude cannot capture screenshots. When a lesson needs one, insert a placeholder on its own line at the exact spot where the image belongs:
  `<!-- TODO(image): mo ta anh can chup, gom trang thai man hinh va phan can khoanh/danh dau -->`
  Description in Vietnamese, specific enough for the trainer to reproduce the screen. HTML comments are not rendered by GitBook, so a placeholder is safe to publish. Never write a visible `//TODO` in body text.
- The trainer replaces the placeholder with the image. Image files go in `.gitbook/assets/`, named `buoi-XX-<short-name>.png` (kebab-case ASCII), inserted as `![mo ta ngan](../.gitbook/assets/buoi-XX-<short-name>.png)` with Vietnamese alt text. Remove the TODO once the image is in place.
- Quality checks report every remaining `TODO(image)` placeholder in the file.

## Cross-references between sessions

When reusing earlier knowledge, cite the source ("destructuring da hoc o Buoi 1"). When mentioning a concept taught later, add a forward reference ("Buoi 9 trinh bay chi tiet playwright.config.ts").

## Editing workflow

1. Edit per the template and style rules above.
2. If pages are added/removed/renamed: update `SUMMARY.md` and the card grid in `README.md`.
3. Repo commit messages: short English (e.g. `fix async example in lesson 2`).
4. Only push after the user confirms. Pushing `main` publishes to students.

## Content scope

- These docs are public to students. Never include: trainer-only notes, in-class time allocations, homework answer keys.
- Sessions 1-8 exist. Sessions 9-15 are written incrementally from `course-outline.md`.
