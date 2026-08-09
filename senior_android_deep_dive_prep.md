# Senior Android Deep-Dive Prep: Coroutines, Flow, Clean Architecture, Compose, DI

How to use this: read a section, then close the file and explain it out loud in your own words before checking. Fill in the "Your Example" blanks with real projects — that's the part that actually gets you hired. We'll mock-interview once you've been through it.

---

## 1. KOTLIN COROUTINES

### Q: What's wrong with launching a coroutine in GlobalScope vs viewModelScope?
**Answer:** GlobalScope is tied to the application's lifetime, not any component's. A coroutine launched there survives even if the ViewModel or Activity that started it is destroyed — it keeps running, holds references (leak risk), and can crash trying to update UI that no longer exists. `viewModelScope` is cancelled automatically when the ViewModel is cleared, so structured concurrency handles cleanup for you. The deeper point: GlobalScope breaks structured concurrency — there's no parent to cancel it, no way to propagate exceptions to a handler, no automatic cleanup. It's essentially opting out of the coroutine cancellation graph.

**Follow-up they might ask:** "When would GlobalScope ever be correct?" — Rare cases: fire-and-forget work that must outlive the caller (e.g., flushing analytics on app close). Even then, most teams prefer a dedicated `CoroutineScope` with `SupervisorJob` tied to application lifecycle, not raw GlobalScope, because it's still testable and cancellable.

### Q: launch vs async — what happens to exceptions?
**Answer:** `launch` propagates exceptions immediately to its parent (and up the job hierarchy) as soon as they occur — this is why an uncaught exception in a `launch` block crashes the app unless handled. `async` defers exceptions until you call `.await()` — the exception is stored in the Deferred and only thrown when you retrieve the result. This means an exception in an `async` block that's never awaited can silently disappear. This is a real bug source: people assume errors surface automatically like `launch`, but `async` requires you to await to see them.

**Follow-up:** "Where does CoroutineExceptionHandler not work?" — It doesn't catch exceptions from `async` (they're stored, not propagated to the handler) and it only works when installed on a root coroutine — nested handlers are ignored because exceptions propagate up to the parent first.

### Q: Is cancellation cooperative? What if a coroutine is doing a blocking call when cancelled?
**Answer:** Yes — cancellation in Kotlin coroutines isn't preemptive, it's cooperative. Calling `cancel()` sets the job's state to "cancelling" but the coroutine only actually stops when it hits a suspension point that checks for cancellation (like `delay()`, or any suspend function that calls `ensureActive()`/`yield()`). If your coroutine is running a tight CPU loop or a blocking call (like a synchronous network call via a non-suspending API), it will NOT be cancelled until that blocking call finishes — this is a classic leak/ANR-adjacent bug. Fix: check `isActive` in loops, use cancellable suspend functions, or wrap blocking calls with `withContext` + ensure the underlying API supports interruption.

### Your Example (fill in):
> Describe a real bug you hit involving scope, cancellation, or exception handling in coroutines — what broke, how you found it, how you fixed it.

---

## 2. FLOW

### Q: StateFlow vs SharedFlow vs LiveData — how do you choose?
**Answer:**
- **StateFlow**: holds a current value, always has one, new collectors get the latest value immediately (conflated). Best for UI *state* — "here's what the screen looks like right now" (loading/success/error, form fields, etc).
- **SharedFlow**: no required initial value, configurable replay and buffering, doesn't conflate by default. Best for *events* — one-time things like "show a snackbar," "navigate to screen X" — where you don't want the latest value redelivered on rotation.
- **LiveData**: lifecycle-aware by default, but lacks Flow's operators (map/combine/debounce chains are clunkier) and doesn't handle backpressure the same way. Most teams building in Compose/Flow-first codebases have moved off LiveData except at legacy boundaries.

**The trap interviewers are checking for:** using StateFlow for one-time events causes the event to "replay" on every new collector (e.g., a snackbar re-shows after rotation because the new collector sees the last emitted value). That's the classic senior-level gotcha.

