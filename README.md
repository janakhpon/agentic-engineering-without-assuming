# Agentic Engineering Without Assuming

_I use coding agents to write most of my code these days, and I have ended up on the other side of the work: verifying and reviewing. What I still write by hand is the checks, the evals, the instructions and the config. This is what holds up across everything I maintain, told through one system's detail — where it earned its keep, and where it quietly told me nothing was wrong._

![Article cover - Agentic engineering without assuming](./assets/agentic-engineering-without-assuming.avif)

Let us start with a simple scenario.

A guest's arrival time was drifting. Every time an operator opened the booking panel and saved it without changing anything, the arrival moved. From New York, a no-op save shifted it eleven hours, onto the previous day.

The fix was clean. All the date handling into one module, a lint rule banning the calls that silently use the viewer's timezone, tests written, everything green.

That's the point where I'd normally stop. Instead I went to staging and broke the code on purpose, to see whether the tests would notice.

I started with the exact regression they existed for and deleted the timezone offset. **All 46 passed.** I tried four more sabotages. Forty-six green, every time.

The tests were well written. They had no route to the code that could break.

**That's the whole problem in miniature.** When an agent writes most of the code, you stop reading all of it and start trusting signals instead. Which makes *did it pass?* the wrong question. The one that matters is *was anything actually being watched?*

To be specific about what that does and doesn't mean: I still read the important changes myself, the ones that touch a main file or a real decision. What I don't get to is the volume underneath — not because any one of them is too big to read, but because there are more small, individually readable ones in a day than I have hours for: the small refactors, the second and third pass at a fix that already passed once, the stuff that's individually minor and is most of what an agent actually produces in a day. That's the gap the rest of this piece is about closing.

## What agentic engineering actually is

Strip the marketing off and there are three different things wearing one name.

There is autocomplete, which suggests the rest of the line you are typing. There is chat, where you paste a problem and get code back. And there's the third thing. You describe a goal, and a process goes off and reads files, edits several of them, runs your test command, reads the failure, and tries again.

That third one is what I mean by agentic engineering, and it's the interesting one, because of what changed. Autocomplete and chat both hand you a suggestion and stop. The loop doesn't stop. It keeps going until something tells it to stop, and what tells it to stop is our test suite, our linter, our type checker, or us.

That's the whole shift, and it's smaller than the discourse suggests. **The bottleneck moves from writing code to verifying it.** DORA, the industry's annual survey on software delivery, has a name for what that feels like, drawn from a large batch of open-ended engineer responses: the *verification tax*. The time you save generating gets spent auditing, and auditing is the harder cognitive job.

The 2025 DORA report found the same thing from the other end, at a much larger scale. Higher AI adoption is associated with an increase in delivery throughput **and** an increase in delivery instability. Both. At once.

So the practical question isn't whether we use agents. **It's what I can know without reading, because I can no longer read all of it.**

That question has two answers, and the rest of this piece is those two answers. A check that fails tells me something is wrong. A written record tells me why it is the way it is. Neither one needs me to read the code. Everything else is detail.

## Where TDD fits, briefly

Test-driven development, in one line: a failing test first, the least code that makes it pass, then a cleanup with the test holding the line. Red, green, refactor.

Honestly, I was never a big fan. I write test suites throughout my workflow to keep things in check without having to assume. But I do it where it's warranted, not strictly. I'd rather say that than pretend otherwise.

It pays clearly in one place and I wouldn't give it up. **A bug fix always starts with a test that reproduces the bug.** Red first, then the fix. That test is the proof you found the real defect rather than a nearby one, and it stays in the suite as a permanent guard.

Beyond that, the evidence is weaker than its reputation. The famous 2008 study across four teams at Microsoft and IBM reports "40% to 90% fewer defects" — a percentage off a starting point the paper never actually shows you, so there's no way to tell whether it was already good or already terrible. And the "15–35% longer development time" everyone quotes is labelled, in its own table, as management estimates. Nobody tracked the time. Then there's the ordering itself, which is the part everyone argues about. A 2005 experiment found the opposite: quality went the other way, and the gap was small enough that it could just be noise. A 2016 replication, run so the analysts didn't know which group was which until the numbers were already in, found no difference at all on anything it measured. And a study that broke TDD into its separate pieces (how small the steps are, how consistent they are, and the test-first ordering itself) found that step size and consistency predicted the outcome. The ordering, the part the technique is named for, didn't move the needle either way.

