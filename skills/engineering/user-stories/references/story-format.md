# User Story Format

## Template

Every user story file must follow this structure exactly:

```markdown
# User Story {ID}: {Title}

## Story
As a **{role}**, I need **{capability}** so that **{benefit}**.

## Description
{1-3 sentences expanding on the story. Explain what the module must do and why this capability matters. Provide enough context for a developer to understand the intent without reading the full system description.}

## Priority
**{Critical|High|Medium|Low}** - {Brief justification for the priority level}

## Acceptance Criteria

### Functional Criteria
{Numbered list of observable behavioral requirements. Each criterion must be testable and unambiguous. Use "shall" for mandatory requirements.}

### Technical Criteria
{Numbered list of implementation-level constraints. Cover API behavior, timing, memory, side effects, and integration concerns.}

## Test Scenarios

### Scenario N: {Descriptive Name}
- **Given**: {Initial state and configuration}
- **When**: {Action or sequence of actions with specific timing/values}
- **Then**: {Expected outcome with specific observable results}

## Definition of Done
- [ ] Unit tests written and passing for all acceptance criteria
- [ ] Code implements the described functionality
- [ ] Edge cases documented and tested
- [ ] Code review completed
- [ ] Documentation updated
```

## Writing Guidelines

### Story Statement
- Always use: **As a [role], I need [capability] so that [benefit]**
- Bold the role, capability, and benefit for readability
- The capability is what the system must do, not how
- The benefit explains business or user value

Good: As a **firmware developer**, I need **to detect when a sensor reading exceeds a threshold** so that **my application can trigger an alarm before damage occurs**.

Bad: As a **user**, I need **the system to work** so that **it is useful**.

### Acceptance Criteria
- Write functional criteria from the user's perspective (what they observe)
- Write technical criteria from the developer's perspective (how it behaves at the API level)
- Use "shall" for mandatory behavior
- Each criterion maps to one or more test cases
- Avoid implementation details (algorithms, data structures) unless they are constraints

### Test Scenarios
- Cover the happy path first, then edge cases
- Include specific values (timestamps, thresholds, durations) - not vague descriptions
- Each scenario must be independently executable
- Name scenarios descriptively: "Timeout After Maximum Duration" not "Test Case 3"
- Include boundary conditions: minimum, maximum, off-by-one, zero, overflow
- Include error/failure scenarios where applicable

### Definition of Done
- Start with the standard checklist items shown in the template
- Add story-specific items for complex stories (e.g., "Both active-high and active-low polarities tested")
- Keep it actionable - each item is a concrete task someone can check off

## Anti-Patterns to Avoid

- **Too large**: If a story has more than 8-10 acceptance criteria, split it
- **Too vague**: "The system should be fast" is not testable
- **Implementation-driven**: Stories describe *what*, not *how*
- **Missing benefit**: Every story must explain *why* it matters
- **Compound stories**: "As a user, I need X and Y" should be two stories
- **No scenarios**: Stories without test scenarios cannot be verified
- **Duplicate coverage**: Two stories should not test the same behavior