### Q: Cold vs hot flows — what problem does a cold flow cause if you're careless?
**Answer:** A cold flow's producer block runs independently *for each collector* — nothing runs until collection starts, and it restarts from scratch for every new collector. If you build a Flow around a network call and two parts of your UI collect it, you'll fire the network call twice, redundantly. Fix: convert to a hot flow with `.stateIn()` or `.shareIn()` with appropriate sharing strategy (`SharingStarted.WhileSubscribed(5000)` is the common pattern in ViewModels) so the upstream work is shared across collectors instead of re-executed.

### Q: flowOn vs withContext inside a Flow builder?
**Answer:** `flowOn` affects everything *upstream* of where it's declared in the flow — it changes the context for the producer, not the collector. `withContext` inside a flow builder changes the context only for that specific block of code. People misuse this by putting `flowOn` in the wrong position and being confused why context switching didn't apply where they expected — the operator's position in the chain matters, unlike most sequential code.

### Q: conflate vs buffer vs collectLatest?
**Answer:** All three exist to handle a producer emitting faster than a collector can process.
- **buffer()**: queues emissions, collector processes all of them eventually, nothing dropped, but memory grows if producer is much faster.
- **conflate()**: drops intermediate values if the collector is busy — only the latest value survives when the collector catches up.
- **collectLatest**: cancels the *in-progress collector block* itself when a new value arrives, restarting it with the new value.
The wrong choice causes either dropped critical events (using conflate when you needed every emission) or wasted work/race conditions (using collectLatest when the collector had side effects that shouldn't be cancelled mid-way, like a partial DB write).

### Q: How do you test a StateFlow-based ViewModel?
**Answer:** Use `runTest` from kotlinx-coroutines-test to control the test dispatcher deterministically, inject a `TestDispatcher` (like `StandardTestDispatcher`) instead of `Dispatchers.Main` via `Dispatchers.setMain()`, and use Turbine to collect and assert on flow emissions (`stateFlow.test { assertEquals(...) ; awaitItem() }`). The key point to make: testing flows without controlling the dispatcher leads to flaky tests because real coroutines run asynchronously — determinism comes from the test dispatcher, not from the flow itself.

### Your Example (fill in):
> Describe a bug caused by cold-flow re-execution, StateFlow replay, or wrong backpressure operator choice.

---

## 3. CLEAN ARCHITECTURE

### Q: What actually crosses a layer boundary, and why map domain models to UI models?
**Answer:** Only the domain layer's models/interfaces should cross into the presentation layer — never raw data-layer models (e.g., a Retrofit DTO or Room entity). You map DTO → domain model → UI model because each layer has different concerns: a DTO's shape is dictated by the API contract (can change without warning, may have nullable fields for JSON leniency), the domain model represents your app's actual business concept, and the UI model is shaped for what the screen needs to render (formatted strings, display flags). Reusing one model everywhere means a backend API change ripples all the way to your Compose UI, and testing becomes harder because you can't fake a domain layer without pulling in networking types.

### Q: When is Clean Architecture overkill?
**Answer:** For a small feature or prototype where the domain logic is trivial (mostly CRUD, one data source, unlikely to change), full layer separation with use cases wrapping single repository calls adds boilerplate without adding safety — you end up with a use case that's a one-line pass-through, which is ceremony, not architecture. The judgment call: apply full separation where there's real business logic, multiple data sources, or code that needs independent testability; for a thin settings screen backed by one DataStore call, a simpler ViewModel-to-repository call is defensible. Saying "we always do full Clean Architecture everywhere" without this nuance actually signals *less* senior judgment, not more — interviewers are listening for whether you can justify not over-engineering.

### Q: How do you decide use case granularity?
**Answer:** A use case should represent a single, named business action, not a wrapper around a repository method. If you find yourself with a use case called `GetUserUseCase` that just calls `repository.getUser()` with no additional logic, that's suspicious — ask whether it needs to exist separately from the repository, or whether it should combine logic (e.g., "get user, then check permission level, then decide default screen") that's actually a meaningful, testable unit of business behavior. Prefer verb-based names that describe a business operation, not a data operation.

### Q: How do you keep implementation errors from leaking across layers?
**Answer:** Catch data-layer exceptions (like `HttpException`, `IOException`) at the repository boundary and map them to a domain-level sealed error type (e.g., `sealed class DomainError { NetworkError, NotFound, Unknown }`). The UI layer should never know or care that Retrofit or Room exists — it only reacts to domain errors it understands. This also makes the UI testable without needing to fake networking exceptions.

### Your Example (fill in):
> Describe a real architecture decision — where you drew a layer boundary, or a case where you deliberately skipped full Clean Architecture and why.

---

## 4. JETPACK COMPOSE

### Q: What actually triggers recomposition, and how do you debug unnecessary recompositions?
**Answer:** Recomposition is triggered when a `State` object read inside a composable changes value. Compose uses "smart recomposition" — only composables that actually read the changed state re-run, not the whole tree, *provided* the compiler can prove the inputs are stable. Instability is the usual cause of unnecessary recomposition: if you pass an unstable type (like a plain `List` or a data class with a `var`, or a lambda capturing changing state without proper keys) as a parameter, Compose can't skip recomposition safely and re-runs more than needed. Debugging tools: Layout Inspector's recomposition counts, adding `@Stable`/`@Immutable` annotations where correct, using the Compose compiler metrics report to find which composables are marked unstable/skippable, and checking for lambdas that aren't remembered and thus get recreated every recomposition.

### Q: remember vs rememberSaveable vs hoisting to ViewModel?
**Answer:** `remember` survives recomposition but not configuration change (like rotation) — fine for transient UI-only state (e.g., whether a tooltip is expanded). `rememberSaveable` survives configuration change via the SavedStateHandle/Bundle mechanism (or a custom `Saver`) — use it for state that should survive rotation but doesn't need to survive process death with business meaning (like scroll position, a text field's typed-but-unsaved value). Hoisting to ViewModel is for state that represents actual app/business state, needs to survive process death robustly (via `SavedStateHandle`), needs to be shared across composables, or needs to be testable independent of the UI. The gotcha: putting business-critical state only in `remember` means it's silently lost on rotation, which won't show up in a demo but will show up in production reviews.

