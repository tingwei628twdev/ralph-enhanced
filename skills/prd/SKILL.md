---
name: prd
description: >
  Generate a Product Requirements Document (PRD) for a new feature. Use when planning a feature, starting a new project, or when asked to create a PRD.
  Triggers on: create a prd, write prd for, plan this feature, requirements for, spec out, 幫我規劃功能, 寫需求文件, 規劃這個專案, 建立 PRD.
user-invocable: true
---

# PRD Generator

Create detailed Product Requirements Documents that are clear, actionable, and suitable for implementation.

---

## The Job

1. Receive a feature description from the user
2. Ask 3-5 essential clarifying questions (with lettered options)
3. Generate a structured PRD based on answers
4. Save to `tasks/prd-[feature-name].md`

**Important:** Do NOT start implementing. Just create the PRD. Ask only critical questions where the initial prompt is ambiguous.

---

## Step 1: Clarifying Questions

Focus on:

- **Problem/Goal:** What problem does this solve?
- **Core Functionality:** What are the key actions?
- **Scope/Boundaries:** What should it NOT do?
- **Success Criteria:** How do we know it's done?

#### Format questions like this:

```
1. What is the primary goal of this feature?
   A. Improve user onboarding experience
   B. Increase user retention
   C. Reduce support burden
   D. Other: [please specify]

2. Who is the target user?
   A. New users only
   B. Existing users only
   C. All users
   D. Admin users only

3. What is the scope?
   A. Minimal viable version
   B. Full-featured implementation
   C. Just the backend/API
   D. Just the UI
```

This lets users respond with "1A, 2C, 3B" for quick iteration.

---

## Step 2: PRD Structure

### 1. Introduction/Overview
Brief description of the feature and the problem it solves.

### 2. Goals
Specific, measurable objectives (bullet list).

### 3. User Stories

Each story needs:
- **Title:** Short descriptive name (US-001, US-002, etc.)
- **Description:** "As a [user], I want [feature] so that [benefit]"
- **Acceptance Criteria:** Verifiable checklist of what "done" means

**Format:**
```
### US-001: [Title]
**Description:** As a [user], I want [feature] so that [benefit].

**Acceptance Criteria:**
- [ ] Specific verifiable criterion
- [ ] Another criterion
- [ ] Typecheck/lint passes
- [ ] **[UI stories only]** Verify in browser using dev-browser skill
```

**Rules:**
- Each story must be small enough to complete in one focused session
- Acceptance criteria must be verifiable, not vague ("Button shows confirmation dialog" ✓ / "Works correctly" ✗)
- For any story with UI changes: always include "Verify in browser using dev-browser skill"

### 4. Functional Requirements
Numbered list:
- "FR-1: The system must allow users to..."
- "FR-2: When a user clicks X, the system must..."

### 5. Non-Goals (Out of Scope)
What this feature will NOT include.

### 6. Technical Considerations (Optional)
Known constraints, dependencies, integration points.

### 7. Success Metrics
How will success be measured?

### 8. Open Questions
Remaining questions needing clarification.

---

## Writing for Junior Developers / AI Agents

- Be explicit and unambiguous
- Avoid jargon or explain it
- Number requirements for easy reference
- Use concrete examples

---

## Output

- **Format:** Markdown (`.md`)
- **Location:** `tasks/`
- **Filename:** `prd-[feature-name].md` (kebab-case)

---

## Example PRD

```markdown
# PRD: Task Priority System

## Introduction
Add priority levels to tasks so users can focus on what matters most.

## Goals
- Allow assigning priority (high/medium/low) to any task
- Provide clear visual differentiation between priority levels
- Enable filtering and sorting by priority

## User Stories

### US-001: Add priority field to database
**Description:** As a developer, I need to store task priority so it persists across sessions.

**Acceptance Criteria:**
- [ ] Add priority column to tasks table: 'high' | 'medium' | 'low' (default 'medium')
- [ ] Generate and run migration successfully
- [ ] Typecheck passes

### US-002: Display priority indicator on task cards
**Description:** As a user, I want to see task priority at a glance.

**Acceptance Criteria:**
- [ ] Each task card shows colored priority badge (red=high, yellow=medium, gray=low)
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

## Functional Requirements
- FR-1: Add `priority` field to tasks table ('high' | 'medium' | 'low', default 'medium')
- FR-2: Display colored priority badge on each task card

## Non-Goals
- No priority-based notifications
- No automatic priority assignment

## Technical Considerations
- Reuse existing badge component with color variants

## Success Metrics
- Users can change priority in under 2 clicks

## Open Questions
- Should priority affect task ordering within a column?
```

---

## Checklist Before Saving

- [ ] Asked clarifying questions with lettered options
- [ ] Incorporated user's answers
- [ ] User stories are small and specific (completable in one session)
- [ ] Functional requirements are numbered and unambiguous
- [ ] Non-goals section defines clear boundaries
- [ ] Saved to `tasks/prd-[feature-name].md`