Their conclusion, which I have not been able to argue my way out of:

> The claimed benefits of TDD may not be due to its distinctive test-first dynamic, but rather due to the fact that TDD-like processes encourage fine-grained, steady steps.

One caveat, because the paper is honest about it and I'd rather not quote it harder than it quotes itself. Step size and consistency together only account for about a tenth of why outcomes actually varied. Nine-tenths is still unexplained by any of it. It is not a strong result, only the strongest one there is, and it points away from the thing the name emphasises.

So **the measured lever is small, steady increments.** That is the thing I protect. A task goes to an agent scoped small enough that I can read the whole diff. If I can't read it, it was too big. That's a planning failure, not a review failure. A forty-minute test-first cycle is worse than a five-minute test-after one, and for a long time I had that backwards.

There's one place I hold strict test-first specifically *because* an agent is involved, and the research above doesn't cover it. **An agent will write a test that passes.** Told to make something green, models modify the test, overload the comparison operator, or special-case the exact input. That is measured, not folklore — on benchmarks deliberately built so the spec and the tests conflict, frontier models took the shortcut on **49% to 76%** of tasks. A test you watched fail for the right reason cannot have been written to pass. The red step is not ceremony there. It is the control.

Kent Beck ran into the same wall and put it better than I can, describing his own agent sessions: it "doesn't want to do TDD. It wants to write the code and then write tests that pass."

That's the whole of my position on TDD. It's a technique, it has one clear win and one agent-specific win, and it is not what this piece is about. What follows is.

## The setup, and why each part earns its place

Most of what follows is one system's detail: a private monorepo for an AI product, the booking panel from the opening included, and the one I can actually show you real numbers for. I can describe its shape but not link it. The patterns aren't specific to it — they're what's held up, and not held up, across the dozen-odd repos I keep.

**One monorepo, two language ecosystems, one tree.** The web app and the HTTP API are TypeScript. The long-running workers and the shared platform packages are Go. Both live in the same repository. pnpm workspaces and Turborepo over the TypeScript half, a Go workspace file over the Go half. One root `check` script runs the union.

The monorepo isn't a philosophy, and the alternative isn't a "single repo" — a monorepo *is* a single repo. The alternative is one repository per service, and that's what I'm arguing against. Keeping them together is what makes the boundary between the two halves reviewable. A change to a shared queue contract touches a Go worker and a TypeScript caller. One diff. I read it once, instead of reconstructing it across two pull requests a week apart. Split the repo and that review stops existing, quietly.

Under it: Postgres, reached through a typed query builder rather than raw SQL, with generated migrations checked in. A Redis-backed queue for the retry-and-park work. Managed hosting for the web surface, containers built and deployed for everything long-running, error tracking wired into both halves.

Swap any of it for the equivalent. Nothing in the argument changes.

What does matter is the check list, and this part I'd defend line by line.

Prettier for format. ESLint and `go vet` for lint. `tsc` for types. Vitest for the TypeScript tests, `go test -race` for the Go ones, Playwright for the handful of browser paths.

Then three scanners, each pointed at a different kind of rot. Gitleaks for committed secrets. The package manager's own audit for vulnerable dependencies. Knip for dead exports and files nothing imports.

Plus a few repo-specific validators. They assert things no general tool can know. That a taxonomy has no gaps. That a feature flag means the same thing in both services.

Nothing exotic. **The point isn't the tools, it's that every one of them exits non-zero.** A linter that warns is a linter that gets ignored. I stopped configuring them that way.

And one detail about *when* each runs, because the tidier version is the wrong one.

Formatting runs on commit. The full chain runs on push. CI runs a strict subset.

One of those choices is the one I'd defend hardest. The secret scan sits in CI rather than in the commit hook, and the comment says why. It catches a secret committed with `--no-verify`. The hook by definition cannot. The gate was placed where the bypass isn't.

The other two, the dependency audit and the dead-code scan, are local-only. I don't have a defence for that. It's a gap I've written down rather than closed.

Anything that only runs locally is one `--no-verify` away from not running at all.

