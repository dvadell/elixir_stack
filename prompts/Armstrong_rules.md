# Armstrong's Programming Rules and Conventions — Adapted for Elixir

*Adapted from Joe Armstrong's 2003 PhD thesis, "Making reliable distributed systems in the presence of software errors" (Appendix B: Programming Rules and Conventions). Original examples were written in Erlang; here they are rewritten for Elixir, with notes on how directly each rule carries over.*

---

## 3. Software Engineering Principles

### 3.1 Export as few functions as possible from a module

**Fully applicable.** In Elixir, every `def` is public and every `defp` is private to the module. A module with a small, deliberate public API (a handful of `def`s) is easier to understand and safer to refactor than one that exposes everything. Keep implementation details `defp`.

```elixir
defmodule MyApp.Queue do
  # public API
  def new(), do: []
  def add(item, queue), do: queue ++ [item]
  def fetch(queue), do: do_fetch(queue)

  # private helpers
  defp do_fetch([h | t]), do: {:ok, h, t}
  defp do_fetch([]), do: :empty
end
```

### 3.2 Try to reduce intermodule dependencies

**Fully applicable.** A module that calls into many other modules is fragile: every change to those modules' interfaces forces you to re-check every call site. Keep the graph of module dependencies as tree-like as possible, and avoid cycles between modules (the compiler will warn about some cyclic dependencies, but logical cycles at the design level are still worth avoiding).

### 3.3 Put commonly used code into libraries

**Fully applicable.** Group functions by *what they operate on*, not by convenience. `MyApp.ListUtils` with only list-manipulation functions is good; `MyApp.Utils` mixing list helpers and math helpers is not. Prefer pure functions in these libraries — side-effect-free code is far more reusable.

### 3.4 Isolate "tricky" or "dirty" code into separate modules

**Fully applicable, with updated examples of "dirty."** In Elixir/Erlang, "dirty" code includes things like:

* Using the process dictionary (`Process.put/2`, `Process.get/1`).
* Using `:erlang.process_info/1` or similar introspection for non-debugging purposes.
* Reaching for mutable ETS/`:persistent_term` state instead of passing data explicitly.
* Any use of `Process.info`, raw message sends, or NIFs that bypass normal supervision/OTP conventions.

Concentrate the dirty code in its own module, document its side effects clearly, and keep the rest of the system pure.

### 3.5 Don't make assumptions about what the caller will do with the results of a function

**Fully applicable.** Return a value describing what happened; let the caller decide how to react (log it, raise, retry, ignore it).

Don't do this:

```elixir
def do_something(args) do
  case check_args(args) do
    :ok ->
      {:ok, do_it(args)}

    {:error, reason} ->
      # Don't do this
      IO.puts("* error: #{format_the_error(reason)}")
      :error
  end
end
```

Do this instead:

```elixir
def do_something(args) do
  case check_args(args) do
    :ok -> {:ok, do_it(args)}
    {:error, reason} -> {:error, reason}
  end
end

def error_report({:error, reason}) do
  format_the_error(reason)
end
```

The caller decides whether and how to print, log, or convert the error descriptor.

### 3.6 Abstract out common patterns of code or behaviour

**Fully applicable.** Duplicated logic should become a shared function — or, where the duplication is structural rather than just repeated code, a shared **behaviour** or set of **macros**. Elixir gives you extra tools here beyond plain Erlang: `defmacro`, `use`/`__using__`, and behaviours (`@behaviour`, `@callback`) are the idiomatic way to factor out a repeated *shape* of code, not just repeated *values*. Use them, but don't reach for macros before a plain function will do.

### 3.7 Top-down

**Fully applicable**, unchanged in spirit. Design the public shape of your module and its high-level functions first; let low-level/private helper functions and data representation follow from that, rather than starting from data structures and building up.

### 3.8 Don't optimize code

**Fully applicable.** Make it correct first. Elixir/BEAM code that "looks slow" (e.g., using `Enum` instead of hand-rolled recursion, or `String` functions on binaries) is very often fast enough; profile with tools like `:timer.tc/1`, `Benchee`, or `:observer` before optimizing, and only after correctness is established.

### 3.9 Use the principle of "least astonishment"

**Fully applicable.** Consistent naming and consistent behaviour across modules (e.g., all "fetch" functions return `{:ok, value} | :error`, all "get" functions raise) reduce astonishment. If a function's behavior surprises you, either it's solving the wrong problem or it's named wrong.

