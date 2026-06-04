# Code Review Discussion Questions

## What It Is

**Code review discussion** is the verbal skill of talking through a pull request in English — explaining what changed, why it changed, what risks it carries, and how you would respond to reviewer feedback. In interviews, it is rarely a silent diff exercise. The interviewer shares a PR (or describes one), and listens to how you reason, how you phrase critique, and how you defend your own code under pressure.

It covers four conversation modes:

1. **Author mode** — walking a reviewer through your own PR.
2. **Reviewer mode** — leaving comments on someone else's PR.
3. **Defender mode** — responding when a reviewer pushes back on your design.
4. **Mentor mode** — explaining to a junior why a pattern is risky without sounding dismissive.

Before — a harsh, low-information comment:

> "This is wrong. Fix it."

After — a constructive, evidence-based comment:

> "If `ChargeAsync` throws after the row is inserted but before the response is returned, the client will retry and we will charge twice. Could we move the insert inside the same transaction as the Stripe call, or add an idempotency key on `OrderId`?"

Both comments take five seconds to write. Only one moves the PR forward.

## Why It Exists

Code review as a practice exists because **bugs found in review cost roughly 10–100x less than bugs found in production** (industry rule of thumb, repeated in NIST and IBM studies). But the *discussion* skill exists because reviews routinely fail for non-technical reasons:

- Authors get defensive and reject valid feedback.
- Reviewers nitpick formatting and miss a SQL injection.
- Comments are so vague (`"refactor this"`) that the author cannot act.
- Senior engineers crush juniors with tone, and juniors stop opening PRs.
- Reviews stall for days because nobody asks the explicit question: "is this blocking or a suggestion?"

Interviewers ask code review questions because they cannot watch you write production code for six months. But they *can* watch you discuss a 40-line diff for ten minutes — and that conversation reveals your judgment, your humility, your prioritization, and your English fluency in one shot.

## When To Use It

**Use structured code review discussion for:**

- Walking interviewers through a PR you brought as a portfolio sample.
- Live "review this snippet" exercises in technical screens.
- Behavioral questions like *"tell me about a time you disagreed with a reviewer"*.
- Explaining why you blocked, approved, or requested changes on a real PR.
- Mentoring junior engineers in 1:1s and design syncs.
- Defending an architecture decision against a skeptical senior.

**Do not use this framing for:**

- Pair programming sessions — those are real-time, not asynchronous review.
- Architecture design reviews — those need an ADR (Architecture Decision Record), not a PR comment.
- Production incidents — use a blameless postmortem format, not PR vocabulary.
- Performance benchmarking discussions — bring numbers, not subjective opinions.

## Why It Is Important

In a .NET backend role, you will spend roughly 20–30% of your week in code review — reading other people's PRs, defending your own, replying to comments, resolving threads. If you cannot do this clearly in English, three things break:

1. **Velocity drops.** PRs sit open for days because comments are unclear and authors guess at intent.
2. **Quality drops.** Reviewers approve out of frustration ("LGTM") rather than fight through a long thread.
3. **Trust drops.** Teammates avoid reviewing your code because every conversation turns into a debate.

For cloud-native systems specifically, review discussion is the last human gate before code touches **production Azure resources** — a bad `Bicep` template can leak storage account keys, a missing `await` on a Service Bus message can silently drop orders, and a JWT validation typo can let unauthenticated traffic into a controller. Interviewers know this, which is why they probe your review instincts even when the role is mostly coding.

## How It's Used in C# / .NET

For .NET teams, code review discussion happens inside a specific toolchain. Knowing the vocabulary signals seniority.

### PR description template

Most .NET teams enforce a PR template in `.github/PULL_REQUEST_TEMPLATE.md` or Azure DevOps PR description. A strong template you can reference verbally:

```markdown
## What
Add idempotency key support to PaymentsController.Charge.

## Why
Closes #4821. Stripe retries on 5xx, and we were double-charging on transient
network failures (3 incidents in last 30 days, see App Insights query in #4821).

## How
- New IdempotencyKey middleware reads `Idempotency-Key` header.
- Stores key + response in Redis with 24h TTL.
- Returns cached response on duplicate key.

## Risk
- Redis dependency added to charge path. Falls back to processing if Redis is down
  (logs warning). Discussed with @sre-team in #channel.

## Test
- Unit tests in PaymentsControllerTests.IdempotencyTests
- Manual test against Stripe test mode with duplicate header.
```

