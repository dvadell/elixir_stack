# elixir_stack - Notes and packages I use
* See also https://mikezornek.com/posts/2026/7/guarding-against-ai-drift/?utm_source=reddit&utm_campaign=guarding-against-ai-drift

# Credo modules
* https://github.com/elixir-vibe/ex_slop
* https://github.com/Jump-App/credo_checks
* https://github.com/Oeditus/oeditus_credo
* https://github.com/Artur-Sulej/excellent_migrations
 
```
      {:excellent_migrations, "~> 0.1", only: [:dev, :test], runtime: false},
      {:ex_slop, "~> 0.1", only: [:dev, :test], runtime: false},
      {:credo_contrib, "~> 0.2.0", only: [:dev, :test], runtime: false},
      {:extra_credo, github: "dvadell/extra_credo", only: [:dev, :test], runtime: false},
      {:jump_credo_checks, "~> 0.4", only: [:dev], runtime: false},
      {:oeditus_credo, "~> 0.1.0", only: [:dev, :test], runtime: false},
```

In `.credo.exs`:
```
%{
  configs: [
    %{
      name: "default",
      files: %{
        "priv/repo/migrations",
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
  ]
}
```

NOTE: If a warning does not apply to your specific case, you can suppress it with # credo:disable-for-next-line 

# TODO: check OeditusCredo.Check.Refactoring.ChangeRiskAntiPatterns from https://github.com/Oeditus/oeditus_credo
# Tracing
https://github.com/nietaki/rexbug or https://github.com/redink/extrace


