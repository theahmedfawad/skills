---
name: implement
description: "Implement a piece of work based on a PRD or set of issues."
disable-model-invocation: true
---

Implement the work described by the user in the PRD or issues.

Use /tdd where possible, at pre-agreed seams.

Build regularly (a warning-clean compile is your fastest feedback — treat warnings as errors), run static analysis (cppcheck / clang-tidy) and single test files regularly, and the full test suite once at the end.

Once done, use /review to review the work.

Commit your work to the current branch.