When you walk through this in an interview, name each section: *"I always lead with **What**, then **Why** with a link to the incident, then **How**, then **Risk**, then **Test**."*

### PR UI vocabulary (GitHub and Azure DevOps)

| Term | What it means | When to use it in discussion |
|---|---|---|
| **Draft PR** | PR marked not-ready-for-merge; reviewers are notified but not requested. | *"I open a draft PR early to get directional feedback before I polish."* |
| **Required reviewer** | Branch protection forces approval from a specific person or team (e.g., `@security-team` for auth code). | *"Our `auth/` folder has CODEOWNERS routing to the security team."* |
| **Suggested change** | GitHub's diff-block suggestion the author can apply with one click. | *"I leave nit-level fixes as suggested changes so the author doesn't context-switch."* |
| **Resolve conversation** | Marks a comment thread as addressed. Author resolves after fixing; reviewer reopens if not satisfied. | *"I never resolve my own threads unless the reviewer agrees."* |
| **Re-request review** | Pings the reviewer after pushing new commits. | *"After pushing the fix, I re-requested review instead of pinging in chat."* |
| **Linked work item** | PR is linked to an Azure DevOps work item or GitHub issue via `Fixes #1234` or the DevOps Work Items tab. | *"Linking the PR auto-closes the bug and gives auditors the trail."* |
| **Branch protection** | Rules requiring CI green, N approvals, signed commits, linear history before merge. | *"`main` is protected — needs CI green and one CODEOWNERS approval."* |
| **Squash vs merge commit** | Merge strategy. Squash collapses PR commits into one; merge preserves history. | *"We squash feature PRs to keep `main` clean; release branches use merge."* |
| **Force push to PR branch** | Rewriting history on the PR branch after rebasing. | *"I avoid force-push after review starts — it invalidates reviewer comments."* |

### Common .NET review-comment patterns

```text
// On an EF Core query
"This will issue N+1 — `order.Items` is lazy-loaded inside the loop.
Could we use `.Include(o => o.Items)` or project to a DTO?"

// On an async method
"`Task.Result` blocks the thread pool. Suggest `await` instead — under
load this caused deadlocks in the legacy NotificationService."

// On a controller
"This action returns 200 with a JSON error envelope. Per our API guidelines,
domain validation errors should be 400 with ProblemDetails. See
[aspnet-core/error-handling-and-problem-details.md]."

// On a Bicep file
"Storage account is created without `allowBlobPublicAccess: false`.
Security policy requires it explicit — would fail an `azqr` scan."

// On a DbContext registration
"AddSingleton on DbContext will leak DbContext across requests.
Should be AddDbContext (scoped) — see [csharp/dependency-injection.md]."
```

### Conventional comment prefixes