**The value isn't any individual check.** It's that an agent finishing a task and an agent finishing a task *correctly* have different exit codes. I don't have to be the one who notices.

![Kakashi, unbothered, reading — Naruto](./assets/agentic-engineering-without-assuming-meme-1.jpg)

> It finished in ninety seconds. I'm on paragraph two of the diff.

## The test suite, by tier — and it isn't a pyramid

I counted the test files before writing this, because I assumed I knew the shape and I was wrong about the proportions.

A hundred and one Go test files against seventeen on the TypeScript side. A fifth of the Go ones are integration tests that stand up a real Postgres. Four browser tests. Zero load tests.

That is not a pyramid. It's a column standing on the deterministic half of the system, and once I saw the count I stopped apologising for it. **Most of an AI product is not the model.** It's the plumbing. The code that reads the message, picks a tool, runs it, writes to the database, retries a failed job, holds a draft for a human. Same input, same output, forever. The model is a small, loud component inside a large, quiet system. That system behaves like every other backend you have tested, and the tests go where the code is.

**Unit.** The floor, and where test-first is easiest to hold. Pure functions, parsers, a state machine's transitions, a backoff calculator. These are the ones I write before the code without thinking about it, because writing the assertion is faster than writing the sentence describing the behaviour.

**Integration.** Real database, real queue, no mock standing in for either. This tier exists because of the `NULL` bug in scenario 3 below, which was invisible one tier down. A truth-table simulation of a `WHERE` clause is a unit test of my beliefs about SQL. The integration test is a test of SQL.

**End-to-end.** Four of them, deliberately. They're slow, they're the flakiest thing I own, and each one costs attention every time it goes yellow. I keep the ones that cover a path a human would notice within minutes of it breaking. I do not try to cover behaviour here that a lower tier can cover.

**Regression.** Not a tier so much as a rule that feeds the others. Every bug fix starts with a failing test that reproduces it, and that test lands in whichever tier can actually see the bug. Half of them end up in integration, which is itself a finding — it means most of what reaches me isn't a logic error, it's a boundary error.

**Evals.** Different animal, and the one that took longest to place. They score model behaviour against a threshold instead of asserting a fixed output. Retrieval recall. Whether the router picks the right tool. Whether a draft is grounded in the facts it retrieved. They need credentials and they cost real money per run, so they can't sit in the per-pull-request job. They run nightly on a schedule instead.

That split has a consequence, and it took an incident to learn it. **The per-PR job runs those same eval tests with no credentials.** So every one of them skips. The job goes green. The tests exist. They execute. They assert nothing. There is now a script whose only job is to read the test log and fail if a named eval either matched nothing or skipped.

**Load.** I don't have any. No benchmarks, no load harness, nothing. The reason is that these are low-volume services and the failure mode I actually get is a wrong answer, not a slow one. But I've been wrong about my own proportions once already on this page. No measurement backs this one either.

Where does test-first sit across all this? Cleanly in unit and integration, awkwardly in end-to-end, and not at all in evals. You can't write a failing eval first in any meaningful sense, because the thing you're measuring doesn't have a right answer. It has a distribution.

## Docs, changelogs, and the rules agents read

Everything above is a check. It runs after the work and it either passes or fails. **That is only half of not-assuming.** For a long time it was the only half I had.

The other half is the written record, and I have come to rate it as highly as the tests. A check tells me something is wrong. It cannot tell me why a thing is the way it is. That's the question I actually have three weeks later, when an agent proposes changing it.

Three layers, and they do different jobs.

**Rules the agent reads at session start.** Two repositories hold these. One has my writing and career material. The other has about two hundred engineering rule files — roles, standards, review checklists, workflow prompts. The header of my check runner says why any of it exists:

> Every one of them exists because a defect got through a manual checklist.

These are not best practices I read somewhere. Every rule is a thing that already went wrong once, written down in the specific form that would have caught it. The test-first prompt I hand agents carries seven rules, and two of them are incidents I can point at:

> A test you have never seen fail is not a test — never skip the red step.

> If a test tier self-skips because infra/env is missing, treat that as a FAILURE in CI, and report the skipped count alongside "passed."

The second exists because a CI run once reported green having executed zero tests in the tier that catches the real bugs. Nothing was wrong. Nothing had run.

