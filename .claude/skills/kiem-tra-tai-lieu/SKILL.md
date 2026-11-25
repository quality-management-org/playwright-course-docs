---
name: kiem-tra-tai-lieu
description: Quality-check a lesson before publishing, and verify repo structure consistency. Always use this skill when the user asks to check, review, QA, or audit a lesson or the whole docs repo, e.g. "bài này ổn chưa", "kiểm tra hộ tôi", "review lại toàn bộ", "sẵn sàng đăng chưa", "có trang nào bị thiếu không", or wants SUMMARY.md compared against actual files.
---

# Documentation checks

Two modes. Pick per the request; if the user is vague ("kiểm tra hộ tôi", "review lại toàn bộ"), run both.

The criteria below reflect decisions the trainer made after CLAUDE.md was written. Where CLAUDE.md's "Standard lesson template" disagrees (opening hint, "Lỗi thường gặp", "(45-60 phút)", checklist), these criteria win.

## Mode 1: Single-lesson quality check

Read the specified file(s) and grade each criterion. Report as a table: Criterion / Pass or Fail / exact location (line number or section name). Report only; do not fix unless asked afterwards.

### 1. Lesson structure

Applies in full to Buổi 9 onward. Buổi 1-8 still use the older template; for them only check the items marked (all).

- (all) YAML frontmatter with `description` (one sentence, double-quoted) and `icon` (Font Awesome name).
- (all) H1 is `# Buổi X · Tên bài`, Vietnamese with diacritics, and X matches the file's position in `docs/SUMMARY.md`.
- Opening: a 2-sentence paragraph directly under the H1 stating the general need the session addresses. Fail if there is a `{% hint %}` "Sau buổi này bạn sẽ:" list, a roadmap table, a recap of earlier lessons ("Sau Buổi 8, project đã có..."), or a restatement of the practice site.
- Body sections are numbered `## 1.`, `## 2.` ... and start directly with the first concept. Fail on a section titled "Đặt vấn đề".
- Contains a "Thực hành tại lớp" section with steps and design questions, and no solution code.
- Closing sections, in order: `## N. Tóm tắt` (summary table directly under it, no H3), then `## N+1. Bài tập về nhà`.
- (all) The homework heading has no time label: fail on "(45-60 phút)" or "(45–60 phút)".
- Homework uses `### Bài 1: ...`, `### Bài 2: ...` headings that state goals and what to verify. Fail on: a "Checklist trước khi nộp" block, a separate "nộp bài qua Pull Request" task, prescribed locators, assertions, function signatures or code skeletons, or contrived requirements added only to exercise an earlier concept.
- Fail on any "Lỗi thường gặp" section or "Câu hỏi ôn tập" section.

### 2. Writing style