Teams adopting [conventionalcomments.org](https://conventionalcomments.org/) prefix every comment with intent. Mentioning this in an interview is a strong signal:

| Prefix | Meaning |
|---|---|
| **nit:** | Minor, non-blocking |
| **suggestion:** | Recommendation, author decides |
| **question:** | Genuine clarification, not disguised criticism |
| **issue:** | Blocking — must fix before merge |
| **praise:** | Positive callout (rare but valuable) |
| **thought:** | Reviewer thinking out loud, not actionable |

> "I prefix `issue:` only for must-fix blockers. Everything else is `suggestion:` or `nit:` so the author knows what's blocking."

## Advantages

- **Catches defects cheaply** — bugs found in review cost a fraction of production fixes.
- **Spreads context** — every reviewer learns a corner of the system they didn't write.
- **Mentors juniors** — well-phrased comments teach patterns without a separate training session.
- **Documents decisions** — PR threads become searchable history (`"why did we choose Service Bus over Event Grid?"`).
- **Enforces standards** — CODEOWNERS routes security PRs to the security team automatically.
- **Builds trust** — a calm, evidence-based reviewer reputation makes your own PRs sail through faster.

## Disadvantages

- **Slow if abused** — bikeshedding on naming can stall a hotfix for days.
- **Asymmetric workload** — senior engineers become review bottlenecks if not rotated.
- **Tone risk** — written comments lack vocal cues, and harsh phrasing damages morale.
- **False confidence** — `LGTM` from a tired reviewer is worse than no review.
- **Local optimization** — reviewers see the diff, not the system; cross-cutting design issues slip through.
- **Process theatre** — checkbox-driven reviews ("did you update CHANGELOG?") miss real risks.

## Common Mistakes

### 1. Reviewing the diff, not the change

> ❌ "LGTM, formatting is consistent."

The diff compiles. But the reviewer never checked whether the new `ChargePaymentAsync` handles partial failures, or whether the new column needs a backfill migration.

✅ **Fix:** Always read the PR description and the linked work item first. Ask: *what behavior is this PR supposed to change, and does the code actually achieve that?* Then check the diff.

### 2. Nitpicking style when analyzers should do it

> ❌ "Please use `var` here instead of `string`."

Style debates waste human time. Configure `.editorconfig`, `dotnet format`, Roslyn analyzers, and StyleCop to auto-enforce. Reserve human review for behavior, security, and design.

✅ **Fix:** *"We enforce style with `.editorconfig` and `dotnet format` in CI. I never leave style comments — if it merged, it's compliant."*

### 3. Vague comments with no actionable suggestion

> ❌ "This feels off."

The author has no idea what to change. They either guess (and probably guess wrong) or ignore it.

✅ **Fix:** State the **observation**, the **risk**, and a **proposed action**. *"This `await Task.WhenAll` swallows individual exceptions — if one Stripe call fails, we won't know which. Could we wrap each in try/catch and log per-order failures?"*

### 4. Approving without running the code

Especially common with EF Core migrations and Bicep templates. The reviewer reads the change, says it looks fine, never runs `dotnet ef migrations script` or `bicep build`, and a broken migration ships.

✅ **Fix:** For migrations, infra-as-code, and SQL changes, *pull the branch locally and run it*. Mention this in interviews — it shows operational maturity.

### 5. Mixing must-fix and nice-to-have in one wall of comments

When ten comments arrive with no prioritization, the author either rewrites everything (slow) or fixes the easy ones and misses the critical one.

✅ **Fix:** Use conventional comment prefixes (`issue:`, `suggestion:`, `nit:`) or explicitly say *"only the first two are blocking; the rest are follow-up suggestions."*

### 6. Defending instead of investigating when challenged

> ❌ Reviewer: "Won't this deadlock under load?"
> Author: "No, it's fine, I've done this before."

The author shut down the conversation without evidence. Even if they're right, the reviewer leaves uncertain — and the next PR review starts with friction.

✅ **Fix:** *"Good question — let me check. I assumed the `SemaphoreSlim` would serialize the writes, but you're right that the outer `Parallel.ForEachAsync` could starve it. I'll write a load test and post results."* Then actually do it.

### 7. Reviewing without understanding the system

Junior reviewers comment on a payment controller without knowing the team uses Stripe's idempotency keys, or flag a SQL query without knowing the table is partitioned. Their comments waste author time.

✅ **Fix:** If you don't know the context, ask first: *"How does this interact with the existing retry policy?"* Comments framed as questions cost less than wrong assertions.

### 8. Letting a PR sit because "I'll get to it tomorrow"

PRs decay. By day three the branch has conflicts, the author has context-switched, and reviewing takes twice as long.

✅ **Fix:** Personal rule — review within 24 business hours, or explicitly say *"I won't get to this until Thursday — please find another reviewer if it's urgent."*

## Best Practices

- **Lead with the why.** Every comment should answer "what's the risk?" — not just "what's wrong?"
- **Use evidence.** Link to docs, prior incidents, App Insights queries, or the spec. Opinions without evidence are noise.
- **Separate blocking from suggesting.** Tag with `issue:`, `suggestion:`, `nit:`, or say it explicitly.
- **Ask, don't accuse.** *"Could this race with the cache invalidation?"* beats *"This races with the cache invalidation."*
- **Praise when warranted.** A `praise:` comment costs nothing and builds trust.
- **Review your own PR first.** Open the diff, scroll through it, leave self-comments explaining tricky parts before any reviewer arrives.
- **Keep PRs small.** Under 400 lines changed gets a real review; 2000 lines gets `LGTM`.
- **Use draft PRs for direction.** *"I'm going down this path, does it look right before I add tests?"*
- **Resolve only after agreement.** Don't mark your own threads resolved unilaterally.
- **Be explicit about merge gates.** *"This is approved pending green CI and one more sign-off from @data-team."*
- **Rotate reviewers.** CODEOWNERS routing prevents one senior from becoming the bottleneck.
- **Default to kindness.** Behind every PR is a human who is probably tired and trying their best.

## Related Concepts

- [communication/behavioral-questions-for-dotnet-developers.md](behavioral-questions-for-dotnet-developers.md) — STAR-format answers about review conflicts.
- [communication/explaining-technical-decisions-in-english.md](explaining-technical-decisions-in-english.md) — Framing for defending design choices.
- [communication/explaining-trade-offs.md](explaining-trade-offs.md) — How to articulate "we chose X because Y" in review threads.
- [testing-quality/code-review-practices.md](../testing-quality/code-review-practices.md) — The mechanical checklist side of code review.
- [testing-quality/clean-code.md](../testing-quality/clean-code.md) — Standards you reference in review comments.
- [devops/ci-cd-pipelines.md](../devops/ci-cd-pipelines.md) — Branch protection and required checks.

## Real-World Usage

### Reviewing a payment controller PR (ASP.NET Core)

A teammate adds `PaymentsController.RefundAsync`. As reviewer you check: does it require an `[Authorize(Policy="RefundApprover")]`? Does it write to the audit log before calling Stripe? What happens if Stripe returns `202 Accepted` with a pending refund — does the code handle that state? Are there integration tests against Stripe's test mode? You leave four comments: one `issue:` on the missing auth policy, two `suggestion:` on the audit log ordering, one `praise:` for the clean separation of `IRefundService`.

### Reviewing an EF Core N+1 PR

A new endpoint returns orders with line items. You see `foreach (var order in orders) { var items = await db.Items.Where(...).ToListAsync(); }`. You comment: *"issue: this fires one query per order — for the analytics dashboard pulling 5k orders, that's 5k roundtrips to Azure SQL. Suggest `db.Orders.Include(o => o.Items)` or project to a DTO with a single join. Want me to share a Snapshot Debugger profile from staging that showed a similar issue last month?"* You attach the SQL captured by `ToQueryString()`.

### Reviewing a Bicep deployment PR

The PR adds a new storage account for export files. You check: `allowBlobPublicAccess: false`, `minimumTlsVersion: 'TLS1_2'`, `supportsHttpsTrafficOnly: true`, network rules denying public access, private endpoint configured. You also check the `azqr` scan result attached in the PR description. You comment: *"issue: `allowSharedKeyAccess` defaults to true. Our security baseline requires `false` with Managed Identity instead. See [azure/configuration-and-secrets-management.md]."*

### Reviewing a JWT implementation PR

A junior adds custom JWT validation. Red flag — almost nobody should hand-roll JWT. You comment kindly: *"suggestion: rather than parsing the JWT manually, can we use `AddJwtBearer` with `TokenValidationParameters` configured for our Entra ID tenant? The built-in handler validates signature, audience, issuer, and expiry — easy to get one of those wrong by hand. Happy to pair on it if useful."* You link to [aspnet-core/jwt-authentication.md](../aspnet-core/jwt-authentication.md).

### Defending your own PR under push-back

A senior reviewer says your Service Bus consumer should use `PeekLock` instead of `ReceiveAndDelete`. You actually chose `ReceiveAndDelete` intentionally for the telemetry pipeline because lost messages are acceptable and throughput matters more than durability. You reply: *"thought: I picked ReceiveAndDelete because this consumer handles non-critical telemetry — at 50k msg/min the PeekLock checkpoint overhead doubled p99 latency in our load test (results in #4901). For the payments queue I agree PeekLock is required. Want me to add a comment in code explaining the choice?"*

### CI/CD: Required reviewers and branch protection

`main` is protected — needs two CODEOWNERS approvals, green CI (build, unit tests, integration tests against a Docker SQL container, `dotnet format --verify-no-changes`, Bicep validation, `azqr` scan), and a signed commit. `auth/` and `payments/` paths route to `@security-team`. Hotfix PRs to `release/*` branches need a release manager approval. You reference this setup in interviews to show you understand merge gating beyond `git push`.

## Example Dialogues — Before and After

### Scenario: Reviewing an EF Core query that has an N+1

**Before — harsh, vague, ego-driven:**

> **Reviewer:** This code is bad. You don't know EF Core. Rewrite the whole method.
>
> **Author:** It works on my machine, what's the problem?
>
> **Reviewer:** It's obvious. Look at the query.
>
> **Author:** Fine, I'll change it. *(merges resentment, doesn't actually learn the pattern, makes the same mistake on the next PR)*

