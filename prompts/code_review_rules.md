# Elixir / Phoenix Code Review Guidelines

*For human reviewers. Static tools (Credo, Dialyzer, Sobelow) catch syntax and type-level issues. These rules cover the things that require understanding intent, data flow, and runtime behavior — the stuff a linter can't see.*

## LiveView Architecture

**1. No unconditional DB queries in mount**
mount/3 runs twice per connection (once before the socket connects, once after). A plain `Repo.get/all/one` call there fires twice, wasting a query and risking a race. Look for `Repo` calls that aren't wrapped in `if connected?(socket)` or handled via `assign_async`. For SEO-critical routes, the disconnected render IS what crawlers see, so it needs a cache-backed fallback instead of skipping the query.

**2. Use streams for lists over ~100 items**
Regular assigns keep the full list in memory per connected user and get re-diffed on every render. Watch for `assign(socket, :items, large_collection)` with collection names like items, entries, records, users, posts, orders, notifications, etc. Not every list needs this, but the pattern is worth a second look — flag it for a human decision rather than assuming it's fine.

**3. PubSub subscribe without a matching unsubscribe**
Subscribing in mount (or in a LiveComponent) without unsubscribing on unmount, or relying purely on process death to clean up, can leak subscriptions across reconnects and LiveComponent re-mounts. Check that subscriptions are torn down deliberately, especially in components that mount/unmount frequently.

**4. Keep socket assigns lean** `NEW`
Large structs, binaries, or deeply nested maps stored directly in assigns get copied and diffed on every render. If an assign is large or rarely needed by the template, consider keeping a reference (ID, ETS key) instead of the full value.

**5. Match component type to actual need** `NEW`
A stateful `live_component` used purely for code organization (no independent state or events) is unnecessary ceremony. Conversely, a function component being asked to manage its own event handling is a sign it should be a LiveComponent. Check that the choice matches the actual behavior needed.

**6. Validate uploads server-side** `NEW`
Client-supplied file extensions and MIME types can't be trusted. Check that accepted types are enforced server-side (not just in the `accept:` attribute), and that `consume_uploaded_entries` cleans up temp files even on failure paths.

**7. Capture locale before spawning**
Gettext/CLDR locale lives in the process dictionary, so a Task or GenServer starts with the default locale, not the caller's. Look for `Task.async(fn -> ... end)` or `GenServer.start_link` that call locale-sensitive functions without capturing the locale in a variable first and passing it into the closure.

## Ecto Query Design

**8. Separate queries for has_many, JOIN for belongs_to**
Joining across a `has_many` association multiplies the parent row for every child row returned — a subtle bug that only shows up with real data volume. `has_many` should generally use a separate query + preload; `belongs_to` is safe to JOIN since there's no multiplication risk. Look for `join: x in assoc(parent, :children)` where children is a `has_many`.

**9. Deduplicate before cast_assoc with shared data**
If the same child record can appear under more than one parent association, dedupe the list before it reaches `cast_assoc`. Otherwise you'll see constraint violations or duplicate inserts that are hard to trace back to the actual cause.

**10. Watch for N+1s from missing preloads** `NEW`
This is distinct from the JOIN/has_many rule above — it's about templates or downstream code accessing an association that was never preloaded, silently firing one query per row. Check that any association touched outside the original query context was explicitly preloaded.

**11. No raw string interpolation into fragments** `NEW`
String interpolation into `fragment/1` is a SQL injection path that Sobelow doesn't reliably catch, since it looks like ordinary Ecto code. Any dynamic value inside a fragment should go through bound parameters, not string building.

**12. Wrap multi-step writes in Ecto.Multi or a transaction** `NEW`
A sequence of dependent inserts/updates that isn't transactional can leave the database in a half-written state if one step fails partway through. Look for multiple `Repo` calls in a row that logically belong together but aren't wrapped in `Ecto.Multi` or `Repo.transaction`.

**13. Don't use the bang variants for expected absence** `NEW`
`Repo.get!` / `Repo.one!` turn a normal 'not found' business case into an unhandled crash (and a 500). Reserve the bang versions for cases where absence really is a bug, not a possible outcome.

## Authorization & Multi-Tenancy

*This is the category static analysis is weakest on, and where real Phoenix apps get burned most often. Give it real attention.*

**14. Scope every lookup to the current user/tenant** `NEW`
A context function like `get_order(id)` that isn't also scoped by `current_user` or `current_account` will happily return someone else's data if the ID is guessable — a classic IDOR. Check that every fetch-by-id function also takes and enforces an ownership/tenant key.

