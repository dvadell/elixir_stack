Search the BranchedLLM codebase for defensive programming patterns where the shape of the data is already known but the code uses type guards, Map.get/3 on structs, overly broad rescues, or nil-coalescing on non-nil values. These are cases where the code "doesn't trust itself" — it checks for possibilities that can't actually happen given the data flow.

 Specifically, find and fix these categories:

 1. `Map.get(struct, :field, default)` on known structs — When the variable is a defined struct (e.g., %ReqLLM.StreamChunk{}, %BranchedLLM.Message{}), use dot access
       (chunk.type, call.metadata) instead of Map.get. The struct fields always exist; the default value is dead code. Exception: Map.get on nested dynamic maps inside struct fields
       (like chunk.metadata[:tool_call_args]) is fine — those keys come from LLM providers and may be absent.

 2. `if is_map(x) / is_binary(x) / is_nil(x)` type guards on values with known types — If a function spec, pattern match, or struct definition already constrains the type,
       remove the runtime check. Examples: if is_map(chunk) when chunks are always strings from StreamResponse.tokens/1, or if message != nil when the spec is String.t().

 3. Overly broad `rescue` — rescue e -> {:error, e} catches ArgumentError, MatchError, etc. — real bugs that should crash during development. Either remove the rescue (if the
       code is pure and can't reasonably fail) or narrow it to specific exception types (e.g., rescue e in [ReqLLM.Error]).

 4. Silent no-ops on configuration errors — if is_nil(repo), do: :error in ToolCache.Ecto silently swallows the fact that the repo isn't configured. If someone chose the Ecto
       cache, a missing repo is a bug — use Keyword.fetch!(:repo) to fail fast.

 5. `to_string(x)` on values that are already strings — If the type spec or upstream contract guarantees String.t(), remove the redundant conversion. If the contract is
       ambiguous (e.g., execute_tool returns term() but the cache expects String.t()), fix the spec instead.

Do NOT change: Map.get on dynamic maps from external sources (LLM API responses, provider metadata), || nil-coalescing when the value can genuinely be nil (e.g., call.name on
partial tool calls where the name arrives via metadata), or try/rescue around actual I/O/network calls.
After fixing, run mix precommit and address any warnings or failures.