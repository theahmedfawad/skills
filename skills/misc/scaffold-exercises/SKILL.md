---
name: scaffold-exercises
description: Create exercise directory structures with sections, problems, solutions, and explainers that pass linting. Use when user wants to scaffold exercises, create exercise stubs, or set up a new course section.
---

# Scaffold Exercises

Create exercise directory structures that pass the course repo's exercise-structure linter, then commit with `git commit`.

## Directory naming

- **Sections**: `XX-section-name/` inside `exercises/` (e.g., `01-peripheral-drivers`)
- **Exercises**: `XX.YY-exercise-name/` inside a section (e.g., `01.03-uart-with-dma`)
- Section number = `XX`, exercise number = `XX.YY`
- Names are dash-case (lowercase, hyphens)

## Exercise variants

Each exercise needs at least one of these subfolders:

- `problem/` - student workspace with TODOs
- `solution/` - reference implementation
- `explainer/` - conceptual material, no TODOs

When stubbing, default to `explainer/` unless the plan specifies otherwise.

## Required files

Each subfolder (`problem/`, `solution/`, `explainer/`) needs a `readme.md` that:

- Is **not empty** (must have real content, even a single title line works)
- Has no broken links

When stubbing, create a minimal readme with a title and a description:

```md
# Exercise Title

Description here
```

If the subfolder has code, it also needs a `main.c` (>1 line). But for stubs, a readme-only exercise is fine.

## Workflow

1. **Parse the plan** - extract section names, exercise names, and variant types
2. **Create directories** - `mkdir -p` for each path
3. **Create stub readmes** - one `readme.md` per variant folder with a title
4. **Run lint** - the course repo's exercise-structure linter to validate
5. **Fix any errors** - iterate until lint passes

## Lint rules summary

The exercise-structure linter typically checks (adapt to your repo's linter):

- Each exercise has subfolders (`problem/`, `solution/`, `explainer/`)
- At least one of `problem/`, `explainer/`, or `explainer.1/` exists
- `readme.md` exists and is non-empty in the primary subfolder
- No `.gitkeep` files
- No `speaker-notes.md` files
- No broken links in readmes
- No project-specific run commands hardcoded in readmes
- `main.c` required per subfolder unless it's readme-only

## Moving/renaming exercises

When renumbering or moving exercises:

1. Use `git mv` (not `mv`) to rename directories - preserves git history
2. Update the numeric prefix to maintain order
3. Re-run lint after moves

Example:

```bash
git mv exercises/01-peripheral-drivers/01.03-uart-with-dma exercises/01-peripheral-drivers/01.04-uart-with-dma
```

## Example: stubbing from a plan

Given a plan like:

```
Section 05: Interrupt Handling
- 05.01 Introduction to Interrupts
- 05.02 GPIO Interrupts (explainer + problem + solution)
- 05.03 Nested Interrupts
```

Create:

```bash
mkdir -p exercises/05-interrupt-handling/05.01-introduction-to-interrupts/explainer
mkdir -p exercises/05-interrupt-handling/05.02-gpio-interrupts/{explainer,problem,solution}
mkdir -p exercises/05-interrupt-handling/05.03-nested-interrupts/explainer
```

Then create readme stubs:

```
exercises/05-interrupt-handling/05.01-introduction-to-interrupts/explainer/readme.md -> "# Introduction to Interrupts"
exercises/05-interrupt-handling/05.02-gpio-interrupts/explainer/readme.md -> "# GPIO Interrupts"
exercises/05-interrupt-handling/05.02-gpio-interrupts/problem/readme.md -> "# GPIO Interrupts"
exercises/05-interrupt-handling/05.02-gpio-interrupts/solution/readme.md -> "# GPIO Interrupts"
exercises/05-interrupt-handling/05.03-nested-interrupts/explainer/readme.md -> "# Nested Interrupts"
```
