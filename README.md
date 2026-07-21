# elixir_stack - Notes and packages I use

# Credo modules
* https://github.com/elixir-vibe/ex_slop
* 
```
      {:ex_slop, "~> 0.1", only: [:dev, :test], runtime: false},
      {:credo_contrib, "~> 0.2.0", only: [:dev, :test], runtime: false},
      {:extra_credo, github: "dvadell/extra_credo", only: [:dev, :test], runtime: false}
```

In `.credo.exs`:
```
%{
  configs: [
    %{
      name: "default",
      plugins: [
        {CredoContrib, []},
        {ExSlop, []},
        {ExtraCredo, []}
      ]
    }
  ]
}
```

# Tracing
https://github.com/nietaki/rexbug or https://github.com/redink/extrace


