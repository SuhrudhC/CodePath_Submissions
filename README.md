# CodePath_Submissions

# Contribution [#]: 1, Week 1, Issue: "initial view state is confusing and set to different values in multiple files"

**Contribution Number:** [1]  

**Student:** Suhrudh Chivukula

**Issue:** [[GitHub issue link]] (https://github.com/openSenseMap/frontend/issues/706)

**Status:** [Phase 3] [Complete]

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

Test case 1 — Boundary (10th succeeds, 11th fails)
I tested the exact edge of the limit rather than just "over" it. The 10th request in a window must return 200 and only the 11th must return 429. This matters because an off-by-one error in flask_limiter's counter — or in how I applied it — could block legitimate users one request too early. The test burns through 9 requests, asserts the 10th is still 200, then asserts the 11th is 429.

Test case 2 — Per-IP isolation
The limiter has to key off the client's IP address, not a single global counter. If it were global, the first person to hit the limit would lock out every other user on the site simultaneously. I tested this by exhausting IP 10.0.0.1's quota (11 requests), confirming it gets 429, then immediately sending a request from IP 10.0.0.2 and asserting it still gets 200. Flask's test client accepts environ_base={"REMOTE_ADDR": "..."} to simulate different source addresses.

Test case 3 — Window reset
After the limit fires, the quota should refill when the time window rolls over. I simulated this by calling limiter.reset() (which clears all counters, the same effect as a new window opening) after exhausting the quota, then confirming the next request succeeds again. Without this test, a bug that permanently blocks an IP after one abuse incident would go undetected.

Test case 4 — Regression: the two endpoints are independent
I tested both directions: exhausting /api/query's 10-per-minute quota should not consume any of /api/feedback's separate 3-per-minute quota, and vice versa. This catches a class of bug where a shared counter or shared storage key would cause the two limits to bleed into each other. Both the existing feedback test and two new cross-endpoint tests confirm the counters stay isolated.

Test case 5 — Env-var wiring
The limit value is read from os.environ at startup into a module-level constant called QUERY_RATE_LIMIT. The test imports that constant and asserts it matches os.getenv("QUERY_RATE_LIMIT", "10 per minute"). This verifies the indirection is actually in place — if someone hard-coded the value instead of reading the env var, this test would catch it.



### Integration Tests

- [ ] Integration scenario 1 - I ran all 10 allowed requests in a window through the real Flask routing stack with the LLM mocked, asserting every one returns 200 with mimetype == "text/plain". This confirms the rate limiter doesn't interfere with normal chat traffic — the streaming response, the stream_with_context wrapper, and the limiter all coexist without the Flask request context getting corrupted. (This was the source of a subtle failure during development: if you don't drain the streaming response body with resp.get_data() before making the next request, Flask raises "Popped wrong request context" because the generator still holds a reference to the old context.)
- [ ] Integration scenario 2 - When the limit fires, the response needs to be well-formed enough for the frontend and any proxy layer to handle it correctly. I checked that the 429 body is non-empty and that the content type is text/html. One thing this surfaced: flask_limiter does not emit a Retry-After header by default in the version this project uses — that requires headers_enabled=True on the Limiter constructor. That's a separate improvement worth adding but is outside the scope of this fix, and the test documents it explicitly so it isn't forgotten.

### Manual Testing

I ran the reproduction script (backend/repro_rate_limit.py) before and after the fix against the live Flask app with mocks. Before: /api/query x20 → {200: 20}, zero 429s in both trials — the bug. After: /api/query x20 → {200: 10, 429: 10}, confirmed in both trials — the fix. The server logs also changed visibly: before, the flask-limiter: ratelimit exceeded at endpoint: chat log line never appeared; after the fix, it fires exactly 10 times per trial. I also ran make --keep-going check (ruff format, ruff lint, ty typecheck, pytest) to confirm nothing else broke — all checks passed except one pre-existing test collection failure in test_measure_evaluator_variance.py caused by a missing socksio package under a SOCKS proxy environment, which is unrelated to this change.

---

## Implementation Notes

### Week [1] Progress

The first thing I did was understand exactly what was broken before touching any code. I ran a reproduction script that fired 20 requests at /api/query and 5 at /api/feedback against the real Flask app with the LLM and email sender mocked out. The results were clear: every single /api/query request came back 200, zero rejections, while /api/feedback correctly throttled after 3. That confirmed the bug was real and gave me a baseline to flip once the fix was in.

From there I traced the behavior to a single gap in backend/tenantfirstaid/app.py. The flask_limiter library was already imported and a Limiter object already existed — the infrastructure was there. The problem was that when /api/query was registered on line 55, it was handed to Flask as a bare class-based view with no limit attached. The only route that ever called limiter.limit(...) was /api/feedback on line 58. So the fix wasn't about adding a new library or changing architecture — it was about connecting the existing tool to the endpoint that needed it.

