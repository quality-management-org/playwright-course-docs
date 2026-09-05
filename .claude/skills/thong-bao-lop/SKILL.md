---
name: thong-bao-lop
description: Draft an announcement message for the student group chat. Use this skill when the user wants to write a notification, message, or announcement for the class or students, e.g. "soạn thông báo cho lớp", "nhắn học viên", about newly published materials, pre-reading, or the upcoming session.
---

# Class announcement

**Output language: Vietnamese, always.** The instructions below are in English only to save tokens. Every message drafted for the class must be written in natural, fluent Vietnamese; never send students an English message.

## Gather information

1. Run `git log --oneline -15` for recent changes; if the user gives a specific range (a commit, a date), use that instead.
2. Map commits to lesson titles via SUMMARY.md so the message uses proper lesson names, not filenames.

## Message format

- The message is written in Vietnamese, 4-6 sentences, ready to paste into Zalo or a group chat.
- Friendly, warm tone (this is a chat message, not documentation), but still no em dash "—".
- Structure: greet the class, what was added or updated, what students should read or prepare before the next session, link to the docs site.
- Docs site URL: ask the user if unknown; remember it for the rest of the session.

## Hand over

Present the draft and invite tone or content adjustments (schedule, homework deadline) before the user sends it. Never commit anything to the repo.
