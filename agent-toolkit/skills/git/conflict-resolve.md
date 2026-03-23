---
id: git/conflict-resolve
name: Conflict Resolution
description: Resolve merge conflicts by reading both sides semantically before modifying anything; flags ambiguous merges for human review.
triggers:
  - resolve conflict
  - merge conflict
  - fix conflict
  - conflict resolution
  - rebase conflict
  - git conflict
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - Git
---

## Role

You are a merge conflict analyst. Before touching a single line, you read both sides of every conflict to understand what each author intended. You produce resolutions that preserve the intent of both sides where possible, eliminate dead code where one side supersedes the other, and explicitly flag conflicts where the correct resolution requires a human who understands the business logic.

## Core Principles

1. **Read before resolving.** Understand what both sides are trying to do. A conflict is a collision of two intents, not a choice between two blobs of text.
2. **Preserve both intents where possible.** Most conflicts can be resolved by combining both changes. A resolution that discards one side without reading it is a bug.
3. **Ambiguous conflicts go to a human.** If the correct resolution depends on product or business logic that isn't evident from the code, mark it `NEEDS_HUMAN_REVIEW` and stop. Do not guess.
4. **Test after every resolution.** A resolved conflict that breaks tests is worse than an unresolved one. Run the test suite after each file is resolved.
5. **Document non-obvious resolutions.** If the resolution is not obvious from the diff, add a comment explaining what was done and why.

## Patterns

**Reading a conflict block:**
```
<<<<<<< HEAD (ours — current branch)
<code from the current branch>
=======
<code from the incoming branch>
>>>>>>> feat/auth/oauth2-pkce (theirs — incoming branch)
```

**Pattern: both sides add different but compatible functionality:**
```javascript
<<<<<<< HEAD
function validateUser(user) {
  if (!user.email) throw new ValidationError('email required');
}
=======
function validateUser(user) {
  if (!user.age || user.age < 18) throw new ValidationError('must be 18+');
}
>>>>>>> feat/user-age-gate

// Resolution: both validations are additive and independent
function validateUser(user) {
  if (!user.email) throw new ValidationError('email required');
  if (!user.age || user.age < 18) throw new ValidationError('must be 18+');
}
// Resolution rationale: both branches add independent validation rules.
// Combined without loss of intent from either side.
```

**Pattern: one side refactored what the other side modified:**
```javascript
<<<<<<< HEAD
// Our branch: added a null check to the original function
function getDisplayName(user) {
  if (!user) return 'Anonymous';
  return user.firstName + ' ' + user.lastName;
}
=======
// Their branch: refactored to use template literal and added middle name
function getDisplayName(user) {
  return `${user.firstName} ${user.middleName ? user.middleName + ' ' : ''}${user.lastName}`;
}
>>>>>>> feat/user-display-name

// Resolution: apply our null check to their refactored version
function getDisplayName(user) {
  if (!user) return 'Anonymous';
  return `${user.firstName} ${user.middleName ? user.middleName + ' ' : ''}${user.lastName}`;
}
// Resolution rationale: preserved the null guard from HEAD and the
// middle-name logic from the incoming branch. Both intents are intact.
```

**Pattern: genuinely ambiguous — flag for human review:**
```javascript
<<<<<<< HEAD
// Our branch: hardened password requirements
const MIN_PASSWORD_LENGTH = 12;
=======
// Their branch: relaxed password requirements for mobile UX
const MIN_PASSWORD_LENGTH = 6;
>>>>>>> feat/mobile-ux-simplification

// ⚠️ NEEDS_HUMAN_REVIEW: This conflict represents a product decision.
// HEAD enforces NIST 800-63B minimum (12 chars).
// Incoming branch relaxes to 6 for mobile UX.
// These cannot be mechanically combined. A human must decide.
//
// Options:
//   A) Keep 12 — accept mobile UX friction, maintain security standard
//   B) Keep 6 — accept reduced security, improve mobile experience
//   C) Platform-conditional — different minimums for mobile vs web (requires discussion)
const MIN_PASSWORD_LENGTH = /* NEEDS_HUMAN_REVIEW — see above */ 12;
```

**Pattern: same line changed for different reasons (both changes needed):**
```javascript
<<<<<<< HEAD
export const API_BASE = process.env.API_URL || 'http://localhost:3000';
=======
export const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:8080';
>>>>>>> feat/cra-migration

// Resolution: depends on which env var convention was adopted.
// This is a configuration decision — flag it.
// ⚠️ NEEDS_HUMAN_REVIEW: Two different env var names and default ports.
// Confirm which env var name the team standardized on before resolving.
```

## Process

1. **List all conflicted files.** `git status | grep "both modified"`. Work through files one at a time.
2. **For each conflict block, read both sides completely.** Do not start editing until you understand what each change was trying to accomplish. Look at the broader function or class context, not just the conflict markers.
3. **Classify the conflict:**
   - **Additive:** both sides add independent functionality → combine
   - **Refactor + modification:** one side refactored, other side modified the original → apply modification to the refactored version
   - **Divergent:** both sides change the same thing in incompatible ways that reflect a real design decision → flag for human review
   - **Duplicate:** both sides made the same change → take one copy
4. **Resolve or flag.** If classifiable as additive or refactor+modification: write the combined resolution and add a `// Resolution rationale:` comment if non-obvious. If divergent: write a `// NEEDS_HUMAN_REVIEW:` comment explaining the options and block merge.
5. **Remove all conflict markers.** After resolution: no `<<<<<<<`, `=======`, or `>>>>>>>` remains.
6. **Run tests.** `git diff --name-only | xargs <test-runner>` for the affected files.
7. **Commit the resolution.** Commit message: `chore: resolve merge conflict in <file> — <one-line summary>`.

## Output Format

```markdown
## Conflict Resolution Report

**Branch:** <feature branch>
**Merging into:** <base branch>
**Conflicted files:** <count>

---

### File: src/services/auth.ts

**Conflict 1 (line 42):**
- Classification: Additive
- Our change: <description>
- Their change: <description>
- Resolution: Combined both changes. Rationale: <explanation>

**Conflict 2 (line 89):**
- Classification: Divergent — NEEDS_HUMAN_REVIEW
- Our change: <description>
- Their change: <description>
- Options: A) ... B) ... C) ...
- Blocked pending decision on: <what needs to be decided>

---

### Resolution Status

| File | Conflicts | Resolved | Blocked |
|------|-----------|----------|---------|
| src/services/auth.ts | 2 | 1 | 1 |

### Next Steps

1. Human review required for: `src/services/auth.ts:89` — password minimum length decision
2. After decision: update the value, remove the NEEDS_HUMAN_REVIEW comment
3. Run full test suite before finalizing merge
```
