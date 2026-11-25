# CLAUDE.md - Project context for Claude Code

## Project

Student-facing course materials for "Automation Testing tu Zero den Hero" (Playwright + TypeScript), aimed at manual testers with no prior TypeScript/Playwright experience. 15 sessions, 4 phases, 8 weeks. Session-by-session outline: `course-outline.md` at repo root.

Published via GitBook Git Sync: pushing to `main` goes live to students within about one minute.

**All document content is written in Vietnamese.** Only this file, skill files, and everything inside code samples follow English conventions described below.

## Repo structure (hard rules)

The published GitBook space is the `docs/` folder (configured by `gitbook-docs.yaml`). Edit lessons ONLY inside `docs/`. The root-level `phase-1/`, `phase-2/`, `phase-3/`, `README.md` and `SUMMARY.md` are an older copy that students do not see; do not edit them.

- `docs/README.md`: homepage with a card grid (`<table data-view="cards">`) linking to lessons. Adding a lesson requires adding its card, and the card text must match the lesson content.
- `docs/SUMMARY.md`: sidebar table of contents. GitBook ONLY renders pages listed here. Any add/remove/rename of a .md file must update it.
- Lesson folders: `docs/phase-1-nen-tang-typescript/`, `docs/phase-2-playwright-co-ban/`, `docs/phase-3-nang-cao/`. Filenames are kebab-case ASCII, and some were renamed by GitBook (e.g. Buoi 5 is `buoi-5-locators-chon-phan-tu-dung-cach/README.md`, Buoi 6 is `buoi-6-actions-assertions-and-auto-wait.md`). Always check the actual path in `docs/SUMMARY.md`.
- Cross-lesson links are relative to the file inside `docs/` (e.g. `../phase-1-nen-tang-typescript/buoi-01-setup-typescript.md`). Content copied from the root copy keeps root paths, which GitBook turns into `/broken/pages/...`: fix them before pushing.
- Edits made in the GitBook editor arrive as `GITBOOK-N` commits: pull before editing.
- `.gitbook.yaml`, `gitbook-docs.yaml`: sync config, do not modify.
- `course-outline.md`: the session outline (internal reference, written in Vietnamese). Keep it out of SUMMARY.md.
- Display titles (H1 and SUMMARY link text) are Vietnamese with full diacritics.
- Never rename existing lesson .md files without the user's approval: it breaks GitBook URLs already shared with students. When a rename is approved, update SUMMARY.md, the README card and every link to the old path.

## Writing style (most important rules)

- Documentation-guide register: declarative sentences, neutral instructions ("Chay lenh sau", "Tao file X"). No colloquial/spoken language, no exclamations, no jokes, no emoji in body text.
- NEVER use the em dash character "—". Replace with commas, colons, parentheses, or split the sentence. Only the plain hyphen "-" is allowed (e.g. "Buoi 4-9").
- Address the reader as "ban", neutral tone.
- Keep technical terms in English (locator, assertion, fixture...). Prefer precise definitions; everyday analogies only for genuinely difficult concepts, at most once per concept, phrased formally.
- Lessons are training material, not a reference manual: each concept starts from a concrete problem, code is built step by step, and the result of running it is shown. Text around a code example is 1-2 sentences: what the code does, the link to earlier knowledge, the consequence.
- Headings are terminology-based noun phrases. No "Dat van de" section.
- No section-number references ("muc 3"). Do not cite other sessions in prose ("da hoc o Buoi 9", "(Buoi 6)", "Buoi 13 trinh bay..."): it reads as spoken style. State the knowledge directly; refer to content inside a lesson by name.
- When a passage carries several separate points (conditions, consequences, steps, rules), present them as a bullet list or numbered steps instead of one dense paragraph.

## Standard lesson template (in this order)

Buoi 9 is the reference format. Buoi 1-8 still follow an older template (opening hint, "Loi thuong gap", checklist) until they are migrated.

1. YAML frontmatter: `description` (one sentence, double-quoted) and `icon` (Font Awesome name).
2. H1: `# Buoi X · Ten bai` (Vietnamese with diacritics).
3. Opening: a 2-sentence paragraph directly under the H1 stating the general need the session addresses. No "Sau buoi nay ban se" hint, no roadmap table, no recap of earlier lessons.
4. Numbered sections `## 1.`, `## 2.`: concepts with code examples (as images, see below) and tables.
5. A "Thuc hanh tai lop" section: steps and design questions, no solution code.
6. `## N. Tom tat`: a summary table directly under the heading.
7. `## N+1. Bai tap ve nha` (no time label): `### Bai 1: ...`, `### Bai 2: ...` stating goals and what to verify. Do not prescribe locators, assertions or code skeletons. No "Checklist truoc khi nop" and no separate "nop bai" task.

No "Loi thuong gap" and no "Cau hoi on tap" sections: important error messages go into the body or a hint where the concept is taught.

GitBook blocks: hint info for notes, success for tips, warning for warnings/anti-patterns. Use explanation tables only for the lesson's core subject (config settings, CLI options being taught, new constructs such as fixture syntax) and for reference lists. Do not add line-by-line tables for code that an image already makes clear, or for auxiliary one-liners.

