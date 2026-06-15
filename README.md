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

Standard README flow (.env copy, uv sync, npm install), but three snags. No Google Cloud access (granted via the project Discord), so I used CI's forked-repo trick — fake GOOGLE_* env vars and pytest -m "not require_repo_secrets". Every test also failed with a misleading AttributeError: module 'evaluate' has no attribute 'run_langsmith_evaluation'; real cause was ALL_PROXY=socks5h://... set without socksio installed, breaking httpx during a conftest import — fixed by unsetting the proxy vars. Lastly, git push fails (no GitHub creds in this shell); branch is committed locally and needs a push from an authenticated terminal.

### Steps to Reproduce

1. Load the app with the LLM and email sender mocked.
2. Burst POST /api/query (20x) and POST /api/feedback (5x), resetting the limiter between.
3. Observed: all 20 /api/query return 200, zero 429s; /api/feedback returns 200 x3 then 429. Limiter works on feedback, not on the costly query endpoint.

### Reproduction Evidence

- **Commit showing reproduction:** [[Link to commit in your fork]](https://github.com/SuhrudhC/tenantfirstaid_codepath/commit/ae78a5c661f1b5bc)
- **Screenshots/logs:** /api/query x20 -> {200: 20} (0 of 429); /api/feedback x5 -> {200: 3, 429: 2}. Bug reproduced 2/2 trials.
- **My findings:** The Limiter has no default limits, so only routes that opt in are throttled — and only /api/feedback does. /api/query, the one endpoint that costs money per call (Vertex AI RAG + Gemini), was never given a limit.

---

## Solution Approach

### Analysis

Root cause is in backend/tenantfirstaid/app.py: the Limiter has no default_limits (lines 20–24), /api/feedback opts in (lines 58–68), but /api/query is registered plain (line 55) with no limit. Not a break — the limit was just never wired to the expensive endpoint.

### Proposed Solution

Wrap the class-based view the same way feedback is wrapped: limiter.limit("10 per minute")(ChatView.as_view("chat")). Confirm the exact number with mentors (possibly env-configurable), and verify production keys off the real client IP (X-Forwarded-For/ProxyFix) behind the DO proxy so traffic doesn't share one bucket.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** /api/query hits Vertex AI RAG + Gemini per call and has no limit, risking cost/availability abuse. It should 429 over a per-IP budget like /api/feedback. Login is out of scope (anonymous tenant tool).

**Match:** Near-copy of the feedback limit (app.py:58–68) and its test test_rate_limiting_returns_429 (test_app.py:91–101).

**Plan:** [Step-by-step implementation plan]
1. Pick the limit
2. wrap ChatView.as_view("chat")
3. Add 429 + within-limit tests
4. Run make check and re-run the repro
5. Optional Turnstile/hCaptcha only if needed.

**Implement:** Phase III on branch feature-rate-limit-api-query. PR link: TBD

**Review:** No CONTRIBUTING.md; follow the PR template (type = Bug Fix, Closes #<n>, tests = Yes) and pr-check.yml checks (ruff, ty, pytest). Branch follows the feature-* convention.

**Evaluate:** Done when the new 429 and within-limit tests pass, make check is green, and the repro flips from zero 429s to throttled.

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
