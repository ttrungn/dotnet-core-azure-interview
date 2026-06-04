# Code Review Practices

## What It Is

Code review is the practice of having one or more engineers read, question, and approve a change before it merges to the main branch. It's a quality gate, a knowledge-sharing mechanism, and a teaching moment in one — and in modern .NET teams it's usually mediated by a pull request on GitHub or Azure DevOps.

A code review is **not** a stamp ("LGTM" in 30 seconds). It's not a style fight either. It's a focused conversation about a specific diff, ranked by what matters: correctness, security, operational risk, then maintainability, then style.

A small before/after captures the spirit:

```text
// BAD comment on a PR
"Nit: use var here."

// GOOD comment on the same PR
"suggestion: This loop allocates a new `List<Invoice>` per iteration when
called from `ReconcileAsync` — that runs every 30 seconds. Consider 
moving the allocation outside the loop or using `ArrayPool<T>` if the 
size is bounded. Profile attached: 12% of CPU on staging."
```

Same reviewer, same line. One is noise; the other is a real engineering signal.

## Why It Exists

Before pull-request-based reviews, .NET teams reviewed code by walking to a colleague's desk, by mandatory pair programming, or by post-deployment code reading sessions. All of these had value but none scaled, and none left an audit trail. Distributed teams, regulatory requirements (SOC 2, PCI, ISO 27001 require "evidence of peer review"), and the rise of GitHub/Azure DevOps PR workflows made async, written review the default.

Review exists because:

- **No engineer is right alone** — fresh eyes catch what the author missed.
- **Knowledge concentrates** — review spreads ownership of every file across the team.
- **Bugs are cheaper before merge** — finding a SQL injection in PR is a comment; finding it in production is an incident.
- **Audits require it** — most compliance frameworks mandate evidence of two-person review for changes touching production systems.
- **Mentoring happens here** — juniors learn from seniors and vice versa through real-code conversations.

## When To Use It

**Use code review for:**

- Every change that touches `main` or any release branch — no exceptions for "small" fixes.
- Infrastructure-as-code changes (Bicep, Terraform, Helm charts).
- Configuration changes that affect production (`appsettings.json`, Key Vault references, feature flags).
- Database migrations — these are often the riskiest changes in the PR.
- Security-sensitive code paths (auth, crypto, secrets handling, PII).
- API contract changes — breaking changes need a second pair of eyes.
- Documentation changes that capture process or runbooks.

**Skip or fast-track review when:**

- The change is a single-character typo fix in documentation (some teams allow self-merge for `.md`-only changes).
- The change is auto-generated and verified (Dependabot bumps with passing tests).
- It's a hotfix during an active incident — review still happens, but post-merge with a written post-mortem.
- You're working alone on a personal project — review yourself the next day instead.

**Avoid review anti-patterns:**

- Mandatory N approvals where N > 2 — diminishing returns, slows everything.
- Reviewing the same PR for weeks — split it or merge what's ready.
- Reviewing only after the change is "done" — early design review on a draft PR is much cheaper.

## Why It Is Important

In production .NET systems, code review is the **last line of human defense** before changes affect customers. It catches what tests don't (intent, security context, operational impact) and creates the social contract that nobody ships alone.

It drives:

- **Lower defect escape rate** — published industry data and internal team studies (Microsoft, Google) consistently show review catches 60–80% of defects that would otherwise reach production.
- **Shared ownership** — when three engineers have reviewed `BillingService.cs`, three engineers can on-call for it.
- **Faster onboarding** — junior engineers ramp via review feedback, not just docs.
- **Audit evidence** — every PR with approval is a record of two-person change control.
- **Architectural drift prevention** — reviews catch "we don't do it that way here" patterns before they spread.
- **Better tests** — reviewers ask "where's the test for this?" and that question alone raises coverage.
- **Cross-team consistency** — review is where conventions become culture.

## How It's Used in C# / .NET

**Platform features (GitHub):**

| Feature | What it does |
|---|---|
| `CODEOWNERS` | Auto-requests reviews from owners of changed paths. |
| Required reviewers | Branch protection rule blocking merge until specific people/teams approve. |
| Required status checks | Merge blocked until CI (`dotnet build`, tests, analyzers, CodeQL) passes. |
| Suggestion blocks | Reviewer proposes a literal code change author can commit with one click. |
| Resolved threads | Reviewer marks discussion resolved; author can't dismiss it silently. |
| Draft PRs | Author signals "early feedback wanted, not ready to merge." |
| `auto-merge` | Merges automatically when all conditions are met. |
| Linked issues (`Fixes #123`) | Auto-closes the issue when PR merges. |