What's wrong:
- No explanation of the risk.
- Personal attack ("you don't know EF Core").
- "Obvious" weaponizes the reviewer's experience.
- Author doesn't understand *why*, so the lesson doesn't stick.

**After — constructive, evidence-based, mentoring:**

> **Reviewer:** issue: I think this will issue an N+1 — the `foreach (var order in orders)` triggers a lazy load on `order.Items` per iteration. For the 5k-order analytics export, that's 5k roundtrips to Azure SQL. We hit this exact pattern in PR #4203 last quarter and it took p99 from 200ms to 12s.
>
> Could we use `.Include(o => o.Items)` for a single query, or project into a `OrderExportDto` with `.Select(...)` if we only need a few fields? I can pair on it if useful.
>
> **Author:** Good catch — I didn't realize lazy loading was on for this DbContext. Let me try the projection approach since we only need three fields per item. I'll push the change and re-request review with the new `ToQueryString()` output in the description.
>
> **Reviewer:** Perfect. Once you push, would you also add a quick comment in the code (`// projected to DTO to avoid N+1 — see PR #4521`) so the next person doesn't re-introduce it?
>
> **Author:** Done — pushed and re-requested. Numbers in the PR description show one query, 180ms.
>
> **Reviewer:** Approved. praise: the `ToQueryString()` snippet in the description is great, let's make that a team norm.

