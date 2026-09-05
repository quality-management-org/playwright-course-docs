---
name: kiem-tra-tai-lieu
description: Quality-check a lesson before publishing, and verify repo structure consistency. Always use this skill when the user asks to check, review, QA, or audit a lesson or the whole docs repo, e.g. "bài này ổn chưa", "kiểm tra hộ tôi", "sẵn sàng đăng chưa", "có trang nào bị thiếu không", or wants SUMMARY.md compared against actual files.
---

# Documentation checks

Two modes. Pick per the request; if the user is vague ("kiểm tra hộ tôi"), run both.

## Mode 1: Single-lesson quality check

Read the specified file and grade each criterion. Report as a table: Criterion / Pass or Fail / exact location (line number or section name). Report only; do not fix unless asked afterwards.

1. Complete 6-part template in order: frontmatter (description + icon), H1 "Buổi X · ...", opening hint "Sau buổi này bạn sẽ:", numbered sections, "Lỗi thường gặp" (3-column table), "Bài tập về nhà (45-60 phút)" with "Checklist trước khi nộp:".
2. Style: no em dash "—"; no exclamations, colloquial language, or emoji in body text; neutral "bạn" address; content in Vietnamese.
3. Code samples: valid TypeScript/Playwright; locators prefer getByRole/getByLabel/getByPlaceholder/getByText; no `waitForTimeout`, no XPath (except labeled anti-patterns); English identifiers and example filenames; branch names `your-name/lesson-X`; Vietnamese comments and test descriptions.
4. Cross-references: every "đã học ở Buổi X" / "Buổi Y sẽ trình bày" matches `course-outline.md`.
5. Structure: the file has a SUMMARY.md entry and a README.md card.
6. Images: list every remaining `<!-- TODO(image): ... -->` placeholder with its line number and description, so the trainer knows which screenshots are still missing. A lesson with open placeholders is not ready to announce as final.

## Mode 2: Repo-wide structure sync

Compare three sources: actual .md files under `phase-*/`, entries in `SUMMARY.md`, cards in `README.md`. Report three kinds of drift:

1. Files missing from SUMMARY.md (page invisible on GitBook).
2. SUMMARY.md entries pointing to nonexistent files.
3. Lessons in SUMMARY.md with no README.md card.

Propose fixes per item and wait for confirmation before editing.