**Platform features (Azure DevOps Repos):**

| Feature | What it does |
|---|---|
| Branch policies | Required reviewers, minimum approvals, work-item linking, build validation. |
| Required reviewers per path | Like CODEOWNERS — folders auto-require specific reviewers. |
| Reviewer scopes | "Required" vs "Optional" vs "Auto-completion observers." |
| Squash merge with linked work items | Commits get traceable history back to PBIs/bugs. |
| Suggested edits | Inline code suggestions accepted by author. |
| Auto-complete | Merge automatically when policies pass. |

**Pull request shape that gets reviewed well:**

- **Small** — under 400 lines of diff (excluding generated files, lock files). Larger PRs review less thoroughly per line.
- **Focused** — one logical change. Refactor + behavior change = two PRs.
- **Self-described** — title is conventional (`feat: add coupon code validation`), description explains *why*, links the issue, lists what was tested.
- **Pre-checked** — CI green, analyzers clean, tests added, before submitting for review.

**Conventional Comments format** (https://conventionalcomments.org/) — gives every comment an explicit type so authors know whether it blocks merge:

```text
suggestion: Use AsNoTracking() here — this query is read-only and 
the change tracker overhead shows up in your perf trace.

question: What happens if the customer is deleted between the auth 
check and the order creation?

issue (blocking): The Stripe webhook secret is being compared with 
==, which is timing-attack-vulnerable. Use CryptographicOperations
.FixedTimeEquals.

nit: Trailing whitespace on line 47.

praise: Nice extraction of the validator. Reads way better now.

thought (non-blocking): Down the road we might want to push this 
into IPricingStrategy so we can A/B test pricing experiments.
```

The labels (`suggestion`, `issue`, `nit`, `praise`, `question`, `thought`, `chore`) make the author's job clear: `issue (blocking)` must be resolved before merge; `nit` and `thought` are FYIs.

**What every .NET reviewer should check (in order):**

1. **Does this match the intent?** Does the diff actually solve the linked issue?
2. **Correctness** — edge cases, null inputs, error paths, concurrency, timezone, encoding.
3. **Security** — input validation, authorization checks, secrets handling, injection, deserialization.
4. **Tests** — does new behavior have tests? do they assert observable behavior?
5. **Operational impact** — logging, metrics, migrations, feature flags, rollback path.
6. **Performance** — N+1 queries, unbounded allocations, blocking calls, missing `ConfigureAwait`.
7. **Architecture** — does it belong in this layer? does it duplicate existing code?
8. **Readability** — names, sizes, comments. Catch these *after* the above are clear.

## Advantages

- **Catches bugs before production** — cheaper than incidents.
- **Spreads knowledge** — no single point of failure in the team.
- **Mentors juniors** — reviews are sustained, contextual learning.
- **Maintains architectural coherence** — catches drift early.
- **Audit evidence** — every PR is a paper trail of who approved what.
- **Raises test discipline** — "where's the test?" becomes a habit.
- **Builds team trust** — when everyone reviews everyone, you ship together.

## Disadvantages

- **Slows delivery** — review queues create lead time; bad processes amplify this.
- **Bikeshedding risk** — reviewers can spend more on style than on correctness.
- **Inconsistent quality** — different reviewers catch different things.
- **Power dynamics** — a strict senior can demotivate juniors; a passive reviewer rubber-stamps.
- **Burnout** — heavy reviewer load is exhausting.
- **Async lag** — distributed teams across timezones can wait a day per round-trip.
- **Encourages overlarge PRs** — when review feels expensive, authors batch changes, making the next review even worse.

## Common Mistakes

### 1. PRs that are too large

```text
PR #847: "Refactor billing + add Stripe webhook + fix invoice numbering"
Files changed: 84
+2,340 / -1,100
```

**Fix** — split before opening. One concern per PR. Aim for under 400 lines of diff. If a refactor is needed for a feature, ship the refactor first as its own PR, then the feature.

### 2. Comments that focus on style instead of substance

```text
Reviewer: "Use var here." (x40 in one PR)
Result: PR approved, but a SQL injection in line 312 was missed.
```

**Fix** — let `.editorconfig` and `dotnet format` handle style. Reviewer attention is finite — spend it on correctness, security, operational impact.

### 3. "LGTM" without reading

```text
Approved 6 minutes after opening a 600-line PR touching auth code.
```

**Fix** — set a minimum read time expectation in the team. If a reviewer doesn't have 30 minutes for a 600-line auth PR, they shouldn't approve it. Better: split the PR.

### 4. Vague feedback

```text
Reviewer: "This feels wrong."
Author: ??? 
```

**Fix** — be specific. "This pattern bypasses our standard tenant filter — see [docs/multitenancy.md]. Suggest using `_db.Orders.ForCurrentTenant()` here."

### 5. Reviewing for personal preference

```text
Reviewer: "I'd write this with a foreach instead of LINQ."
Author: "It works correctly and performance is identical."
Reviewer: "Yes but I'd write it differently."
```

**Fix** — preference is not a review comment. If the team agrees on a convention, encode it (analyzer, `.editorconfig`, or written team standard). If not, accept the author's choice.

### 6. Blocking on questions that should be discussions

```text
Reviewer: "Why this approach instead of pattern X?"
[PR sits for 3 days]
```

**Fix** — tag the comment as `question:` (non-blocking) unless the answer might change the approach. For architectural debates, take it offline (15-minute call) and post the decision back to the PR.

### 7. Ignoring CI signals

```text
PR has 3 approvals. CodeQL alert: SQL injection. Build: red (analyzer error).
Reviewer comment: "looks good, merging once you fix the lint."
```

**Fix** — branch protection should make this impossible. Required status checks for build, tests, analyzers, security scans. If CI is red, merge is blocked regardless of approvals.

### 8. Stale PR rot

```text
PR opened 14 days ago. 3 review rounds. Author's branch is now 60 commits behind main.
Merge conflicts in 12 files.
```

**Fix** — keep PRs alive. If review is delayed, the author re-bases daily. If the PR is genuinely stuck, close it with a note and re-open when ready. Old PRs are negative-value.

## Best Practices

- **Keep PRs under ~400 lines of diff** — the smaller, the better the review.
- **One concern per PR** — refactor, feature, fix; each is its own PR.
- **Write a useful description** — what changed, why, what was tested, how to verify.
- **Self-review before requesting** — read your own diff line by line first.
- **Use draft PRs** for early feedback on direction before polish.
- **Adopt Conventional Comments** so blocking vs informational feedback is unambiguous.
- **Use suggestion blocks** — one-click acceptance speeds the cycle.
- **Resolve threads only after the change is made**, never silently dismiss.
- **Review in priority order**: correctness → security → tests → ops → perf → readability → style.
- **Push style to tooling** — analyzers and `dotnet format` should make style debates unnecessary.
- **Acknowledge good work** — `praise:` comments matter. Reviews shouldn't only be critical.
- **Set response SLAs** — first review within 4 working hours; second-round within 2.
- **Use `CODEOWNERS` / required reviewers** — make sure the right humans see the right changes.
- **Merge yourself, ideally** — if branch protection allows, the author merging signals readiness and ownership.
- **Track review metrics** — time-to-first-review, PR size distribution, rework rate. Use them to improve process, not to police people.

## Related Concepts

- [testing-quality/clean-code.md](clean-code.md) — clean code is easier to review well.
- [testing-quality/refactoring.md](refactoring.md) — most refactor opportunities are spotted in review.
- [testing-quality/static-analysis.md](static-analysis.md) — automate what doesn't need human review.
- [testing-quality/unit-testing.md](unit-testing.md) — "where's the test?" is the most common review comment.
- [testing-quality/integration-testing.md](integration-testing.md) — review should check that integration tests cover real seams.
- [devops/ci-cd-pipelines.md](../devops/ci-cd-pipelines.md) — review gates and CI gates work together.
- [aspnet-core/authentication-and-authorization.md](../aspnet-core/authentication-and-authorization.md) — auth changes deserve security-focused review.
- [data-access/migrations.md](../data-access/migrations.md) — migrations are the highest-risk part of most PRs.

## Real-World Usage

**GitHub-hosted ASP.NET Core API** — `CODEOWNERS` routes `/src/Payments/**` to the payments team, `/infra/**` to the platform team. Required reviewers + required status checks (build, tests, CodeQL, analyzers) make it impossible to merge a red PR. Conventional Comments are documented in `CONTRIBUTING.md`.

**Azure DevOps with multi-stage pipelines** — branch policy on `main` requires 2 approvals, build validation, work-item linking, and resolution of all comments. Suggested edits + auto-complete with squash merge keep the history clean. The "Required reviewers per path" policy ensures the security team is added on every change under `/src/Auth/**`.

**Database migrations** — every PR that adds an EF Core migration is auto-tagged for DBA review. Reviewer checks for: index implications, lock-free schema changes, rollback path, multi-tenant impact, and that the migration is idempotent on retry. See [data-access/migrations.md](../data-access/migrations.md).

**Infrastructure-as-code PRs** — Bicep/Terraform changes go through `terraform plan` or `bicep what-if` in CI, and the diff is posted as a PR comment. Reviewer compares the IaC change against the plan output before approving.

**Security-sensitive code paths** — auth, crypto, PII handling, and secrets-loading code paths have `@security-team` as a required reviewer via `CODEOWNERS`. CodeQL results post inline as PR annotations.

**Hotfix during incident** — process allows merge-then-review for genuine P1 incidents, with the explicit requirement that a follow-up PR documents what was done, what was learned, and what to refactor. The original hotfix gets reviewed post-merge before the next deploy.

**External contractor PRs** — every PR from outside the org requires 2 approvals (one architectural, one functional) and runs in a sandboxed CI environment with no secrets. Squash merge with strict commit-message format keeps history clean.

**Multi-tenant SaaS** — reviewer's tenancy checklist: does every new query filter by `TenantId`? does every new endpoint check authorization? does the migration handle existing tenant data correctly? These are reviewed by the platform team via `CODEOWNERS`.

## Code Example — Before and After

A real example of how a payment-related PR's review converted from noise into signal.

**Before — typical low-signal review:**

```text
PR #2104 — "Add Stripe refund endpoint" — 320 lines changed

Reviewer 1:
- Line 23: "Use var here please."
- Line 45: "Add a blank line after this method."
- Line 78: "Maybe rename to ProcessRefund?"
- Line 112: "I'd prefer foreach over Where().ToList()."
- "LGTM after nits."

Reviewer 2:
- "Approved." (no comments)

Merged. Two weeks later: production incident — refund endpoint allowed 
arbitrary refund amounts because `request.Amount` was never validated against 
the original charge.
```

**After — same PR, useful review:**

```text
PR #2104 — "Add Stripe refund endpoint" — 320 lines changed

Reviewer 1 (security focus):
- Line 67 — issue (blocking):
  The refund amount is taken from the request body without validating it
  against `original.AmountCaptured`. A malicious caller can request a 
  refund larger than the original charge. Suggest:
  
  ```suggestion
  if (request.Amount <= 0 || request.Amount > charge.AmountCaptured)
      return BadRequest(new ProblemDetails { Title = "Invalid refund amount" });
  ```

- Line 89 — issue (blocking):
  Webhook signature comparison uses `==`. This is vulnerable to a timing 
  attack. Use `CryptographicOperations.FixedTimeEquals`:
  
  ```suggestion
  if (!CryptographicOperations.FixedTimeEquals(
          Encoding.UTF8.GetBytes(computedSignature),
          Encoding.UTF8.GetBytes(request.Headers["Stripe-Signature"])))
      return Unauthorized();
  ```

- Line 145 — question:
  What happens if Stripe returns success but the database `SaveChangesAsync`
  throws? It looks like we'd refund the customer but not mark it in our 
  system, leading to duplicate refunds on retry. Should we wrap this in 
  an outbox pattern or at least an idempotency key?

- Line 200 — suggestion (non-blocking):
  Consider extracting the audit-log call into a method. There are now 
  three places we do `_audit.Log("refund", ...)` with slightly different 
  payloads.

- praise:
  Nice use of `record` for the response type and including the original
  charge ID for traceability — that's going to help reconciliation.

Reviewer 2 (architecture / ops):
- Line 23 — thought (non-blocking):
  We've talked about pushing payment-provider abstractions behind 
  `IPaymentGateway` consistently. Refunds via Stripe direct here means 
  one more place to update when we add Adyen next quarter. Not blocking 
  this PR, but let's track it in #2087.

- Line 156 — issue (blocking):
  No structured logging on the refund failure path. If this fails in 
  production at 2 AM, on-call will have nothing to work with. Add 
  `_logger.LogError(ex, "Refund failed for charge {ChargeId} amount {Amount}", 
   charge.Id, request.Amount)`.

- General — chore:
  Migration `20260604_AddRefundsTable.cs` is missing a rollback `Down()` 
  method. Required by our migration policy.

Author then pushed 4 commits addressing each blocking issue, requested
re-review, both reviewers approved. Merged. No incident.
```

The second review took ~30 minutes per reviewer. It cost ~1 person-hour. It prevented at least one production incident, one security vulnerability, and one ops blind spot — easily worth a person-week of cost saved.

## Interview Questions and Answers

### 1. What do you look for first when you start reviewing a pull request?

**Why this matters:** The order reveals priorities. A reviewer who starts with style is misallocating attention.

**Answer:** I start with the description and the linked issue — does this change actually solve the problem it claims to? Then the test changes, because the tests show what behavior the author considers important. Then I scan the diff at a high level looking for architectural surprise (new dependencies, new layers, new abstractions). Only then do I read line-by-line, in priority order: correctness, security, error handling, observability, performance, readability, style. Style I usually defer to analyzers and tooling.

**Trade-off:** This is slower than "read top to bottom and comment as I go," but it catches the high-impact issues first. If I run out of time, I've at least covered the things that matter.

**Real project:** Reviewing a payment-refund PR using this order, I caught a refund-amount validation bug in the first 10 minutes — before I'd noticed the unused `using` directives. The author and I both agreed afterward that ordering attention this way was worth more than catching every nit.

### 2. How do you keep PR size manageable?

**Why this matters:** Large PRs review badly. The interviewer wants concrete habits, not just "split them up."

**Answer:** Plan the PR before writing it. If the change naturally splits into refactor-then-feature or schema-then-code, those are separate PRs in order. I use draft PRs early to get directional feedback before investing in polish. If a PR grows past ~400 lines mid-work, I stop and ask whether part of it can ship now — for example, the new repository class, with the feature that uses it landing later. Feature flags help: I can merge code that's not yet wired up, behind a flag set to off in production. When a PR genuinely can't be split (a single coherent algorithm change), I write a longer description that walks the reviewer through it.

**Trade-off:** Splitting into many small PRs creates merge ordering and rebase work for the author. The win is faster review and less risk per PR; the cost is more PR overhead.

**Real project:** A 1,200-line PR for a new tenant-isolation feature became four PRs over two days: (1) extract `ITenantContext`, (2) add tenant-aware EF Core query filters, (3) add the new admin endpoint behind a feature flag, (4) flip the flag in a fifth tiny PR after canary deployment. Each was reviewed in under an hour.

### 3. How do you handle a reviewer who blocks on personal preference?

**Why this matters:** This is a real interpersonal situation. The answer reveals professional maturity.

**Answer:** I respond on the PR with a short, factual explanation: "I'm using LINQ here because [reason]; the team standard in `CONTRIBUTING.md` doesn't have a preference and analyzers don't flag it." If the reviewer pushes back, I take it offline — a quick chat usually resolves it. If we genuinely disagree, I'll often defer for the PR's sake (consistency matters and one method shape isn't worth a fight), but I'll raise the pattern in a team discussion so we decide as a group rather than relitigating every PR. The key is to distinguish review feedback from imposing personal style, and to keep the PR from rotting while the discussion happens.

**Trade-off:** Deferring too often makes the codebase shaped by whoever pushes hardest. Standing ground too often slows reviews. The sustainable middle is "decide team-wide once, then point at the standard in PR comments."

**Real project:** Repeated `var` vs explicit type disagreements on PRs led the team to add `csharp_style_var_for_built_in_types = true:suggestion` to `.editorconfig`. After that, the debates disappeared — the tooling enforced the team decision.

### 4. How do you review a database migration PR?

**Why this matters:** Migrations are usually the riskiest part of any change. The answer reveals whether you understand operational impact.

**Answer:** I check several specific things: Is the migration backwards-compatible during a rolling deploy (the old code must still work for a few minutes while new code rolls out)? Does it take locks on large tables (an `ALTER TABLE ADD COLUMN NOT NULL DEFAULT` on a million-row table can lock the table for minutes)? Is there an index implication — does this query plan still scale? Does the `Down()` method actually reverse the change cleanly? Is the migration idempotent on retry, in case it fails midway? For multi-tenant systems, does it handle existing tenant data correctly? I also verify the migration was tested against a copy of production data, not just an empty dev database.

**Trade-off:** Thorough migration review is slow — sometimes I'll book a 15-minute call with the author and a DBA rather than ping-pong on a PR thread. The cost is worth it: a bad migration can take a service down for hours.

**Real project:** Caught a migration in review that would have rebuilt an index on a 50M-row `Invoices` table during deploy — taking the API down for ~20 minutes. We rewrote it as a `CREATE INDEX CONCURRENTLY` (PostgreSQL) split across two deploys, deployed safely with zero downtime.

### 5. A junior engineer's PR has good intent but several real issues. How do you give feedback?

**Why this matters:** Tests teaching ability — a reviewer who demoralizes juniors becomes a hiring liability.

**Answer:** I lead with what's good. Specifically, not just "nice work" — call out the specific decision that was right. Then I separate the issues: the truly blocking ones (correctness, security) versus the suggestions. I phrase blocking issues factually ("this allocates per request because…" not "this is wrong"). For suggestions I link to docs or examples so the junior can learn the pattern, not just memorize my comment. I avoid lecturing on principles in PR comments; if the feedback is bigger than a paragraph, I move it to a 15-minute call or a doc, then summarize the decision on the PR. I always close with what's needed to merge.

**Trade-off:** This takes longer than "here's everything wrong, fix it." Done well, the junior is more capable next PR and asks fewer of the same questions over time.

**Real project:** A new hire's first PR had a real concurrency bug, a missing test, and several style nits. I left exactly two comments: the concurrency bug with a code suggestion, and "let's add a test that demonstrates this race condition," with a link to a similar test from another file. The style nits I let `dotnet format` handle. The PR merged the next day, and the hire told me later that getting one focused, fixable review was much more useful than a wall of feedback.

### 6. How do you avoid reviews becoming a bottleneck?

**Why this matters:** Slow reviews kill velocity. The answer reveals process awareness.

**Answer:** Several mechanisms together. Team SLA: first review within 4 working hours; second-round within 2. PRs that exceed SLA get pinged in the team chat — not as blame, as a signal to redistribute load. Smaller PRs (under 400 lines) review faster, so the team incentivizes them. Reviewers in different timezones partner on overnight reviews. Automated checks (analyzers, tests, format) catch as much as possible so humans review intent, not style. For complex PRs, pair-program or screen-share for 15 minutes instead of multiple async rounds. And when load is consistently high, hire more reviewers or invest in better tooling (CodeQL, custom analyzers) to take human work off the queue.

**Trade-off:** Pressuring fast reviews can make reviewers superficial. The fix is to combine speed targets with quality signals (rework rate, post-merge defect rate).

**Real project:** A team had average time-to-first-review of 2 days. We introduced SLAs + a "reviewer of the day" rotation that ensured someone was always responsible for the queue. Time-to-first-review dropped to 3 hours within two sprints, and rework rate didn't change.

### 7. How does code review fit into compliance frameworks like SOC 2 or PCI-DSS?

**Why this matters:** Most senior .NET roles touch regulated systems. Knowing what auditors expect is a real differentiator.

**Answer:** Compliance frameworks require evidence of "change control" — typically a documented review by a person other than the author before production deployment. The PR with at least one non-author approval, the linked work item or ticket, the CI run log, and the merge commit together form that evidence. To make audits easy: enforce two-person approval via branch protection (no overrides), require linked issues, require status checks to pass, retain PR history (don't delete branches), and forbid direct pushes to `main` and release branches. For PCI specifically, changes touching card data paths need additional reviewers from the security team — `CODEOWNERS` rules enforce this. The auditor's job becomes "show me 10 random PRs that merged to main last quarter" and the evidence is right there.

**Trade-off:** Strict enforcement (no overrides, no force-push) occasionally hurts during outages. The pragmatic answer is a documented emergency-change procedure that logs the override and triggers a post-incident review.

**Real project:** During a SOC 2 audit, the auditor sampled 30 PRs from the previous year. All had: linked work items, at least one non-author approver, green CI, and a clean merge commit. The audit took less than a day for code-change control because the evidence was the normal workflow output — no separate paperwork required.

## Summary Checklist

- [ ] I keep PRs small (under ~400 lines of diff) and focused on one concern.
- [ ] I self-review before requesting human review.
- [ ] I write a useful PR description: what, why, what was tested, how to verify.
- [ ] I review in priority order: correctness → security → tests → ops → performance → readability → style.
- [ ] I use Conventional Comments so blocking vs informational feedback is unambiguous.
- [ ] I let `.editorconfig` and analyzers handle style; reviewers focus on substance.
- [ ] I use suggestion blocks for one-click acceptance.
- [ ] I configure `CODEOWNERS` / required reviewers and branch protection rules.
- [ ] I require passing CI (build, tests, analyzers, security scans) before merge is allowed.
- [ ] I respond constructively, acknowledge good work, and keep reviews from becoming personal.