### Q: LaunchedEffect vs SideEffect vs DisposableEffect?
**Answer:**
- **LaunchedEffect**: runs a suspend block tied to composition, restarts if its keys change, cancelled when it leaves composition. Use for one-shot suspend work triggered by composition (e.g., fetching data when a screen appears, showing a snackbar once).
- **SideEffect**: runs on every successful recomposition, not suspendable. Use for syncing Compose state to a non-Compose system (e.g., updating an Analytics SDK's current screen name).
- **DisposableEffect**: for effects that need explicit cleanup when leaving composition or when keys change — e.g., registering/unregistering a listener (BroadcastReceiver, sensor listener). Forgetting the `onDispose {}` block here is the classic bug — it causes leaked listeners.

### Q: What causes a Compose screen to lag, and how do you profile it?
**Answer:** Common causes: unstable parameters causing excessive recomposition, heavy work (like formatting/parsing) happening inside composition instead of hoisted/remembered, unnecessarily large recomposition scope (not using `key()` or splitting composables so only small parts re-run), or expensive layout/measure passes (deeply nested layouts, using `Modifier.weight` in ways that force extra measure passes). Profiling: Android Studio's Layout Inspector for recomposition counts/skips, the Compose Compiler Metrics report (build-time report showing stability of each composable), and standard CPU profiling (Macrobenchmark / systrace) for frame timing under real interaction.

### Q: How do you collect a Flow safely in Compose without leaking?
**Answer:** Use `collectAsStateWithLifecycle()` (from lifecycle-runtime-compose) instead of the plain `collectAsState()`. The lifecycle-aware version stops collecting when the composable's lifecycle drops below STARTED (e.g., app backgrounded) and resumes when it comes back — plain `collectAsState()` keeps collecting regardless of lifecycle state, which wastes resources and can cause state updates to arrive for a screen that's not visible.

### Your Example (fill in):
> Describe a real performance issue you diagnosed in Compose — what the symptom was, how you found the cause, what you changed.

---

## 5. DEPENDENCY INJECTION

### Q: Hilt vs manual DI vs Koin — real tradeoffs?
**Answer:**
- **Manual DI**: zero magic, fully compile-time-checked, no reflection/codegen overhead, but scales poorly — as the graph grows, wiring becomes tedious and error-prone by hand, especially across feature modules.
- **Hilt**: compile-time code generation (via annotation processing/KSP), catches missing dependencies at build time, integrates directly with Android component lifecycles (`ViewModelComponent`, `ActivityComponent`, etc.) which is genuinely valuable on Android specifically. Tradeoff: build time cost from annotation processing, and the generated code / component hierarchy has a learning curve.
- **Koin**: runtime DI via a DSL, no code generation so faster builds, easier to learn, but dependency resolution errors surface at *runtime* rather than compile time — a missing binding crashes the app instead of failing the build. Also has some runtime performance cost (service locator pattern under the hood) compared to Hilt's compile-time generated code.
The senior-level answer isn't "Hilt is best" — it's articulating the actual cost/benefit (compile-time safety and Android lifecycle integration vs build speed and simplicity) and when each fits.

### Q: What breaks if you scope something to SingletonComponent when it should be ViewModelComponent?
**Answer:** A dependency scoped to `SingletonComponent` lives for the entire application process. If it holds state that should be scoped to a single screen/flow (e.g., a mutable cache or a StateFlow representing one particular flow's progress), scoping it too broadly means that state persists and leaks across unrelated usages — e.g., a user starts a checkout flow, backs out, starts a new one, and sees stale state from the first attempt because the "singleton" never got recreated. It's a memory leak risk too, since app-scoped singletons never get garbage collected until process death. The fix is choosing the narrowest scope that matches the actual lifecycle the dependency's state should follow.

### Q: How do you test code that uses Hilt-injected dependencies?
**Answer:** For unit tests, you typically bypass Hilt entirely and construct the class under test manually with fake/mock dependencies passed directly — Hilt's DI graph isn't needed for pure unit tests. For instrumented/UI tests where you want the real Android component graph, use `@HiltAndroidTest` with a custom test application and `@TestInstallIn` to swap real modules for fake ones (e.g., replacing a real network module with one that returns canned responses). Prefer fakes over mocks where behavior matters (a fake repository with in-memory data) and reserve mocks for simple interaction verification.

### Q: How do you structure Hilt modules across feature modules without circular dependencies?
**Answer:** Define shared contracts (interfaces) in a core/common module that feature modules depend on, and have feature modules provide their own implementations bound via Hilt in their own modules — this way feature modules depend downward on core, never on each other directly. If feature A needs something from feature B, that's a sign the shared logic should be extracted into a core module both depend on, rather than creating a feature-to-feature dependency. This is really a modularization architecture question wearing a DI hat — the DI setup follows from correct module dependency direction, not the other way around.

### Your Example (fill in):
> Describe a real DI scoping bug, or a decision about which DI framework/approach to use and why.

---

## Before the Mock Interview

For each of the 5 topics, have at least one real "Your Example" filled in with:
1. What you were building
2. What specifically went wrong or what decision you had to make
3. What you tried, what didn't work, what you landed on
4. What you'd do differently now (shows growth, not just competence)

When you're ready, tell me and we'll run a live mock — I'll ask follow-ups the way a real interviewer would, including pushing on the parts of your answer that are vague.
