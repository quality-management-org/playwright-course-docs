---
name: viet-bai-hoc
description: Workflow for writing a new lesson document for the Playwright course. Always use this skill whenever the user asks to write, generate, draft, or create materials for any session, e.g. "viết Buổi 9", "generate bài API testing", "soạn tài liệu phase 3", even if they do not literally say "new lesson".
---

# Writing a new lesson

Follow the steps in order. Do not skip any. All lesson content is written in Vietnamese per CLAUDE.md.

## Step 1: Gather context

1. Read `course-outline.md` and locate the requested session. If it is not in the outline, stop and ask the user.
2. Read one recently finished lesson (prefer same phase) as the reference for voice and depth.
3. Determine the filename: `phase-N/buoi-XX-short-name.md`, kebab-case ASCII.

## Step 2: Present an outline before writing

List the planned numbered sections (title of each, key points, planned code samples, proposed frontmatter icon). Wait for the user to approve the outline before writing the full content.

## Step 3: Write using the 6-part template

Follow the exact order defined in CLAUDE.md (frontmatter, H1, opening hint "Sau buổi này bạn sẽ:", numbered sections, "Lỗi thường gặp" table, "Bài tập về nhà (45-60 phút)" with the "Checklist trước khi nộp:" line).

Content requirements:

- Documentation-guide register per CLAUDE.md. Never use the em dash "—".
- Code samples must run on the practice site(s) listed in the outline for that session. No `page.waitForTimeout()`, no XPath, except as clearly-labeled anti-pattern examples.
- English naming conventions for all identifiers, filenames in examples, branch names (`your-name/lesson-X`); Vietnamese for code comments and test description strings.
- Back-reference earlier sessions when reusing knowledge; forward-reference later sessions for concepts not yet taught.

## Step 4: Update structure

1. Add the SUMMARY.md entry in the correct phase group with the Vietnamese diacritics title.
2. Add the matching card to the `<table data-view="cards">` grid in README.md.

## Step 5: Self-check and hand over

Run the single-file quality checklist from the `kiem-tra-tai-lieu` skill against the new file and report results. Do not commit; wait for the user to review and confirm.
