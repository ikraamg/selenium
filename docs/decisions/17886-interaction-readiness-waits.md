# 17886. Selenium waits for interaction readiness, not just element presence

- Status: Proposed
- Discussion: https://github.com/SeleniumHQ/selenium/pull/17886

## Context

The single most common cause of flaky Selenium tests is acting on an element that is present but
not yet ready: the page is still hydrating, a spinner is still animating, a modal is fading in over
the button, a list is re-rendering under the pointer. WebDriver's own actionability checks run at
the moment of the command — click scrolls into view and verifies the in-view centre point is
pointer-interactable — and then either succeeds or throws. They do not wait, and they do not
consider whether the *page* has finished doing work.

Users therefore write the wait themselves. Every binding gives them a polling primitive and, at
best, conditions that check element flags:

| Binding    | Current behavior |
|------------|------------------|
| Java       | `ExpectedConditions.elementToBeClickable` — `isDisplayed()` and `isEnabled()` only |
| Python     | `expected_conditions.element_to_be_clickable` — `is_displayed()` and `is_enabled()` only |
| Ruby       | No conditions library; generic `Selenium::WebDriver::Wait#until { }` block |
| .NET       | No conditions library; generic `DefaultWait.Until(lambda)` (`ExpectedConditions` was moved out of the project) |
| JavaScript | `until.elementIsVisible` and `until.elementIsEnabled` separately; no combined condition |

No binding can express "the page has stopped changing" at all. `document.readyState` and the
`pageLoad` timeout say nothing about a single-page application, and nothing observes timers,
in-flight requests, animation loops, or DOM mutation. The gap is filled with `sleep`, with
retry-on-`ElementClickInterceptedException`, or by migrating to a tool that has the wait built in —
Playwright and Cypress both auto-wait on readiness, and users consistently name this as the reason
they leave.

Three forces are in tension. Readiness is genuinely undecidable from outside the browser: a page
that polls forever is never quiet, so any answer is a heuristic. Selenium's high-level API must stay
protocol-neutral ([17670](17670-bidi-implementation-boundaries.md)), so the mechanism cannot leak.
And the semantics must be identical in five bindings, or the wait becomes another source of
cross-binding divergence rather than a fix for one.

## Decision

Selenium ships interaction-readiness waits as supported, protocol-neutral API, with the semantics
defined once and implemented once.

**1. Readiness is modelled in three composable layers.**

- *Pending work* — a live ledger of timers, intervals, animation frames, `fetch`, `XMLHttpRequest`
  and WebSocket activity for the document. Long-delay timers, long-period intervals, and periodic
  tasks observed to complete repeated invocations with no DOM mutation, no network dispatch and no
  storage write are classified inert and do not block. Clocks and keep-alive polls therefore do not
  prevent quiescence.
- *DOM settledness* — no *meaningful* mutation for a settle window. Meaningful excludes attribute
  churn that cannot affect layout, and excludes mutation confined to a region annotated inert or
  detected as periodic noise. Running CSS animations and transitions count as activity. Waiting can
  be scoped to a subtree.
- *Element actionability* — visible, enabled (including `aria-disabled` and the `fieldset`/`legend`
  exception), editable when the planned interaction writes, in the viewport, not obstructed at its
  interaction point, and not moving. Which checks apply depends on the interaction: `type` and
  `clear` additionally require editable; `drop` and `screenshot` do not require enabled.

**2. Two supported entry points, both on the driver, both protocol-neutral.** Names follow each
language's casing convention:

```
driver.waitForDomSettled(root?, timeout, settle, requirePendingQuiet)  → readiness result
driver.waitUntilActionable(element, interaction, timeout)              → readiness result
```

`waitUntilActionable` composes the layers: settle the document, then wait for the element,
scrolling it into view once if it starts outside the viewport. The pending-work ledger is an input
to the other two (`requirePendingQuiet`), not a third public entry point — "are there pending
timers?" is not a question users should have to ask.

**3. A timeout raises the binding's existing timeout error, and the error carries the diagnosis.**
Consistency with `WebDriverWait` matters more than the convenience of a boolean. The error message
names what was still active: the blocking work classes, the still-mutating regions, or for an
element the specific failing check and the obstructing element. "Timed out after 10s" is what makes
current waits unhelpful; this is the main thing the feature has to fix. A non-blocking inspection
call returning the same snapshot without waiting is available for diagnostics and for users
building their own conditions.

**4. Existing commands do not change.** `click`, `send_keys` and friends keep their current
behavior. Making the wait implicit alters the timing of every existing test and is a separate
decision; it is not foreclosed, but it is not made here.

**5. One implementation, shared by all bindings.** The readiness oracle is a single JavaScript
implementation in `javascript/atoms/`, injected into the page; bindings are thin wrappers that
marshal arguments and translate the result. Heuristics this delicate — what counts as meaningful
mutation, when a spinner is inert, how obstruction is hit-tested — will not stay identical across
five hand-written ports, and any divergence is a behavioral difference users hit as flakiness.