The actual code change was two lines in app.py: capture the result of ChatView.as_view("chat"), wrap it with limiter.limit(QUERY_RATE_LIMIT), then pass that wrapped view to add_url_rule. I also made the limit value read from an environment variable (QUERY_RATE_LIMIT) rather than hard-coding "10 per minute", so the budget can be tuned in production without pushing new code. After the fix, the repro script flipped from {200: 20} to {200: 10, 429: 10} in both trials.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** backend/tenantfirstaid/app.py — Core fix. Two lines changed: wrap ChatView.as_view("chat") with limiter.limit(QUERY_RATE_LIMIT) before passing it to add_url_rule, and read the limit value from the QUERY_RATE_LIMIT environment variable so it's tunable without a redeploy.
backend/tests/test_app.py — Added test_query_within_limit_returns_200 and test_query_rate_limiting_returns_429 to the existing TestQueryRoute class, modeled directly on the feedback endpoint's existing rate-limit test.
backend/.env.example — Documented QUERY_RATE_LIMIT=10 per minute so contributors and operators know the variable exists and what format it takes (flask_limiter syntax).
backend/tests/test_query_rate_limit.py — New file with 13 tests across 6 categories covering boundary, per-IP isolation, window reset, env-var wiring, regression, and response shape.
- **Key commits:** ae78a5c — Reproduction script and solution plan. Documents the bug with evidence before touching any fix code.
70185c2 — The actual fix: rate limit applied to /api/query, two tests added, env var documented.
266b541 — Comprehensive verification suite confirming the fix holds across all meaningful scenarios.
- **Approach decisions:** I chose to wrap the view function at registration time (limiter.limit(...)(ChatView.as_view("chat"))) rather than putting a decorators class attribute on ChatView itself, because the limit value comes from an env var resolved at startup and this keeps the configuration centralized in app.py alongside the feedback limit — a reader can see both limits in the same file without jumping into the view class. I chose 10 per minute as the default because a real conversation involves a message every 10–30 seconds at most, so 10 per minute is generous for humans but a hard wall for scripts firing as fast as the network allows.

---

## Pull Request

**PR Link:** github.com/codeforpdx/tenantfirstaid/pull/376

**PR Description:** /api/query is the most expensive thing this app does — every request fans out to Vertex AI RAG plus a Gemini completion. Right now it has no rate limit at all, even though flask_limiter is already set up and used on /api/feedback. Anyone (or any script) can hit it as fast as they want, which is a real cost and availability risk.

This PR wires the existing limiter up to /api/query the same way it's already used on /api/feedback. It defaults to 10 requests per minute per IP, which should be plenty for a normal back-and-forth conversation but stops a script from hammering the endpoint. The limit is also configurable through a QUERY_RATE_LIMIT env var, so it can be tuned in production without a code change if it turns out to be too tight or too loose.

I also wrote a small script (backend/repro_rate_limit.py) that reproduces the bug by bursting both endpoints and comparing results — before the fix /api/query returned 200 for all 20 requests with zero rejections, and after the fix it correctly starts returning 429s past the 10th request.

Closes #168. Tests: 2 new tests in test_app.py, plus 13 new tests in test_query_rate_limit.py covering the boundary, per-IP isolation, window reset, env-var wiring, and regression against the feedback endpoint's existing limit.

**Maintainer Feedback:**
- No feedback yet — PR was just opened. CI checks are running against it. Will fill out the form below when feedback is received.
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

I got hands-on with flask_limiter for the first time. Specifically, the gotcha is that wrapping a class-based view (ChatView.as_view(...)) with limiter.limit() must occur before the view is registered with add_url_rule, since the decorator returns a new wrapped function rather than mutating the view in place. I also learned that flask_limiter keys its counters by whatever the key_func returns (here, get_remote_address), which is what makes per-IP isolation work, and that testing that isolation requires faking REMOTE_ADDR through Flask's test client (environ_base={"REMOTE_ADDR": "..."}) rather than just hitting the endpoint normally. On the testing side, I learned that streaming Flask responses (stream_with_context) need their body fully drained with resp.get_data() in a test before the next request, or Flask throws a confusing "Popped wrong request context" error, which cost me real debugging time before I traced it to the actual cause.

### Challenges Overcome

The single biggest time-sink wasn't the fix itself; it was getting the test environment running at all. Every test failed with AttributeError: module 'evaluate' has no attribute 'run_langsmith_evaluation', which sounded like a totally unrelated import problem. After digging through the traceback, I found the real cause buried underneath: my sandbox had ALL_PROXY set to a SOCKS proxy, but the socksio package wasn't installed, so building an httpx client during an autouse pytest fixture failed silently and surfaced as that misleading attribute error several layers up. Unsetting the proxy env vars fixed it. The second real challenge was the streaming-response test context bug mentioned above, easy to miss because it only shows up when you loop multiple requests against a streaming endpoint without draining each response first.

### What I'd Do Differently Next Time

I'd check for proxy/network env vars in my environment before touching the test suite, since that one red herring ate more time than the actual root-cause investigation of the bug itself. I'd also reproduce the bug with a dedicated script even earlier in the process than I did. Having that script ready meant the "did the fix actually work" question at the end was a 30-second rerun instead of a fresh investigation, and I'd want that safety net in place from minute one next time, not partway through. Lastly, I'd ask upfront whether GitHub push access would be available in my working environment, since that turned into a multi-step detour (trying gh, checking keychain, eventually using a scoped PAT) that could have been resolved earlier if I'd surfaced the question before starting Phase III instead of after.

---

## Resources Used

- flask-limiter documentation — reference for the library already in use in this codebase
- .github/workflows/pr-check.yml in this repo — showed me how CI handles missing GCP credentials for forked PRs, which is what unblocked running the test suite locally without real Google Cloud access
- Issue #168 and the maintainer's re-scoping comment on it — clarified that the issue's original framing (auth/session endpoints) was stale and the real ask was just rate limiting /api/query
- The existing /api/feedback rate-limit implementation and its test in test_app.py — used directly as the template/pattern for both the fix and the new tests, rather than designing something from scratch