- (all) No em dash "—" anywhere. Only the plain hyphen "-" is allowed.
- (all) No exclamations, colloquial or spoken phrasing, jokes or emoji in body text. Neutral "bạn" address. Content in Vietnamese; technical terms stay in English.
- Headings are terminology-based noun phrases ("Khái niệm fixture", "Xây dựng LoginPage"), not conversational sentences.
- No section-number references ("mục 3", "Buổi 5, mục 12"). Other lessons are cited as "Buổi X" only; content inside the lesson is referred to by name.
- Training-material shape: each concept starts from a concrete problem, code is built incrementally, and the result of running it is shown. Flag passages that read like a reference manual (a final code dump followed by a long line-by-line table).
- Auxiliary one-liners (helper commands that are not the lesson's subject) get one introductory sentence and the result, no "Lệnh / Ý nghĩa" table.

### 3. Code samples

- (all) Valid TypeScript/Playwright. Locators prefer getByRole, getByLabel, getByPlaceholder, getByText. No `waitForTimeout()` and no XPath, except labeled anti-patterns.
- (all) English identifiers and example filenames (kebab-case). Branch names `your-name/lesson-X`.
- Everything inside code is English: comments, `test("...")` and `test.describe("...")` titles, error messages, and `--list` output shown in images. Lesson prose that quotes a test name uses the English name. Applies to Phase 3 now; for Buổi 1-8, report Vietnamese code text as "not yet migrated" rather than Fail.
- Practice site for Buổi 9 onward: only Toolshop, `https://practicesoftwaretesting.com` and its API `https://api.practicesoftwaretesting.com`. Fail on Saucedemo, DemoQA, The Internet, ReqRes, JSONPlaceholder, FakeStoreAPI.
- Toolshop data rules: no hard-coded product, category or brand ids (data and ids are regenerated daily; look ids up by name); never a wrong password for the shared account `customer@practicesoftwaretesting.com` (it locks after 3 attempts; negative login cases use an unregistered email); no assertions on exact counts of shared-account data; API requests send `Accept: application/json`.
- Single browser: config has only the `chromium` project. Fail on extra projects (firefox, webkit, mobile, a separate `api` project) and on `--project` flags.
- Imports from Buổi 10 onward: every import of a project file uses the tsconfig path aliases `@api/*`, `@fixtures/*`, `@pages/*`, `@test-data/*`, `@typings/*` (folder `types/`), `@utils/*`, including same-folder imports (`@pages/base-page`). Fail on `from "./..."` or `from "../..."` for project files, and on `baseUrl` in tsconfig samples. Library imports (`@playwright/test`, `dotenv`) stay unchanged.
- Topics not taught: network mocking (`page.route`, `route.fulfill`), multi-browser config. Fail if a lesson teaches them.

### 4. Code presentation (Phase 3 onward)

- Code examples are shown as VS Code-style images, not as text code blocks: file tab with file name, line numbers, Dark+ colors, optional highlighted token, and a TERMINAL panel with the real command and output when the example is run. Images live in `docs/.gitbook/assets/buoi-XX-<short-name>.png` and are referenced as `![mô tả tiếng Việt](../.gitbook/assets/buoi-XX-<short-name>.png)`.
- Report, per lesson, the count of remaining ` ```typescript ` / ` ```json ` blocks (should be 0) and the count of images. Plain blocks for terminal commands and CLI output may remain.
- Text around an image is 1-2 sentences: what the code does, the link to earlier knowledge, and the consequence. Flag line-by-line explanation tables ("Dòng / Ý nghĩa", "Phần / Ý nghĩa") that the image and output already make clear.
- Every referenced image file exists.

### 5. Cross-references and links

- Every "đã học ở Buổi X" / "Buổi Y trình bày" matches the current session order in `course-outline.md` and `docs/SUMMARY.md`. Phase 3 is: Buổi 9 config, Buổi 10 API testing, Buổi 11 hooks and fixtures. Fail on references to a Buổi 12 lesson or to removed content.
- Every relative `.md` and image link resolves to an existing file inside `docs/`, and no link contains `/broken/pages/`.

### 6. Repo structure

- The file has a `docs/SUMMARY.md` entry and a `docs/README.md` card, and the card text matches the lesson's current content.

### 7. Image placeholders

- List every remaining `<!-- TODO(image): ... -->` placeholder with its line number and description, so the trainer knows which screenshots are still missing. A lesson with open placeholders is not ready to announce as final.

## Mode 2: Repo-wide structure sync

Compare three sources: actual .md files under `docs/phase-*/`, entries in `docs/SUMMARY.md`, cards in `docs/README.md`. The root-level `phase-*/`, `SUMMARY.md` and `README.md` are an unpublished old copy: ignore them. Report:

1. Files missing from SUMMARY.md (page invisible on GitBook).
2. SUMMARY.md entries pointing to nonexistent files.
3. Lessons in SUMMARY.md with no README.md card, or cards linking to nonexistent files.
4. Links in any lesson that point to renamed or deleted lesson files.

Propose fixes per item and wait for confirmation before editing.

## Running scripted checks

Python on this Windows machine prints with cp1252 by default and crashes on Vietnamese text. Run scripts with `PYTHONIOENCODING=utf-8` (Bash) or `$env:PYTHONIOENCODING="utf-8"` (PowerShell).