**Per-repository instructions.** An `AGENTS.md` at the root of a service — eight of my twelve repositories have one, which is its own small admission. It holds what is true about *that* codebase: the branch convention, the deploy targets, which directories are generated, what must never be touched. The global rules say how to work. These say where you are.

**The changelog, which is the one I'd least want to give up.** Every change batch gets an entry, recorded with the commit. Not a version bump — a written trace: what the problem was, what changed, why, and what it fixed. Four prompts, answered in sentences. It's the operational record for incident response, and it's what you read at 2am.

The reason it matters more with agents than without them is specific. **A check catches a wrong edit. The changelog catches an *unexplained* one** — the drive-by change that works, passes every gate, and that nobody can reconstruct later. That is the failure mode an agent actually has. It is also why prose is the right instrument. An agent can satisfy an assertion without understanding it. It cannot write "what the problem was" without saying something, and a thin answer shows up in the diff immediately.

The same argument applies to reasoning and findings, not just changes. An agent that is told *what* the rule is will follow it. An agent told *why* the rule exists will apply it to the case you didn't anticipate. It will also tell you when the reason no longer holds. That difference is worth the writing time.

Now the awkward part, which I only noticed while fact-checking this section. The rule file says "do not merge without it." **Nothing enforces that.** No CI step, no hook, no script. It is a sentence in a document, which is the precise thing this piece spends a whole section condemning.

And it has been followed **115 times** in one repo.

I don't think that pair rescues the "declared, not wired" argument so much as complicate it. Some rules really are held by habit. But you cannot tell from the outside which ones — the only visible difference between a convention that holds and one that quietly stopped is that somebody counted. So I counted, and it's 115, and that's the entire basis for trusting it.

A rule followed a hundred times is a practice. A rule followed twice is a comment with good intentions. You find out which one you have by looking, not by writing the rule more firmly.

This is the loop the whole setup runs on. Something breaks. I work out the general shape of it. That shape becomes a rule an agent reads at the start of every session, and where possible a check that fails. The next time the same shape shows up, the constraint is already there.

**The prompt is per-session. The rule file is permanent.** That's the entire advantage.

## Four scenarios

Two where the system told me the truth. Two where it didn't. The last two are why I distrust a green build now.

### 1. The half of the system that holds still

In one system the least glamorous piece was also the most dangerous if it broke, and it had almost no tests: the job queue that retries, gives up, and parks failed work. I wrote them. The retry backoff, the give-up rule, the recovery sweep that re-drives a stuck message.

Those tests caught a defect the model layer could never have surfaced, because it wasn't in the model. A message could be silently lost between a failed enqueue and a recovery scan that never ran. No eval would have found that. No prompt change would have fixed it.

This is the boring case. It is also the majority case. Ordinary tests, written first, on the deterministic half, paying back at the usual rate.

### 2. The decision I took away from the model

Some behaviour is too important to leave with a component that changes its mind.

The sharpest case I've hit is the "no reply" call. When a guest writes only "thanks!", the assistant should stay quiet. It's also the one action with no human review in front of it, so if the model wrongly decides a real question needs no reply, the guest is dropped in silence and nobody finds out.

The model kept getting it wrong in one direction. After a back-and-forth about housekeeping, a plain "thanks for sorting that out" would get pulled back into the housekeeping tool instead of being left alone.

I tried to fix it in the prompt. Every version fixed the "thanks" case and quietly broke a worse one: a guest answering a real question, "I can only do it on the 9th", got treated as a throwaway and dropped. I was trading a small annoyance for a silent failure, and the prompt wouldn't stop making that trade no matter how I worded it.

So I stopped asking the model. **There's no fixed assertion to write on what a model outputs, but there is one to write on what a safe output has to satisfy.** That distinction is the whole move.

The decision went into plain code. A small function that recognises an unmistakable closer, a bare "thanks" or "perfect, thank you", and only those. It refuses to fire on anything carrying a question mark, a number, a request word, or any leftover content. Any doubt at all and it does nothing, leaving the call to the model.

I wrote it test-first, and the useful detail is which tests came first. Most of them assert the negative: this is *not* a pure closer, do not suppress this. That's the direction where a mistake is unrecoverable, so that's the direction that got the coverage. Written the other way round, the suite would have been a list of strings that correctly stay quiet, all passing, while the case that drops a real guest went untested.

