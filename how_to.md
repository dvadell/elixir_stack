# Standard Elixir/Phoenix Project Setup

This document outlines the standard procedure for bootstrapping and configuring new Elixir/Phoenix web applications.

1. Ensure you have `asdf` installed along with the necessary plugins:
```bash
asdf plugin add nodejs
asdf plugin add erlang
asdf plugin add elixir
```

2. Create a .tool-versions file with the following content:
```
nodejs 22.20.0
elixir 1.20.2-otp-29
erlang 29.0.4
```

3. Install the specified versions:
```
asdf install
```

4. Choose a name for your project (e.g., cerca) and generate the Phoenix application:
```
# See `mix phx.new --help` for additional options (e.g., --no-html, --database)
mix archive.install hex phx_new
mix phx.new cerca
```

5. Navigate into the newly created project folder and copy the .tool-versions file into it to pin the versions for the project:
```
cd cerca
cp ../.tool-versions .
```

6. Open mix.exs and add the following standard modules to your deps function:
```
  defp deps do
    [
      # ... existing phoenix deps ...
      
      {:dialyxir, "~> 1.4", only: [:dev, :test], runtime: false},
      {:excellent_migrations, "~> 0.1", only: [:dev, :test], runtime: false},
      {:ex_slop, "~> 0.1", only: [:dev, :test], runtime: false},
      {:credo_contrib, "~> 0.2.0", only: [:dev, :test], runtime: false},
      {:extra_credo, github: "dvadell/extra_credo", only: [:dev, :test], runtime: false},
      {:jump_credo_checks, "~> 0.4", only: [:dev], runtime: false},
      {:oeditus_credo, "~> 0.8", only: [:dev, :test], runtime: false},
    ]
  end
```

7. Fetch the new dependencies:
```
mix deps.get
```

8. Create a .credo.exs file in the root of the project with the following custom configuration:
```
%{
  configs: [
    %{
      name: "default",
      files: %{
        included: ["lib/", "src/", "test/", "priv/repo/migrations"],
        excluded: [~r"/_build/", ~r"/deps/"]
      },
      plugins: [
        {CredoContrib, []},
        {ExSlop, []},
        {ExtraCredo, []}
      ],
      checks: %{
        enabled: [
          # Jump rules
          {Jump.CredoChecks.VacuousTest, []},
          {Jump.CredoChecks.TestHasNoAssertions, []},
          {Jump.CredoChecks.WeakAssertion, []},
          {Jump.CredoChecks.UnusedLiveViewAssign, [ignored_assigns: []]},
          {Jump.CredoChecks.AssertElementSelectorCanNeverFail, []},
          {Jump.CredoChecks.ConditionalAssertion, []},
          {Jump.CredoChecks.TooManyAssertions, [max_assertions: 20]},

          # Excellent_migrations
          # {ExcellentMigrations.CredoCheck.MigrationsSafety, []},

          # Oeditus
          # Error Handling
          {OeditusCredo.Check.Warning.SilentErrorCase, []},
          # Database & Performance
          {OeditusCredo.Check.Warning.InefficientFilter, []},
          {OeditusCredo.Check.Warning.NPlusOneQuery, []},
          {OeditusCredo.Check.Warning.MissingPreload, []},
          # LiveView & Concurrency
          {OeditusCredo.Check.Warning.UnmanagedTask, []},
          {OeditusCredo.Check.Warning.SyncOverAsync, []},
          {OeditusCredo.Check.Warning.MissingHandleAsync, []},
          {OeditusCredo.Check.Warning.MissingThrottle, []},
          {OeditusCredo.Check.Warning.InlineJavascript, []},
          # Readability
          {OeditusCredo.Check.Readability.UnnecessaryInterpolatingSigil, []},
          # Code Quality
          {OeditusCredo.Check.Warning.DirectStructUpdate, []},
          {OeditusCredo.Check.Warning.CallbackHell, [max_nesting: 2]},
          {OeditusCredo.Check.Warning.BlockingInPlug, []},
          {OeditusCredo.Check.Warning.UnsafeMapAccess, []},
          # Refactoring Suggestions
          {OeditusCredo.Check.Refactoring.SuggestFSM, []},
          # Telemetry & Observability
          {OeditusCredo.Check.Warning.TelemetryInRecursiveFunction, []},
          {OeditusCredo.Check.Warning.MissingTelemetryInAuthPlug, []},
          {OeditusCredo.Check.Warning.MissingTelemetryForExternalHttp, []},
          # Security - Injection
          {OeditusCredo.Check.Security.SQLInjection, []},
          {OeditusCredo.Check.Security.OSCommandInjection, []},
          {OeditusCredo.Check.Security.CodeInjection, []},
          {OeditusCredo.Check.Security.XSSVulnerability, []},
          # Security - Auth
          {OeditusCredo.Check.Security.MissingAuthentication, []},
          {OeditusCredo.Check.Security.MissingAuthorization, []},
          {OeditusCredo.Check.Security.IncorrectAuthorization, []},
          {OeditusCredo.Check.Security.InsecureDirectObjectReference, []},
          # Security - Data Protection
          {OeditusCredo.Check.Security.SensitiveDataExposure, []},
          {OeditusCredo.Check.Security.HardcodedCredentials, []},
          {OeditusCredo.Check.Security.UnsafeDeserialization, []},
          # Security - Input & File Handling
          {OeditusCredo.Check.Security.ImproperInputValidation, []},
          {OeditusCredo.Check.Security.PathTraversal, []},
          {OeditusCredo.Check.Security.UnrestrictedFileUpload, []},
          # Security - Web
          # {OeditusCredo.Check.Security.MissingCSRFProtection, []}, -- too many false positives
          {OeditusCredo.Check.Security.SSRFVulnerability, []},
          # Security - Race Conditions
          {OeditusCredo.Check.Security.TOCTOU, []}
        ]
      }
    }
  ]
}
```

9. To ensure code quality before committing, add a precommit alias to mix.exs.
```
  defp aliases do
    [
      setup: ["deps.get", "ecto.setup", "assets.setup", "assets.build"],
      # ... other aliases ...
      
      precommit: [
        "format",
        "credo --strict",
        "dialyzer",
        "test --cover"
      ]
    ]
  end
```

10. Run your full pre-commit check anytime by executing:
```
mix precommit
```

