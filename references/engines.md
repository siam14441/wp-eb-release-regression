# Engines — how findings are actually produced

Axes tell you *where* to look. Engines are *how* you find something once you are looking. Reach for
these when a sweep is coming back clean but you do not yet believe the release is clean.

## 1. Differential — compare two states, change one thing

The highest-yield technique in this codebase. Nearly every confirmed finding came from a comparison.

- **Version differential.** Same action on N-1 and N. This is what separates a regression from
  pre-existing behaviour, and it is required before calling anything a regression.
- **Merge-result differential.** Test the merge, not the branch. Build a detached merge commit and
  deploy it, and separately deploy the baseline, so the only variable is the change.
- **Artifact differential.** Byte-compare a file from the tag against the same file in the ZIP:
  `diff <(git -C "$REPO" show "$TAG:path/File.php") "$INSTALLED/path/File.php"`.
- **Live A/B in the browser.** The cleanest isolation available: remove one `<link>` element, re-read
  computed styles, put it back. If the bug flips on and off with nothing else changed, the cause is
  proven, not inferred. This is how the Astra Customizer conflict was root-caused.
- **Peer differential.** One block does something correctly and its sibling does not. Where a fix was
  applied to one instance of a pattern, grep for every other instance.

## 2. Spec-versus-code diff

A ticket claims three behaviours; grep proves only one exists. Three features from one release
(pause-out-of-viewport, lazy loading, mobile poster) were **entirely unimplemented** and were caught
only by grepping both repos for `IntersectionObserver` and lazy attributes.

Read the release card and the in-repo feature docs, list the claims, then grep for the mechanism each
claim requires. A claim with no mechanism is a finding.

## 3. Incomplete-fix hunting

The most productive place to look is a fix that already shipped.

- Did the fix cover **every** call path? A sold-count gate was applied to the REST branch but not the
  block's own render, so the data stayed public on page 1 and vanished on Load More.
- Did it cover **every instance** of the pattern? Never stop at the one the ticket names.
- Does the fix hold under the *positive* case, or did it break legitimate use while blocking abuse?
- Was the same class of bug fixed elsewhere before and left unfixed here?

## 4. Fault injection

Do not wait for a failure to occur naturally; cause it.

- **mu-plugin probe** on `pre_http_request` to force each failure shape (`WP_Error`, an API error
  envelope, an empty body, an undecodable body) and observe what the plugin does with each.
- **Browser `fetch` override** narrowed to a single action, so only the request under test fails.
- **`page.addInitScript`** to fail the first N calls then serve a deterministic mock — the way to
  measure retry timing precisely.
- **DevTools request blocking** as a genuine network-level failure, distinct from a JS override.
  Cross-confirming a finding with both methods is what makes it undeniable.
- **Forced corruption**, then check self-recovery: set a container's inline height wrong and see
  whether the layout heals.

Always remove probes afterwards and verify they are gone.

## 5. Isolated harnesses

When the WordPress context is noise rather than signal, take the code out of it.

- **Node harness** re-implementing a module against both pre-fix and post-fix source, driving the exact
  inputs. Fast, deterministic, and it proves behaviour rather than reading it.
- **PHP harness** feeding the exact bug input to both file versions pulled via `git show`.
- **Core source tracing.** Read the shipped `wp-includes/js/dist/blocks.js` to confirm what Gutenberg
  actually does with undeclared attributes on *this* WP version, rather than assuming.

## 6. Doc-as-oracle

The in-repo docs (`.claude/docs/`, stripped from the ZIP) state contracts the code is supposed to
honour — the baseline-asset contract, the free/pro extension points, the render pipeline. Any
divergence between a documented contract and observed behaviour is a finding against one or the other.
Say which.

## 7. Boundary and shape probing

For anything that accepts input, enumerate the shapes rather than testing the happy value:
`true` / `"true"` / `1` / `"1"` / `"yes"` / `[true]` / `{"0":true}` / case variants / duplicate keys /
`__proto__` / absent entirely. Prior runs found real bugs at `"true"`-as-string and at absent-key
paths that the happy value never reached.

For anything stored: check the length limit. A transient key exceeding `option_name`'s VARCHAR(191)
silently never reads back, so the cache never works and every page view hits the API.

## Combining them

The strongest findings stack engines: a version differential locates *when* it broke, an isolated
harness proves *what* breaks, fault injection shows *how often* a user hits it, and a live A/B proves
the cause. A finding backed by two independent methods is one nobody argues with.