That's the rule I took out of it. Where a boundary must not fail, I stop asking the model to promise it. The promise moves into deterministic code, and the tests start from the failure I can't afford rather than the one I can.

### 3. Three times a green suite certified nothing

Everything above assumes the checks are telling the truth. Here is what it looks like when they aren't, from one batch of work on one system.

**A zero value is a value.** A regenerate path set a column to Go's zero UUID. That isn't SQL `NULL` — it's a real, specific id of all zeros, and there was a foreign key on the column. No row has that id, so the write violated the constraint, the transaction rolled back, and the job retried five times against a paid API before dying. Meanwhile the browser had shown its success message the moment the job was accepted.

No test caught it because that code path was dead until a new button became its first caller. Nothing had exercised it, so nothing had failed.

**Absence is `NULL`, and `NULL` is not false.** A recovery query was meant to skip message-type events, matching a field out of a JSON payload. Before shipping I checked the logic with a truth-table simulation. It passed.

The first time it ran against a real Postgres, it failed on the exact case the simulation called fine. When the JSON key is absent, the field doesn't come back as an empty string. It comes back `NULL`. And a `WHERE` clause keeps a row only when its condition is *true* — `NULL` is not true. So every event missing that key was silently dropped from recovery. The query written to rescue stuck events was quietly excluding a whole class of them.

My simulation had modelled a missing key as an empty string, which is what absence looks like in most languages. The bug wasn't in the logic. It was in the difference between how a language and a database treat "nothing there."

**A tier that skips itself reports green.** The database-backed tests skip when the connection variables aren't set, so a developer can run the quick check without standing up infrastructure. Reasonable convenience. A misconfigured CI run with those variables unset skips the entire tier and still reports green. Zero tests ran. The build looks passed.

Three failures, one shape. **Each check stood somewhere the real thing wasn't.** A green build certifies that the code does what the tests check, and for anything that only breaks at a boundary, that's a narrow guarantee wearing a broad one's clothes.

Two of those three were found by reading the real path by hand, one line at a time. The third was found by a test tier that hadn't existed until I built it.

### 4. The test that was checking a copy of itself

That was one mechanism, three times. There's a second, and it unsettles me more, because nothing about it looks like a shortcut.

This is the sabotage from the opening, and this is why it passed.

The underlying bug was worth understanding on its own. Two date defects pointed opposite ways. They cancelled on screen and compounded in the database, which is why it took a no-op save from a New York timezone to surface them. The five sabotages I tried after the fix were the obvious ones: delete the offset, swap it, invert it to a fourteen-hour error, return null, re-parse through the trap the code comment warns about. Nothing I did in staging changed a single result.

The reason was in the test file, with a comment above it explaining itself:

> Mirrors `route.ts:129` exactly. Kept as a local literal rather than imported, because the route is a Next handler that pulls in db/auth/Sentry; what needs locking is the expression, and it is one line.

Every word of that is true and the conclusion is wrong. The route was never imported by any test. The assertions were checking a copy of the line against itself, and they'd pass forever no matter what production did.

The same defect showed up inverted elsewhere. A properly written parse helper, a good test, a round-trip case. And no callers at all. The two places that actually wrote to the database had each re-typed the expression inline. My careful test was guarding an export nothing used while four live copies went unwatched.

**A test earns trust from the path between it and the code, not from the assertions inside it.** Both of the ones that fooled me had good assertions and no route to the line that could break.

So I ask two questions now, and both are cheap. What does the test import — if the thing under test isn't in that list, the test is about something else. And does a comment in it justify a shortcut, because I read those as a flag now rather than a resolution. Mine was clear, honest, and correct in every detail except the conclusion — which made the defect harder to see, because a reviewer reading it finds the question already answered.

## Five antipatterns, and the pattern that closed each one

Those are stories. Here are the shapes underneath them, because the shapes are what transfer. Every one of these is now a comment in my own CI, explaining why the tidier version was wrong.

**Scanning a path that isn't there.** A guard greps a directory for a pattern that must not appear, counts the matches, and fails if the count is above zero. Rename the directory and grep matches nothing, the count is zero, the guard passes forever. → *Assert the scan target exists before scanning it.* One `test -d` above the grep. The rename now breaks the check loudly instead of disarming it.

