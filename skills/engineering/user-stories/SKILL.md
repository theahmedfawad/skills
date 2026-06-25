---
name: user-stories
description: Generate a standalone, detailed user-story backlog from a system or module description for embedded systems and software projects. Use when asked to write user stories, create stories from a description, elicit requirements as user stories, break down a module into stories, or develop a product backlog. Outputs one file per story with acceptance criteria, test scenarios, and priority, plus an index. Supports quick (one-shot) and thorough (interactive) modes. NOT the same as `/to-prd` (which embeds a lightweight one-liner story list inside a PRD) or `/to-issues` (which splits a plan/PRD into grabbable vertical-slice issues) — reach for `user-stories` when the detailed backlog is itself the deliverable.
---

# User Stories

Generate user stories from a system or module description following industry best practices.

## Workflow

### 1. Gather Input

Identify the system or module description. This may be:
- A description document (e.g., `module_description.md`)
- A conversation with the user describing what they want to build
- An existing codebase that needs stories written retroactively

If the description is ambiguous or incomplete, run the `/grilling` skill to interview the user before proceeding. Focus the interview on:
- Who are the users/roles interacting with this system?
- What are the key capabilities the system must provide?
- What are the constraints (hardware, timing, memory, safety)?
- Are there features explicitly excluded?
- What is the target environment (MCU/RTOS, host tooling, etc.)?

`/grilling` handles the questioning discipline — one question at a time, recommended answers, exploring the codebase where it can answer a question itself.

### 2. Identify Roles

Extract all distinct user roles from the description. Common embedded systems roles:
- Embedded software developer (integrating the module)
- Application developer (using the module's API)
- System integrator (combining modules)
- End user (interacting with the final product)
- Test engineer (validating behavior)

Use the most specific role that fits. Prefer "embedded software developer" over generic "developer" when the context is embedded.

### 3. Decompose into Stories

Break the system description into independent, valuable stories. Apply the INVEST criteria:

- **Independent** - Stories can be developed and tested in any order
- **Negotiable** - Details can be refined during implementation
- **Valuable** - Each story delivers value to the identified role
- **Estimable** - Scope is clear enough to estimate effort
- **Small** - Each story is implementable in a single sprint/iteration
- **Testable** - Clear criteria determine when the story is done

Group stories into categories:
- **Core Functionality** - The fundamental capabilities
- **Advanced Features** - Optional or enhanced capabilities
- **Architecture & Design** - Non-functional qualities (portability, configurability, scalability)

### 4. Prioritize

Assign priority to each story:

| Priority | Meaning |
|----------|---------|
| **Critical** | System cannot function without this. Must be implemented first. |
| **High** | Important capability that most users need. Implement after critical. |
| **Medium** | Valuable enhancement. Can be deferred if needed. |
| **Low** | Nice to have. Implement if time permits. |

Dependencies influence priority: if Story B depends on Story A, Story A must be equal or higher priority.

### 5. Write Stories

Write each story using the format defined in [story-format.md](references/story-format.md).

Key rules:
- Use the canonical form: **As a [role], I need [capability] so that [benefit]**
- The capability must be concrete and specific, not vague
- The benefit must explain *why* this matters to the role
- Acceptance criteria must be testable (given-when-then where possible)
- Each story gets a unique sequential ID (zero-padded 3-digit: 001, 002, ...)

### 6. Generate Story Index

After all stories are written, generate a README.md index containing:
- Story index table (ID, title, file link, description) grouped by priority
- Category summary describing what each group of stories covers
- Dependency flow diagram showing story relationships
- Recommended implementation order organized into phases
- Cross-cutting concerns that apply to all stories

### 7. Identify Gaps

After drafting all stories, review for completeness:
- Is every capability from the description covered by at least one story?
- Are there implied capabilities not explicitly stated (error handling, initialization, cleanup)?
- Are non-functional qualities captured (performance, portability, testability)?
- Are there edge cases or boundary conditions that warrant their own story?

Present any gaps to the user for confirmation before adding stories.

## Output Structure

Write each story to its own file in the output directory:

```
stories/
├── README.md                          # Story index and overview
├── 001_story_title.md                 # Individual story files
├── 002_story_title.md
└── ...
```

File naming: `{ID}_{title_in_snake_case}.md`

## Modes

### Quick Mode
When the user wants stories generated without interactive refinement:
1. Read the description
2. Generate all stories in one pass
3. Present the complete set for review

### Thorough Mode
When the user wants to collaborate on story development:
1. Read the description
2. Run the `/grilling` skill to interview the user about roles, scope, and priorities
3. Present story categories and proposed story list for approval
4. Write stories incrementally, soliciting feedback after each category
5. Review for gaps and refine

Default to **thorough mode** unless the user requests quick generation or provides a very clear, complete description.