### 3.10 Try to eliminate side effects

**Fully applicable**, and arguably even more central in Elixir, where immutability is enforced by the language itself (you can't mutate a data structure in place — you always get a new one back). Still, effectful operations exist: I/O, `GenServer` state, ETS, the process dictionary, message sends. Maximize pure functions; concentrate effects in clearly-identified places (the edges of your system, or explicit `GenServer` callbacks) and document them.

### 3.11 Don't allow private data structures to "leak" out of a module

**Fully applicable**, and Elixir gives you a stronger tool for it than Erlang did: **structs**. A naive queue implemented as a bare list forces callers to know the representation:

```elixir
# Don't do this
new_q = []
queue1 = MyQueue.add(:joe, new_q)
queue2 = MyQueue.add(:mike, queue1)
```

Better — hide the representation behind constructor and accessor functions:

```elixir
defmodule MyQueue do
  defstruct front: [], back: []

  def new(), do: %__MODULE__{}

  def add(item, %__MODULE__{front: front, back: back} = _q) do
    %__MODULE__{front: [item | front], back: back}
  end

  def fetch(%__MODULE__{front: [], back: []}), do: :empty

  def fetch(%__MODULE__{front: [], back: back}) do
    fetch(%__MODULE__{front: Enum.reverse(back), back: []})
  end

  def fetch(%__MODULE__{front: [h | t], back: back}) do
    {:ok, h, %__MODULE__{front: t, back: back}}
  end

  def len(%__MODULE__{front: front, back: back}) do
    length(front) + length(back)
  end
end
```

```elixir
new_q = MyQueue.new()
queue1 = MyQueue.add(:joe, new_q)
queue2 = MyQueue.add(:mike, queue1)
MyQueue.len(queue2)  # don't do `length(queue2)` — it isn't a plain list!
```

Because the struct's internal shape is only touched inside `MyQueue`, you're free to change the representation later (as in the original example, switching to a two-list "banker's queue" for O(1) amortized operations) without breaking any caller.

### 3.12 Make code as deterministic as possible

**Fully applicable.** If you need to start several independent things (e.g., a `Supervisor`'s children) and order doesn't logically matter, prefer starting and verifying them one at a time over kicking them all off concurrently with `Task.async_stream` or similar — determinism makes bugs reproducible. (Once you *are* confident in correctness, controlled concurrency for startup is a legitimate performance optimization — see 3.8: get it right first.)

### 3.13 Do not program "defensively"

**Fully applicable**, and it maps directly onto Elixir's "let it crash" philosophy. Trust your input at internal boundaries; validate at the edges where data enters the system (e.g., in a Phoenix controller, or where external input first hits your app). Pattern-match aggressively and let a `MatchError`/`FunctionClauseError` happen if invariants are violated — a supervisor will restart the process.

```elixir
# Option is :all | :normal
def get_server_usage_info(option, pid) when is_pid(pid) do
  case option do
    :all -> get_all_info(pid)
    :normal -> get_normal_info(pid)
  end
end
```

This will raise a `CaseClauseError` if `option` is neither `:all` nor `:normal` — and it should. The caller is responsible for supplying correct input.

### 3.14 Isolate hardware interfaces with a device driver

**Applicable, reframed.** Elixir rarely talks to hardware directly (that's more the domain of Erlang NIFs/ports, or Nerves for embedded work), but the principle generalizes cleanly to **any external boundary**: wrap external services, hardware, or OS resources behind a `GenServer` or a small dedicated module so the rest of the system interacts with them the same way it interacts with any other Elixir process — via messages/function calls, with predictable error behavior. This is exactly the pattern behind libraries like `Nerves` (hardware), `:gen_tcp`-based clients, or a `Port`-wrapping `GenServer`.

### 3.15 Do and undo things in the same function

**Fully applicable**, and Elixir's `File.open/2` + `with`/`after` make it easy to follow. Open and close a resource in the same function, or use `File.open!/2` with a function argument that closes automatically:

```elixir
# Recommended: File.open/2 with a fun closes the file for you
def do_something_with(file) do
  File.open(file, [:read], fn stream ->
    doit(stream)
  end)
end
```

Or if you must manage it manually, keep open/close paired and visible in one place:

```elixir
def do_something_with(file) do
  case File.open(file, [:read]) do
    {:ok, stream} ->
      result = doit(stream)
      File.close(stream)
      result

    error ->
      error
  end
end
```

Don't scatter `File.close/1` deep inside some unrelated helper function three calls away from where the file was opened — it becomes unclear which file is being closed and easy to leak a handle on an error path.

---

## 4. Error Handling

### 4.1 Separate error handling and normal-case code

**Fully applicable**, and this is one of the pillars of idiomatic Elixir/OTP: "let it crash." Write the happy path. If something unexpected happens, let the process crash and let a **supervisor** restart it, rather than defensively catching and patching up errors inline. Recovery logic belongs in a different process (the supervisor), not woven into the normal-case function.

### 4.2 Identify the error kernel

**Fully applicable**, and this is precisely what an OTP **supervision tree** formalizes. The "error kernel" is the small, must-be-correct part of your system — typically your top-level `Application` and `Supervisor` modules, plus any process holding critical state. Everything below it (workers, transient processes) can crash and be restarted without compromising the kernel's integrity. Designing your supervision tree *is* identifying your error kernel.

---

## 5. Processes, Servers and Messages

### 5.1 Implement a process in one module

**Fully applicable — this is the `GenServer` pattern.** The code implementing one process's "loop" (in Elixir, its `GenServer`/`Agent`/`Task` callbacks) should live in a single module, and a single module should implement at most one *kind* of process.

```elixir
defmodule MyApp.Worker do
  use GenServer

  # client API
  def start_link(opts), do: GenServer.start_link(__MODULE__, opts, name: __MODULE__)

  # server callbacks
  @impl true
  def init(opts), do: {:ok, opts}

  @impl true
  def handle_call(:get_state, _from, state), do: {:reply, state, state}
end
```

### 5.2 Use processes for structuring the system

**Fully applicable.** Processes (and `GenServer`s, `Task`s, `Agent`s built on them) are the basic structuring unit — but don't reach for a process where a plain function call would do. Spawning a process has real cost and adds concurrency-related complexity that isn't always needed.

### 5.3 Registered processes

**Fully applicable.** Register a long-lived process under the same name as its module (`name: __MODULE__` is the idiomatic Elixir equivalent of Erlang's same-name registration), so it's obvious where to find the code. Don't register short-lived or dynamically-spawned processes — use a `Registry` or `DynamicSupervisor` for those instead.

### 5.4 Assign exactly one process to each true concurrent activity

**Fully applicable, unchanged.** Model the concurrency structure of the real-world problem 1:1 with Elixir processes — one process per connection, per device, per user session, etc. — and the program stays easy to reason about.

### 5.5 Each process should only have one "role"

**Fully applicable**, and OTP names these roles explicitly: `Supervisor`, `GenServer` (worker), and by convention a process that must never be allowed to crash silently (a "trusted worker"). Don't write a `GenServer` that's simultaneously acting as a client to itself or mixing supervisor-like restart logic into worker logic.

### 5.6 Use generic behaviours for servers and protocol handlers wherever possible

**Fully applicable — use OTP behaviours.** Prefer `GenServer`, `Supervisor`, `DynamicSupervisor`, `Task`, `:gen_statem` (via Erlang interop) over hand-rolled receive loops. Consistent use of the standard behaviours across your codebase does for Elixir exactly what Armstrong describes for Erlang's `gen_server`.

### 5.7 Tag messages

**Fully applicable.** Tag every message with an atom so pattern matching in `handle_info`/`handle_cast`/`receive` stays unambiguous as you add new message types.

Don't do this:

```elixir
def handle_info({mod, fun, args}, state) do  # Don't do this
  apply(mod, fun, args)
  {:noreply, state}
end
```

A later addition like `{:get_status_info, from, option}` risks colliding in the clause ordering. Instead:

```elixir
def handle_info({:execute, mod, fun, args}, state) do
  apply(mod, fun, args)
  {:noreply, state}
end

def handle_call({:get_status_info, option}, _from, state) do
  {:reply, get_status_info(option, state), state}
end
```

If a request is synchronous, tag the *reply* with a different atom than the request (e.g. request `:get_status_info` → reply `:status_info`); it makes debugging traces much easier to read. (In idiomatic `GenServer` code, `handle_call/3`'s return value plays this role for you automatically — the "reply tag" concern mostly applies when you're doing raw `send`/`receive` instead of `GenServer.call/2`.)

### 5.8 Flush unknown messages

**Fully applicable.** Always have a catch-all clause in `handle_info/2` (or in a raw `receive`), or unexpected messages will pile up unbounded in the process mailbox.

```elixir
def handle_info(other, state) do
  Logger.error("Process #{inspect(self())} got unknown message #{inspect(other)}")
  {:noreply, state}
end
```

### 5.9 Write tail-recursive servers

**Fully applicable to raw receive loops; largely handled for you by `GenServer`.** If you write a manual `receive`-based loop, every clause must end by tail-calling the loop function, or the process leaks memory over its lifetime.

```elixir
# Don't do this — not tail-recursive
def loop() do
  receive do
    {:msg1, msg1} ->
      handle(msg1)
      loop()

    :stop ->
      true
  end

  IO.puts("Server going down")  # runs after the receive returns — breaks tail call
end
```

```elixir
# Correct — tail-recursive
def loop() do
  receive do
    {:msg1, msg1} ->
      handle(msg1)
      loop()

    :stop ->
      IO.puts("Server going down")

    other ->
      Logger.error("unexpected message: #{inspect(other)}")
      loop()
  end
end
```

If you use `GenServer` (as you generally should — see 5.6), this class of bug basically disappears: the behaviour's own loop is already correctly tail-recursive, and your `handle_*` callbacks just return a value.

### 5.10 Interface functions

**Fully applicable — this is exactly the GenServer "client API" convention.** Never make callers `send/2` messages directly or reach into a process's mailbox protocol; wrap it in a plain function.

```elixir
defmodule MyApp.FileServer do
  use GenServer

  # client API — hides the messaging protocol
  def open_file(filename) do
    GenServer.call(__MODULE__, {:open_file_request, filename})
  end

  # server callback
  @impl true
  def handle_call({:open_file_request, filename}, _from, state) do
    {:reply, do_open(filename), state}
  end
end
```

The message protocol (`{:open_file_request, filename}`) is internal and should never leak to callers of `MyApp.FileServer`.

### 5.11 Time-outs

**Fully applicable.** Be careful with `after` timeout clauses in `receive` (or the `timeout` option to `GenServer.call/3`) — always consider what happens if the expected message arrives *after* the timeout has already fired (see 5.8: flush unknown/late messages instead of letting them sit in the mailbox forever).

### 5.12 Trapping exits

**Fully applicable.** As few processes as possible should call `Process.flag(:trap_exit, true)`. A process should consistently either trap exits or not — toggling it on and off during a process's lifetime is a recipe for confusing, hard-to-reproduce bugs.

---

## 6. Elixir-Specific Conventions

*(This section maps onto Armstrong's "Various Erlang Specific Conventions" — some rules translate almost unchanged, others are superseded by Elixir language features that didn't exist in 1990s Erlang.)*

### 6.1 Use structs (not raw records) as the principal data structure

**Adapted — Elixir replaces Erlang records with `defstruct`.** Erlang records are compile-time tuple sugar with no runtime identity; Elixir structs are proper tagged maps with a runtime `__struct__` field, which is strictly better for this purpose (you can `is_struct/2`, pattern-match on the module, and get compile-time field-name checking).

If a struct is shared across modules, define it in its "owning" module and reference it as `%MyApp.Person{}` — there's no need for a separate header-file mechanism (`.hrl`) since Elixir modules are the natural namespace.

```elixir
defmodule MyApp.Person do
  defstruct name: nil, age: nil, phone: [], info: %{}
end
```

### 6.2 Use pattern matching and struct/map access, not positional tuple matching

**Fully applicable**, and directly analogous to Armstrong's advice to use record selectors/constructors instead of raw tuple matching.

```elixir
def demo() do
  p = %MyApp.Person{name: "Joe", age: 29}
  %MyApp.Person{name: name1} = p   # pattern match, or...
  name2 = p.name                    # dot access
end
```

Don't do this:

```elixir
def demo() do
  p = %MyApp.Person{name: "Joe", age: 29}
  # Don't do this — bypasses the struct, brittle if fields change
  {MyApp.Person, name, _age, _phone, _info} = Map.from_struct(p) |> ...
end
```

### 6.3 Use tagged return values

**Fully applicable — this is core Elixir idiom.** Prefer `{:ok, value} | :error` or `{:ok, value} | {:error, reason}` over returning a bare value that might collide with a legitimate result.

Don't do this:

```elixir
def keysearch(_key, []), do: false
def keysearch(key, [{key, value} | _tail]), do: value  # untagged!
def keysearch(key, [_ | tail]), do: keysearch(key, tail)
```

This is ambiguous if a stored `value` happens to be `false`. Do this instead:

```elixir
def keysearch(_key, []), do: :error
def keysearch(key, [{key, value} | _tail]), do: {:ok, value}
def keysearch(key, [_ | tail]), do: keysearch(key, tail)
```

(In practice, reach for `List.keyfind/3`, `Keyword.fetch/2`, or `Map.fetch/2` — but the tagging principle behind them is the point.)

### 6.4 Use `try`/`rescue`/`throw`/`catch` with extreme care

**Fully applicable.** Don't reach for `try`/`rescue`, or especially `throw`/`catch`, unless you're sure you need them. They're legitimate for handling genuinely unreliable, deeply-nested external input (e.g., parsing untrusted data, a compiler front-end) — not as a substitute for tagged return values or normal control flow within your own trusted code.

### 6.5 Use the process dictionary with extreme care

**Fully applicable, unchanged.** `Process.put/2` and `Process.get/1` still exist in Elixir (they're thin wrappers over the same Erlang primitives) and the warning is identical: avoid them. A function using the process dictionary becomes non-deterministic with respect to its visible arguments — the same call can behave differently depending on hidden process state, which makes it harder to test, harder to reason about, and harder to debug (crash reports show the function's arguments, never the process dictionary).

Don't do this:

```elixir
def tokenize([]) do
  # Don't use Process.get/1 like this
  case get_characters_from_device(Process.get(:device)) do
    :eof -> []
    {:value, chars} -> tokenize(chars)
  end
end
```

Do this instead — pass it explicitly:

```elixir
def tokenize(_device, [h | t]), do: # ...

def tokenize(device, []) do
  case get_characters_from_device(device) do
    :eof -> []
    {:value, chars} -> tokenize(device, chars)
  end
end
```

### 6.6 Don't use bare `import` — prefer explicit module calls or `alias`

**Adapted.** Armstrong's `-import` doesn't exist in Elixir the same way, but the underlying concern (you can't tell at a glance which module a bare function call comes from) still applies directly to Elixir's `import`. Prefer calling functions with an explicit or aliased module prefix (`Enum.map/2`, or `alias MyApp.Foo` + `Foo.bar/1`) so the origin of every call is visible in the code. Reserve `import` for cases where it clearly improves readability (e.g., `import Ecto.Query` in a module that's overwhelmingly queries, or macros like `import ExUnit.Assertions` in tests) — and even then, scope it as tightly as possible.

### 6.7 Group and comment exported functions by purpose

**Adapted.** Elixir has no `-export` attribute — visibility is per-function (`def` vs `defp`) — but the underlying discipline still matters: make it clear *why* each public function exists. Use `@moduledoc`/`@doc` and grouping/comments to distinguish, for example:

```elixir
defmodule MyApp.Server do
  @moduledoc """
  ## Public API
  Functions intended for callers outside this module.

  ## Intermodule exports
  Functions intended to be called from sibling modules in this app.

  ## Internal (still public for testability, not part of the contract)
  """

  # --- user interface ---
  def help(), do: # ...
  def start(), do: # ...

  # --- intermodule exports ---
  def make_id(x), do: # ...

  # --- exported only for GenServer callback dispatch ---
  @doc false
  def init(arg), do: # ...
end
```

`@doc false` is the idiomatic Elixir way to mark a function as technically public (e.g., required by a behaviour) but not part of the intended API surface, keeping it out of generated docs.

---

## 7. Specific Lexical and Stylistic Conventions

### 7.1 Don't write deeply nested code

**Fully applicable.** Deeply nested `case`/`if`/`cond`/`receive` drifts code to the right and becomes unreadable. Limit nesting to roughly two levels by extracting private helper functions — or use `with` to flatten a chain of "happy path" steps instead of nesting `case`s.

```elixir
# Instead of nested case/case/case...
def process(input) do
  with {:ok, parsed} <- parse(input),
       {:ok, validated} <- validate(parsed),
       {:ok, result} <- run(validated) do
    {:ok, result}
  end
end
```

### 7.2 Don't write very large modules

**Fully applicable**, same guidance. Keep modules under roughly 400 lines; prefer several small, focused modules over one large one.

### 7.3 Don't write very long functions

**Fully applicable.** Keep functions to roughly 15–20 lines. In Elixir this often means favoring multiple function *clauses* (pattern-matched heads) over one long function body full of `case`/`cond`.

### 7.4 Don't write very long lines

**Fully applicable.** `mix format` enforces a default line length (98 characters) — run it, and treat its output as the house style. Multi-line strings and heredocs (`"""`) are the Elixir equivalent of Erlang's automatic string-literal concatenation.

### 7.5 Variable names

**Fully applicable, with Elixir naming conventions.** Elixir uses `snake_case` for variables (not `CamelCase`/underscored-capital as in Erlang). Choose meaningful names. For intentionally-unused variables, prefix with `_` (e.g., `_reason`) rather than a bare `_` if you might want the value later — makes it trivial to find and un-ignore.

### 7.6 Function names

**Fully applicable, with Elixir's own strong conventions layered on top.** The function name must match what it does and shouldn't surprise the reader. Elixir has specific suffix/prefix conventions worth following:

* `is_*` — only for **guard-safe** boolean checks, following the same convention Armstrong describes (`is_atom?` is *not* idiomatic; note there's no `?` on `is_*` guards by convention, e.g. `is_nil/1`).
* `*?` — for ordinary (non-guard) boolean-returning functions, e.g. `valid?/1`, `empty?/1`.
* `*!` — for functions that raise on failure instead of returning a tagged tuple, e.g. `File.read!/1` vs `File.read/1`.
* `to_*` — for pure conversions, e.g. `to_string/1`.

Same-purpose functions across different modules should share a name — this is exactly why `Enum.map/2`, `Stream.map/2`, and `Map.map` (via `for`) share the verb `map`.

### 7.7 Module names

**Fully applicable, and Elixir formalizes this with real nested namespacing.** Unlike Erlang's flat module space simulated with prefixes, Elixir module names are genuinely dotted/nested (`MyApp.Isdn.Init`, `MyApp.Isdn.PartB`), which is a direct, built-in solution to the exact problem Armstrong describes solving with naming conventions like `isdn_init`, `isdn_partb`.

### 7.8 Format programs in a consistent manner

**Superseded (in a good way) by `mix format`.** Elixir ships with an official formatter as part of `mix`. Run `mix format` and commit a `.formatter.exs`; this eliminates the entire class of "tabs vs spaces / comma spacing" debate Armstrong is describing, project-wide, automatically.

---

## 8. Documenting Code

### 8.1 Attribute code

**Adapted.** Elixir doesn't have Armstrong's custom `-revision`/`-created_by` module attributes; that role is filled by your source control system's commit history/blame (see 8.9/8.12) plus, where relevant, a `@moduledoc` note on provenance. Still: never take someone else's code without attribution.

### 8.2 Provide references in the code to the specifications

**Fully applicable, unchanged.** If code implements a documented protocol, spec, or RFC, cite the exact document and section/page in a comment or `@moduledoc`.

### 8.3 Document all the errors

**Fully applicable.** Use `Logger.error/2` (Elixir's standard replacement for `error_logger:error_msg/2`) at the point an error is detected, and keep a running document of all distinct error conditions your system can raise/log.

```elixir
Logger.error("failed to process order", order_id: order_id, reason: reason)
```

### 8.4 Document all the principal data structures in messages

**Fully applicable.** Use tagged tuples or structs as the canonical shape for inter-process messages, and document each one's fields and meaning — structs make this easier since `@type`/`@enforce_keys`/`@moduledoc` can carry the documentation right next to the definition.

### 8.5 Comments

**Adapted.** Elixir's comment conventions differ syntactically but the discipline is the same:

* `@moduledoc """ ... """` — module-level documentation (replaces `%%%`).
* `@doc """ ... """` — function-level documentation (replaces `%%`).
* `#` — inline comments within code (replaces `%`), placed above or beside the line it refers to.

Comments should be accurate, kept up to date, and written in clear English (or your project's chosen language) — never let them drift from the code.

### 8.6 Comment each function

**Fully applicable, with `@doc` and `@spec` as the concrete Elixir mechanism.** Document:

* The purpose of the function.
* Valid input domain (reinforced with `@spec`).
* Output domain / all possible return shapes (also reinforced with `@spec`).
* Any complex algorithm.
* Possible failure modes — what raises, what returns `{:error, _}`.
* Any side effects.

```elixir
@doc """
Get various information from a process.

## Arguments
  * `option` - `:normal` or `:all`
  * `pid` - the process to inspect

## Returns
A keyword list of `{key, value}` pairs, or `{:error, reason}` if the
process is dead.
"""
@spec get_server_statistics(:normal | :all, pid()) ::
        keyword() | {:error, term()}
def get_server_statistics(option, pid) when is_pid(pid) do
  # ...
end
```

`mix docs` (via ExDoc) turns these into real, browsable documentation — a significant upgrade over Armstrong's plain-text convention.

### 8.7 Data structures

**Fully applicable, via `@moduledoc`/`@typedoc` on a struct.**

```elixir
defmodule MyApp.Person do
  @moduledoc """
  ## Fields
    * `:name`  - a string (default `nil`)
    * `:age`   - an integer (default `nil`)
    * `:phone` - a list of integers (default `[]`)
    * `:info`  - a map of miscellaneous info about the person (default `%{}`)
  """
  defstruct name: nil, age: nil, phone: [], info: %{}
end
```

### 8.8 File headers, copyright

**Fully applicable, unchanged in spirit.** If your project requires a copyright header, put it at the top of the file as a `#` comment block, same as any other language. Many open-source Elixir projects instead keep licensing in a single `LICENSE` file at the repo root — either is fine; be consistent within a project.

### 8.9 File headers, revision history

**Superseded by source control.** Don't hand-maintain a revision-history comment block in each file — `git log`/`git blame` already give you exactly this information, always up to date, without the maintenance burden or risk of staleness. (This is arguably true in modern Erlang shops too; it just wasn't in 1990s practice.)

### 8.10 File header, description

**Fully applicable, via `@moduledoc`.**

```elixir
defmodule MyApp.FoobarDataManipulation do
  @moduledoc """
  Foobars are the basic elements in the Baz signalling system. Functions
  here manipulate foobar data.

  If you know of any weaknesses, bugs, or incomplete features, note them
  here rather than hiding them.
  """
end
```

### 8.11 Do not comment out old code — remove it

**Fully applicable, unchanged.** Your source control system (see 8.9/8.12) is exactly the mechanism for recovering deleted code if you ever need it again. Dead, commented-out code just adds noise.

### 8.12 Use a source code control system

**Fully applicable, unchanged** (git being the near-universal modern choice; RCS/CVS/Clearcase have simply been superseded).

---

## 9. The Most Common Mistakes

Directly applicable to Elixir, unchanged:

* Writing functions that span many pages (§7.3).
* Writing deeply nested `if`/`case`/`receive`/`cond` (§7.1).
* Writing badly "typed" functions — untagged return values (§6.3).
* Function names that don't reflect what the function does (§7.6).
* Meaningless variable names (§7.5).
* Using processes where a plain function call would do (§5.4).
* Badly chosen data structures.
* Missing or bad `@doc`/comments — always document arguments and return values.
* Unformatted code (run `mix format`).
* Using `Process.put/2`/`Process.get/1` (§6.5).
* No control of message queues / mailbox (§5.8, §5.11).

---

## 10. Required Documents

**Fully applicable, with modern Elixir tooling filling most of these roles automatically:**

### 10.1 Module Descriptions
Largely generated for you: `@moduledoc` + `@doc` + `@spec` on every public function, rendered into browsable docs by **ExDoc** (`mix docs`). This directly satisfies Armstrong's requirement for per-module, per-function documentation of arguments, return values, purpose, and failure modes.

### 10.2 Message Descriptions
No standard tool covers this automatically — document the shape of your inter-process messages (structs or tagged tuples) explicitly, e.g. in `@moduledoc` on the struct that represents the message (see 8.4/8.7), or in a dedicated protocol-design doc for cross-service messages.

### 10.3 Process
Document all named/registered processes (`GenServer`s registered via `name: __MODULE__`), their client-API interface, and purpose — plus how dynamically-spawned processes (via `DynamicSupervisor`/`Task.Supervisor`) are supervised and identified (e.g., via `Registry`).

### 10.4 Error Messages
Keep a living document (or a well-organized set of `@moduledoc`s / a wiki page) describing every distinct logged error condition in the system and what it means — same requirement as §8.3, kept centrally as well as inline.