**Green from having run nothing.** `go test -run` with a pattern that matches no test prints "no tests to run" and exits zero. Worse, the `=== RUN` line prints *before* the test body. A `t.Skip` inside still emits it. Checking for that line proves the test exists, not that it ran. → *Assert the named test both started and did not skip.* That's a separate script reading the log, and it is the only reason I know my eval gate is a gate.

**A red step hiding the steps below it.** One sequential job, first failure cancels the rest. A check that had been failing on every run silently removed the seven steps under it. The build was red either way. Nothing distinguished "one floor slipped" from "the seven below it never ran at all." → *Every step runs unless the run was cancelled.* Not `always()` — a human hitting cancel should still stop it.

**Two commands in one step.** Same shape, smaller. Two verification commands sharing one shell block; the first fails, the second never runs, and only the first is named anywhere in the log. → *One assertion per step, each reporting its own result.* Tidiness in a CI file is not free.

**A default that qualifies the wrong subject.** A configuration value read as `secret or fallback`. When the secret was absent, the fallback silently substituted a different model from the one serving traffic. So the quality gate passed. It had measured something that wasn't in production. → *Pin the value and assert it matches the deployment config.* A fallback is a guess about what you meant, evaluated at the worst possible moment.

One shape, five costumes. **Every one of them produces a passing or failing signal that is true about the wrong thing.** Not a false negative in the usual sense — the check is fine, the check is just pointed somewhere else.

And this is the red step again, at a different scale. A test you have watched fail cannot have been written to pass; a gate you have watched fail cannot be pointed at nothing. Same control, same reason, one level up. The only difference is that nobody thinks of a CI file as something that needs a failing test first.

## Where it drifts

Now the part that undercuts everything above, because I audited it this month and it did not hold up as well as I would have told you.

**Eight repositories, one template, no standard.** I counted, and the split is cleaner and worse than I'd have guessed. Four of the agent rule files forbid pushing outright. The other four pre-authorise pushing a feature branch — and each of those four says, in its own text, that the newer rule *supersedes* the older never-push one.

So this isn't two defensible positions written months apart. It's one decision that reached half the fleet and stopped, and the half that got it is carrying the receipt. An agent obeys whichever file it read last, and both answers look authoritative.

Nothing detects this, and the reason is structural: **every check I own runs inside one repo.** A contradiction between two of them is invisible to all of them.

**A gate that assumes a step nobody takes.** The full check chain runs on push. Two of those repos have an editor integration that syncs commits to the remote without anyone running `git push`. Both files say so. It's a warning I wrote and then did not follow to its conclusion. If the commit reaches the remote without a push, the pre-push hook never fires. The heaviest gate I own is attached to an event that sometimes doesn't happen.

**A config file that nothing invokes.** One repo has a carefully reasoned linter config, sixty-odd lines. Its preamble explains that an audit found only basic vetting, and that this was the fix. Nothing runs it. Not CI, not a hook, not a script. Its own comments reference two scripts that do not exist. The finding was real. The fix was written. The wiring never happened, so the audit's conclusion is still true, with a config file standing where the enforcement should be.

**A filter that quietly narrows what "lint the code" means.** A workspace glob covering `apps/*` excludes `packages/*`. Three of those four packages define a working lint script, and none of the three ever runs.

Put those together and you get the shape that keeps recurring, in my own work and in the research both. **A control that's present, visible in the diff, reasonable in review, and not running.** Adding a config file is not adding a gate. The gate is the line in CI that fails, and a well-argued config is the easiest place to forget it.

My own tooling does it too, at a smaller scale. I keep five scripts that scan my writing for claims contradicting each other across files. One of them went from reporting sixteen problems to reporting none, right after I changed a line in it. The change had broken what the script was reading, so it was scanning an empty string, and finding nothing in an empty string is the correct answer. It exited clean.

I caught it only because sixteen to zero is not a believable improvement. Four of the five had self-tests. The one that broke is the one that didn't, and it broke in the direction that looks like good news.

## What this costs

The honest ledger, since the piece has been mostly favourable.