What's right:
- Concrete risk with a number (200ms → 12s).
- Two solutions offered, not a demand.
- Offer to pair softens the criticism.
- Author engages instead of capitulating.
- Reviewer follows up with a code comment request — locks in the lesson.
- Closing `praise:` builds trust for next time.

### Scenario: Author defending a design choice

**Before — defensive, no evidence:**

> **Reviewer:** Why are you using `MediatR` here? It's overkill.
>
> **Author:** Because I like it. It's clean.
>
> **Reviewer:** That's not a reason. Remove it.
>
> **Author:** Fine. *(rips out MediatR, loses the decoupling, ships worse code)*

**After — calm, evidence-based defense:**

> **Reviewer:** Why are you using `MediatR` here? It's overkill for a single command handler.
>
> **Author:** Fair question. Three reasons: (1) the team standardized on MediatR for command handlers in the `Orders` module — see [architecture/cqrs.md] — so this is consistent. (2) The pipeline behaviors give us `ValidationBehavior` and `LoggingBehavior` for free, which we'd otherwise re-implement. (3) When we add the `OrderCreated` integration event next sprint (linked work item #4733), MediatR's notification publishing is already wired. If we drop MediatR for this one handler, we'll add it back in two weeks. Want me to add a comment in the PR description explaining the choice?
>
> **Reviewer:** That makes sense — I missed the link to #4733. Approved. suggestion: add a one-liner in the controller comment so the next reviewer doesn't ask the same question.
>
> **Author:** Good idea, pushing now.

What's right:
- Three concrete reasons, not "I like it."
- Reference to existing architecture docs.
- Reference to upcoming work that validates the choice.
- Offer to document the decision permanently.

## Interview Q&A

### Q1: Walk me through how you review a pull request.

**Why this matters:** Tests whether you have a repeatable process or just skim diffs.