**6. The oracle is installed as a preload script over the BiDi session, and degrades explicitly.**
Instrumenting timers and network requires running before page script, so pending-work tracking
requires a BiDi session and a preload registration. DOM settledness and actionability do not, and
fall back to injection into the current document. When pending-work state cannot be observed the
result says so rather than silently reporting quiet; the two entry points remain available.

**7. This is a prototype of a protocol feature.** The heuristics exist in page script only because
the browser does not expose the answer. The implementation is written as a reference for a proposed
BiDi `quiescence` module and its limits are documented, not hidden: effects performed
asynchronously by a periodic callback are misattributed, and workers, `MessageChannel`,
server-sent events, WebTransport, IndexedDB and canvas-only animation frames are not tracked.
Selenium takes this to the WebDriver BiDi working group; if it lands, the bindings keep their API
and drop the polyfill.

## Considered options

**Where readiness is decided**

1. *Leave it to users* (Rejected) — the status quo. Every user re-derives the same heuristics badly,
   most settle on `sleep`, and Selenium carries the reputation cost for flakiness it could remove.
2. *Add more expected conditions* (Rejected) — conditions are built from the same element flags that
   are already insufficient, and cannot see timers, requests or mutation at all. This does not reach
   the problem.
3. *Wait for readiness inside `click`/`send_keys`* (Rejected for now) — what users ultimately want,
   and where Playwright sits, but it silently changes the timing and failure modes of every existing
   test. Ship the explicit wait first, then decide implicit adoption on evidence.
4. *Explicit driver-level readiness waits* (Accepted).

**Where the semantics live**

5. *Each binding implements the oracle in its own language* (Rejected) — five ports of a heuristic
   this fiddly diverge immediately, and the divergence surfaces to users as cross-binding flakiness.
6. *Specify the semantics in this record, implement per binding* (Rejected) — prose cannot pin down
   "meaningful mutation" tightly enough to make five implementations agree.
7. *One shared JavaScript implementation, thin binding wrappers* (Accepted) — one place to fix a
   misclassification, and a single artifact to hand to the BiDi working group.

**How it reaches the page**

8. *Specify the BiDi module first and wait for browsers* (Rejected as a precondition) — the right
   end state, but it leaves users with nothing for years and gives the working group no evidence.
   Pursued in parallel, not instead.
9. *CDP* (Rejected) — Chromium-only, and being retired as an implementation mechanism.
10. *BiDi preload script, with fallback injection where the layer allows it* (Accepted).

**What a timeout does**

11. *Return a result record, never raise* (Rejected) — inconsistent with every other wait in
    Selenium, and a not-ready result is easy to ignore by accident.
12. *Raise the binding's timeout error carrying the diagnosis, plus a non-blocking inspection call*
    (Accepted) — familiar control flow, and the diagnosis is not lost.

## Consequences

- Users get a supported answer to "wait until this is safe to click" for the first time, and a
  failure message that names the spinner, the overlay, or the request still in flight.
- Selenium takes on maintenance of a heuristic that will be wrong sometimes. The policy knobs
  (settle window, inert thresholds, ignore patterns, cooperative inert-region annotation) are the
  escape hatch, and misclassifications are bug reports we will receive and have to tune.
- The waits are best-effort by construction. This must be stated in the user documentation, not
  only in the code, or the API will be read as a guarantee.
- A page that never stops working never settles. `waitForDomSettled` on such a page always reaches
  its timeout, which is correct behavior and will be reported as a bug.
- The shared atom is loaded and injected by each binding, so each needs the packaging wiring to ship
  a JavaScript resource. Bindings that already ship atoms have this; the rest gain a build step.
- Registering the preload lazily on the BiDi session is per-binding plumbing that must not affect
  navigation: registration failure is logged and swallowed, never raised.
- Follow-up decisions this makes necessary: whether interactions eventually wait implicitly; whether
  `ExpectedConditions.elementToBeClickable` and its equivalents are deprecated in favour of
  `waitUntilActionable`; and the shape of the BiDi `quiescence` module proposal.

## Appendix

A working reference implementation exists for Python: the oracle as
`javascript/atoms/quiescence.js`, registered as a BiDi preload script and exposed as
`driver.wait_for_dom_settled` and `driver.wait_until_actionable`, with roughly 1,500 lines of
behavioral tests covering mutation classification, periodic-noise detection, shadow DOM and frame
boundaries, obstruction hit-testing, motion stability, and the per-interaction check matrix. It is
linked from this record's PR as evidence that the semantics above are implementable and testable,
not as the proposed final API shape — in particular it returns a result record rather than raising,
which decision 3 changes.