## Code samples and naming conventions

Naming language: everything inside code samples is English.

- Folder and file names in examples: English kebab-case (`lesson-01`, `homework-lesson-01.ts`, `login-page.ts`). Not Vietnamese-without-diacritics (`buoi-01`, `baitap-buoi1.ts`).
- Variables/functions: camelCase. Classes/interfaces: PascalCase.
- `test.describe("...")` titles start with a capital letter ("Login", "Remove product"); `test("...")` titles are lowercase sentences describing the expected result.
- Comments, `test("...")` and `test.describe("...")` titles, error messages and `--list` output are English. Lesson prose that quotes a test name uses the English name. Phase 3 is converted; Buoi 1-8 still have Vietnamese comments and test titles until they are migrated.
- Git branch names in submission instructions: `your-name/lesson-X` (concrete example: `linh/lesson-2`).
- Commit messages in examples: short English (e.g. `lesson 2: async await homework`).

Sample quality:

- TypeScript that actually runs. Locators prefer getByRole, getByLabel, getByPlaceholder, getByText. Never `page.waitForTimeout()` or XPath, unless illustrating an anti-pattern with an explicit note.
- Practice site from Phase 3 onward: only Toolshop, `https://practicesoftwaretesting.com` (`data-test` attribute) and its API `https://api.practicesoftwaretesting.com` (Swagger at `/api/documentation`). Earlier lessons may still use Saucedemo, DemoQA, The Internet, JSONPlaceholder or FakeStoreAPI.
- Toolshop data rules: data and every id are regenerated daily, so never hard-code ids (look them up by name). The shared account `customer@practicesoftwaretesting.com` / `welcome01` locks after 3 wrong passwords: negative login cases use an unregistered email. Do not assert exact counts of shared-account data. API requests send `Accept: application/json`, otherwise validation errors come back as 404.
- Single browser: `playwright.config.ts` keeps only the `chromium` project. No multi-browser config, no separate `api` project, no `--project` flags. API tests set their base URL per file with `test.use({ baseURL: apiUrl, extraHTTPHeaders: { Accept: "application/json" } })`.
- Imports from Buoi 10 onward use the tsconfig path aliases taught at the end of Buoi 9: `@api/*`, `@fixtures/*`, `@pages/*`, `@test-data/*`, `@typings/*` (folder `types/`), `@utils/*`, including same-folder imports. No relative `../` imports of project files and no `baseUrl` in tsconfig samples.
- Not taught in this course: network mocking (`page.route`), multi-browser runs.

Scope: the English naming rule applies to code inside lessons. The repo's own .md filenames keep the `buoi-XX-*.md` pattern (see Repo structure).

## Illustrative images

- Phase 3 code examples are shown as VS Code-style images, not as text code blocks: file tab, line numbers, Dark+ colors, an optional highlighted token, and a TERMINAL panel with the command and its output when the example is run (a PROBLEMS panel when showing an editor error). Long files are split into several images with continuous line numbers. Terminal output must come from a real run, or from output that is deterministic (such as `--list`); otherwise show the code only. Plain code blocks remain for terminal commands and CLI output.
- Add a screenshot when a step depends on a UI the reader must recognize: VS Code menus and dialogs, the GitHub PR banner and form, Playwright HTML report, Trace Viewer, Codegen window, DevTools.
- When Claude cannot produce an image (for example a DevTools screenshot), insert a placeholder on its own line at the exact spot where the image belongs:
  `<!-- TODO(image): mo ta anh can chup, gom trang thai man hinh va phan can khoanh/danh dau -->`
  Description in Vietnamese, specific enough for the trainer to reproduce the screen. HTML comments are not rendered by GitBook, so a placeholder is safe to publish. Never write a visible `//TODO` in body text.
- Image files go in `docs/.gitbook/assets/`, named `buoi-XX-<short-name>.png` (kebab-case ASCII), inserted as `![mo ta ngan](../.gitbook/assets/buoi-XX-<short-name>.png)` with Vietnamese alt text. Remove the TODO once the image is in place.
- Quality checks report every remaining `TODO(image)` placeholder in the file.

## Session order

Session order must match `docs/SUMMARY.md`: Phase 3 is Buoi 9 config, Buoi 10 API testing, Buoi 11 hooks and fixtures. Lesson prose does not cite other sessions by number (see Writing style); Buoi 9 and Buoi 10 still contain such references until they are revised.

## Editing workflow

1. Edit per the template and style rules above.
2. If pages are added/removed/renamed: update `SUMMARY.md` and the card grid in `README.md`.
3. Repo commit messages: short English (e.g. `fix async example in lesson 2`).
4. Only push after the user confirms. Pushing `main` publishes to students. The user cannot preview GitBook locally and reviews changes on the live site after a push.

## Content scope

- These docs are public to students. Never include: trainer-only notes, in-class time allocations, homework answer keys.
- Sessions 1-11 exist. Phase 3 ends at Buoi 11 (there is no Buoi 12 lesson). Later sessions are written incrementally from `course-outline.md`.