**Answer:** I start with the PR description and linked work item to understand *what behavior should change*. Then I scan the file tree — is the change scoped to one module, or sprawling? Then I read the tests before the implementation; tests tell me what the author thinks the contract is. Then the implementation, focusing on correctness, security, error paths, and observability. I leave comments using conventional prefixes (`issue:`, `suggestion:`, `nit:`, `question:`) and explicitly call out which are blocking. For infra changes or migrations, I pull the branch locally and run it.

**Trade-off:** This takes 15–30 minutes per PR. For tiny one-line fixes, I shortcut it. For PRs over 400 lines I usually ask the author to split.

**Real project:** Reviewing a `PaymentsController.RefundAsync` PR — found a missing `[Authorize]` policy and an ordering bug where audit log was written *after* the Stripe call, meaning failed refunds had no audit trail. Both caught because I read the tests first and noticed neither scenario was covered.

### Q2: A senior engineer pushes back hard on your design in a PR comment. How do you respond?

**Why this matters:** Tests emotional regulation and engineering humility.

**Answer:** First, I assume they have context I don't. I re-read their comment, then re-read my code, then ask a clarifying question — *"to make sure I understand, are you concerned about X or Y?"*. If they're right, I thank them, update the code, and resolve. If I disagree, I respond with evidence — benchmark numbers, links to docs, prior incidents — never *"because I said so."* I never resolve a thread unilaterally if the reviewer is still uncertain.

**Trade-off:** Sometimes you genuinely disagree and have to escalate. I'd rather take it to a 15-minute call than fight in PR comments where tone gets lost.

**Real project:** A senior insisted I switch from `ReceiveAndDelete` to `PeekLock` on a Service Bus consumer. I had picked it for throughput on a non-critical telemetry pipeline. I shared the load test showing 2x latency under PeekLock and explained the message-loss tolerance. He agreed, and we added a comment in the code documenting the choice.

### Q3: How do you give feedback to a junior engineer without crushing their confidence?

**Why this matters:** Tests mentoring instinct, which is a leveling signal.

**Answer:** Three rules. First, lead with a question, not an assertion — *"have you considered..."* invites thinking; *"this is wrong"* shuts it down. Second, explain the *why* behind every comment, not just the *what* — if I just say "use `Include`", they don't learn the N+1 pattern. Third, offer to pair if the gap is large. And I always include a `praise:` comment when something is well-done — junior engineers especially need positive reinforcement that survives the negative comments.

**Trade-off:** Detailed mentoring comments take longer. I budget for it because it pays back tenfold in the junior's next PR.

**Real project:** A junior hand-rolled JWT validation in a controller. Instead of rejecting it, I left a comment explaining why the built-in `AddJwtBearer` is safer (signature, audience, issuer, expiry all validated), linked the docs, and offered to pair on rewriting it. Two sprints later they were the one teaching another junior the same pattern.

### Q4: What review comments should *not* exist on a healthy team?

**Why this matters:** Tests whether you understand automation as the foundation of fast review.

**Answer:** Style comments — `.editorconfig`, `dotnet format`, Roslyn analyzers, and StyleCop should enforce style in CI. Naming preferences without behavior impact. Import ordering. Trailing whitespace. Any comment a machine could leave. Human review time is too expensive for what a linter handles. I also push back on "could you also..." scope creep — separate PRs for separate concerns.

**Trade-off:** Setting up analyzers takes upfront work and some debate ("tabs vs spaces"). Once configured, it pays back forever.

**Real project:** Joined a team with 30% of review comments being style debates. We adopted `.editorconfig` + `dotnet format --verify-no-changes` in CI. Review comment count dropped by a third, and average PR merge time went from 2.3 days to 1.1 days within a month.

### Q5: How do you handle a PR that has been sitting for a week?

**Why this matters:** Tests ownership and team awareness.

**Answer:** If it's mine, I check why — is it stuck on a reviewer, on CI flakiness, on a blocker I missed? I rebase to clear conflicts, leave a polite ping with a specific ask (*"@reviewer — any blocking concerns, or OK to merge?"*), and if still stalled, I bring it up in standup. If it's someone else's PR I was asked to review, I either review it today or explicitly hand it off — saying *"I can't get to this until Thursday, please find another reviewer if urgent"* is better than silence.