**15. Don't trust client-supplied ids in handle_event** `NEW`
Event params come from the client and can be edited before they reach the server. Re-verify permission/ownership server-side inside the handler rather than assuming an id in `socket.assigns` or the event payload is still valid for this user.

## OTP Architecture

**16. No process without a runtime reason**
Processes exist to model concurrency, isolate state, or contain failure — not to organize code into a nicer-looking module. If a GenServer just delegates every call straight to another module, ask what actually happens if it dies, and whether that's the behavior you want. If not, it's probably unnecessary.

**17. Supervise every long-lived process**
Any `GenServer.start_link`, `Agent.start_link`, or `Task.start_link` meant to live for a while needs to sit under a supervisor — otherwise a crash just disappears instead of restarting. `Task.async` is fine unsupervised for short-lived work; `Task.start` is not. Check `application.ex` covers everything long-lived.

**18. Add a catch-all handle_info clause** `NEW`
GenServers receive messages from monitors, timers, and other unexpected sources beyond the ones you wrote handlers for. Without a fallback clause, one of these crashes the process. Add a catch-all, even if it just logs and ignores.

**19. Check GenServer.call timeouts in hot paths** `NEW`
The default 5-second timeout can turn a slow downstream dependency into a crashed caller under load. In hot paths, make the timeout an explicit, deliberate choice rather than the default.

## Oban Job Design

**20. Jobs must be idempotent**
Oban retries failed jobs, so anything non-idempotent (charging a payment, sending an email, an insert without a unique constraint) can fire twice. Look for `Repo.insert` (not `insert!`) being used without a `conflict_target`, and for jobs that should only ever run once per key but don't declare `unique: [fields: ...]`.

**21. Pass ids in job args, not full structs** `NEW`
Serializing a whole record into a job's args means the job executes against stale data days later if anything changed. Pass an id and re-fetch fresh state at execution time instead.

**22. Match retry/backoff to the operation** `NEW`
Not everything should retry indefinitely with default backoff — a payment charge without an idempotency key retried five times is a real risk. Check that `max_attempts` and backoff are a deliberate choice, not just whatever Oban defaults to.

## Migrations & Schema

**23. Check migrations are safe to deploy** `NEW`
A column drop or rename that happens in one step can break a running instance of the old code during deploy. For anything beyond additive changes, check whether it needs to be split into a backward-compatible multi-step migration.

**24. Keep schema and migration in sync** `NEW`
A migration that changes a column without the corresponding Ecto schema being updated in the same PR (or vice versa) is easy to miss and causes confusing failures later. Check both sides land together.

## Behavioral & Workflow

**25. Check changeset errors before UI debugging**
If a form submits with no visible error but the expected side effect didn't happen, look at the `{:error, changeset}` branch first, before chasing it as a UI bug. A silently-ignored error branch is a very common root cause.

**26. Hidden inputs for required embedded fields**
Any required field in an embedded schema that isn't directly editable in the form still needs a `hidden_input`, or it simply won't be submitted — and validation will fail without an obvious reason. Check `inputs_for` blocks for embedded schemas against the schema's required fields.

**27. Facade third-party library calls**
Direct calls to HTTPoison, Tesla, ExAws, or similar from business-logic modules make the code hard to test and hard to swap later. Push the dependency behind a project-owned wrapper module (`MyApp.HTTP`, `MyApp.Storage`) that can be mocked in tests.

**28. Verify before claiming a fix works**
Don't sign off on a change as correct without actually running `mix compile && mix test`. If tests can't be run in the current context, say explicitly what's unverified rather than implying it's been checked.

**29. Comments explain code, not history**
A comment explaining why a change was made — issue links, PR references, "we used to do X" — belongs in the commit message or ticket, not the source. Keep inline comments for durable facts: footguns, invariants, non-obvious behavior. "TODO: fix this" goes in a ticket; "Don't remove — causes X" stays in the code.

**30. No secrets or PII in code, logs, or telemetry** `NEW`
Hardcoded API keys or tokens (even in dev config) and `Logger.info`/telemetry calls that dump entire structs can leak credentials or personal data into logs. Check that config secrets come from env vars, and that logged/traced data is scrubbed of sensitive fields.

## Severity & Verdict Reference

- **BLOCKER**: data loss, security hole, silent corruption, or a race condition.
- **WARNING**: suboptimal pattern, likely issue under load, or a maintainability concern.
- **SUGGESTION**: style or clarity improvement, non-critical.

**Verdicts**: PASS (nothing but suggestions) · PASS WITH WARNINGS (warnings only, nothing critical) · REQUIRES CHANGES (coverage gaps or unmet requirements) · BLOCKED (blockers present).

*Findings on code the diff didn't touch are marked PRE-EXISTING and noted but don't affect the verdict.*