**Review is the bottleneck and it does not get easier.** I generate more than I can carefully read, and the gap is the risk. Measured across 220,000 closed pull requests, agent-authored ones merge at rates from 43% to 84% depending on the agent. Humans sit at roughly 85%. Split each repo's own agentic history into four, and the merge rate barely moves between the first slice and the last: **no clear improvement trend**. Humans are still gating hard, and they should.

**Two hundred rule files is a maintenance surface.** I don't re-read them. Some are certainly stale. And the repository that holds two hundred rules for how agents should behave has **no agent config of its own** — no `AGENTS.md`, nothing. I noticed that while writing this.

**Writing the record costs real time.** The changelog schema is four questions per change batch, and answering them properly takes longer than the change sometimes does. I think it pays. I can't prove it pays, and on a fast-moving team with a different tolerance it might not.

**I can't tell you the whole thing works.** I have no controlled comparison, no baseline, no measurement of my own defect rate before and after. The most rigorous study on the question found experienced developers **19% slower** with AI. Its own authors have since walked back the scope. The follow-up reports −18% and −4%, with confidence intervals that both cross zero (meaning the true effect could just as easily be nothing at all), and notes the design likely understates the benefit because the enthusiasts opted out. I cite it as a caution, not as proof of anything, including my own setup.

**And it is one person's.** Every constraint here was written by the person it constrains. I have never tested any of it against a colleague who disagrees, a codebase I did not design, or a review culture that is not mine.

## What I'd keep

If I started again tomorrow, this is the short list — the favorites, the pieces that earned their place enough times that I stopped second-guessing them.

- **One monorepo over one repository per service.** Turborepo and pnpm workspaces on the TypeScript side, a Go workspace file on the Go side. It keeps services in sync and makes a cross-service change one reviewable diff instead of two.
- **One root command that runs every check**, and every check exits non-zero. Format, lint, types, tests, secret scan, dependency audit, dead code. A warning is not a gate.
- **Vitest and Playwright** for unit and end-to-end, `go test -race` for the Go half, and integration tests against a real database rather than a mock of one.
- **Gitleaks, the package audit, and Knip** — secrets, dependency health, dead code. Three different kinds of rot, three different scanners.
- **Evals as their own tier**, on a schedule rather than per-pull-request, with a separate assertion that they actually ran.
- **A changelog entry per change batch**, in prose: what the problem was, what changed, why, what it fixed.
- **Rules and reasoning the agent reads at session start**, in version control, alongside a per-repository `AGENTS.md`.
- **A personal knowledge base behind all of it**, so a lesson learned once becomes a constraint that applies automatically the next time.

The through-line is the same in every item. Each one replaces something I would otherwise assume. That the change is safe. That the tests ran. That the rule was followed. That somebody remembered why. **I would rather have control over the direction than trust what an agent tells me it did.**

## The part that actually matters

Not the setup. The loop that produced it.

Every rule I have came out of the same sequence. Something broke. I worked out the general shape of it, wrote that shape into the file an agent reads at session start, and where I could, turned it into a check that exits non-zero. Not a comment saying to be careful. A line in CI.

Run that loop enough times and what comes out fits your own failures instead of mine. Which is why I'd put almost no value on my specific rules travelling anywhere. The loop travels. The rules are just what it left behind in one codebase.

The step I keep relearning is the last one: confirming a check can actually fail. Break the thing it guards on purpose, in staging, watch it go red, put it back. That's the sabotage this piece opened with, and it's the cheapest hour in the whole setup. I hadn't spent it on one of my five scripts, and that's the one that broke. What caught it wasn't the system. It was me finding a number implausible, which is not a control and does not scale.

It has a self-test now. Writing this is what made me go and add it, which is either a good argument for writing things down or an embarrassing one for how long it sat there. Probably both.

So the whole thing reduces to one idea at three scales. **A test I have watched fail. A gate I have watched fail. A written record of why a thing exists**, so the next person, or the next agent, doesn't have to work it out from scratch. The scales I neglected are exactly where the holes turned out to be.

None of this is a silver bullet and I'm not offering it as one. It's a set of trades that fit low-volume services, one reviewer, and a tolerance for saying no to my own tools. The checks worth keeping are the ones that match where the bugs actually come from.

What I can say plainly is the thing in the title. I no longer assume the code is right because it's green, or that a rule held because it's written down. Both of those turned out to be assumptions I was making without noticing, and finding them cost me more than building any of the checks did.
