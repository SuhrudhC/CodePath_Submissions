# CodePath_Submissions

# Contribution [#]: 1, Week 1, Issue: "initial view state is confusing and set to different values in multiple files"

**Contribution Number:** [1]  
**Student:** Suhrudh Chivukula
**Issue:** [[GitHub issue link]  ](https://github.com/openSenseMap/frontend/issues/706)
**Status:** [Phase I] [Complete]

---

## Why I Chose This Issue

I chose this issue because it addresses application state management and map initialization, offering an excellent opportunity to learn how  applications are built and how state is managed across modules in a frontend project. Tracking down variable declarations and understanding the map's components aligns perfectly with my goal of improving my debugging and code-tracing skills in larger, more valuable codebases.

Additionally, this is marked as a "good first issue" and "help wanted," which means it is an accessible entry point. It still provides a tangible improvement to the user interface, and fixing it will help me get familiar with openSenseMap's contribution workflow while improving the initial user experience.

---

## Understanding the Issue

The default coordinates and zoom level for the map's initial load are defined in multiple different files with conflicting values. This lack of a "single source of truth" in the application's configuration causes the map framework to initialize using default variables and leaves the user looking at a blank patch of ocean.

### Expected Behavior

When a user first loads the application, the map should read its default viewport state from a single, centralized configuration file.

### Current Behavior

Upon initial load, the map incorrectly centers itself in the middle of the Indian Ocean. This buggy behavior was specifically reproduced and reported on an iPhone 13 mini running iOS 18.7.3 in the Safari browser.

### Affected Components

[Which parts of the codebase are involved?]

- The primary map component/view (where the map instance is initialized).

- Global state management stores or context files where the view state is held.

- Configuration or constants files where default latitude, longitude, and zoom levels are defined.

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