**Trade-off:** Pinging too aggressively damages relationships. I usually wait 48 business hours before the first ping.

**Real project:** A PR for a hot bug fix sat for three days because two reviewers each assumed the other would handle it. I added an explicit `@reviewer1 @reviewer2 — one of you, please?` comment, paired briefly to walk one of them through it, and merged the same day. We changed our process to require one *named* primary reviewer, not just a team mention.

### Q6: A reviewer leaves "LGTM" on your security-sensitive PR after 90 seconds. What do you do?

**Why this matters:** Tests whether you optimize for ceremony or for real quality.

**Answer:** Politely ask for a deeper look. *"Thanks — given this touches the JWT validation path, would you mind a slower pass on the token handling and the test coverage? I'd rather catch a hole now than ship one."* If I have access controls via CODEOWNERS, security-sensitive paths should require a `@security-team` reviewer automatically — relying on individual diligence is fragile.

**Trade-off:** It can feel awkward to question someone's approval. I frame it as "I want to be careful here" rather than "you didn't review properly."

**Real project:** A teammate `LGTM`'d a 600-line auth refactor in two minutes. I asked for a re-review and they found a case where `ValidIssuer` wasn't set, meaning any valid JWT from any tenant would have passed. We then added `auth/` to CODEOWNERS pointing at the security team so it could never happen again.

### Q7: When is it appropriate to merge your own PR without an approval?

**Why this matters:** Tests judgment about emergencies vs. discipline.

**Answer:** Almost never. The two cases I've done it: (1) a production incident where waiting for a reviewer would extend downtime, and even then I pinged someone for an immediate async review and merged with `[hotfix]` tag, then opened a follow-up PR with the post-fact review and lessons; (2) trivial doc-only changes on a personal sandbox repo. In a team repo with branch protection, I'd never bypass — the protection exists for a reason and `--no-verify` or admin override is a smell.

**Trade-off:** During incidents, speed matters. But "we'll review it later" usually means never, so I make the follow-up PR mandatory.

**Real project:** A 2am incident where a config typo locked out our App Service. I pushed the one-character fix, pinged the on-call SRE for async eyes, merged with `[hotfix]`, then next morning opened a postmortem PR adding the missing config-validation step to CI so the same typo couldn't happen again.

### Q8: How do you discuss code review in your team's process — async only, or with meetings?

**Why this matters:** Tests understanding of remote-first vs. in-person dynamics.

**Answer:** Default to async in PR comments — it's searchable, slow, and gives the author thinking time. Move to a call when the thread is over five back-and-forths, or when tone is escalating, or when the design needs a whiteboard. After the call, I always summarize the decision back into the PR thread so future readers see the resolution. For especially large changes, I run a 30-minute "PR walkthrough" with the team before opening the PR — it gets feedback earlier when changes are cheap.

**Trade-off:** Calls cost more time but resolve faster. Async is cheaper but can stall.

**Real project:** A migration from Newtonsoft.Json to System.Text.Json affected 40 files. Instead of a giant PR, I ran a 30-min walkthrough with the team to agree on the converter strategy, then opened five small PRs each with the strategy already approved. All five merged within two days with minimal comments.

## Summary Checklist

- [ ] I can walk through a PR description structure (What / Why / How / Risk / Test) verbally.
- [ ] I know the vocabulary for GitHub and Azure DevOps PR UI (draft, required reviewer, suggested change, resolve, CODEOWNERS, branch protection).
- [ ] I use conventional comment prefixes (`issue:`, `suggestion:`, `nit:`, `question:`, `praise:`) and can explain why.
- [ ] I lead every comment with the risk, not just the observation.
- [ ] I separate must-fix from nice-to-have explicitly in every review.
- [ ] I defend my own designs with evidence — benchmarks, links, prior incidents — not opinion.
- [ ] I respond to push-back by investigating first, defending second.
- [ ] I mentor juniors with questions rather than assertions, and offer to pair on large gaps.
- [ ] I never leave style comments — analyzers and `dotnet format` handle those.
- [ ] I pull infra-as-code, migrations, and SQL changes locally before approving.
- [ ] I review my own PR before requesting review, and leave self-comments on tricky parts.
