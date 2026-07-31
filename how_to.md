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

6. Create the database
```
mix ecto.create
```

7. Open mix.exs and add the following standard modules to your deps function:
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

8. Fetch the new dependencies:
```
mix deps.get
```

9. Create a .credo.exs file in the root of the project with the one from https://github.com/dvadell/elixir_stack/edit/main/.credo.exs

10. To ensure code quality before committing, add a precommit alias to mix.exs.
```
  defp aliases do
    [
      setup: ["deps.get", "ecto.setup", "assets.setup", "assets.build"],
      # ... other aliases ...
      
      precommit: [
        "format",
        "credo --strict",
        "dialyzer",
        ~S(cmd sh -c "MIX_ENV=test mix test --cover")
      ]
    ]
  end
```

11. Run your full pre-commit check anytime by executing:
```
mix precommit
```

