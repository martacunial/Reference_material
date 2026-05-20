# Click CLI Reference Guide

Personal backbone for building Python CLI tools with Click.

**What Click does:** turns regular Python functions into command-line commands by decorating them. It handles argument parsing, help text generation, type validation, error messages, and interactive prompts — so you write zero argparse boilerplate.

**How it works mentally:** every `@click.command()` function maps to one CLI invocation. `@click.group()` creates a parent namespace (like `git`) that holds sub-commands (like `git commit`, `git push`). Decorators are applied **bottom-up** (closest to the function first).

---

## Install

```bash
pip install click
# or with uv:
uv add click
```

```python
import click
```

---

## Basic command

```python
@click.command()
@click.option("--name", default="world", help="Who to greet.")
@click.option("--count", type=int, default=1, help="Number of times.")
def greet(name, count):
    """Greet someone a number of times."""  # becomes the --help description
    for _ in range(count):
        click.echo(f"Hello, {name}!")

if __name__ == "__main__":
    greet()
```

```bash
python app.py --name Marta --count 3
python app.py --help
```

> **Note:** `click.echo()` is preferred over `print()` — it handles encoding correctly on Windows and can be redirected/captured in tests.

---

## Option types

```python
@click.option("--lr",       type=float,              default=0.001)
@click.option("--epochs",   type=int,                default=50)
@click.option("--name",     type=str,                default="run")
@click.option("--mode",     type=click.Choice(["train", "eval", "both"]), default="train")
@click.option("--input",    type=click.Path(exists=True))          # validates path exists
@click.option("--output",   type=click.Path(file_okay=False))      # must be a directory
@click.option("--config",   type=click.File("r"))                  # opens the file for you
@click.option("--verbose",  is_flag=True, default=False)           # boolean flag
@click.option("--level",    type=click.IntRange(1, 10), default=5) # integer with bounds
```

`click.Path` vs `click.File`:
- `click.Path` — validates the path string, returns a `str`. Use when you open the file yourself later.
- `click.File` — validates **and opens** the file, returns a file object. Handles `-` as stdin/stdout automatically.

---

## Required options & arguments

```python
# Option: named, order-independent (--flag value)
@click.option("--model-key", required=True, help="Model architecture key.")

# Argument: positional, order-dependent
@click.argument("input_file", type=click.Path(exists=True))
@click.argument("output_dir", type=click.Path())
```

**Option vs Argument rule of thumb:** use arguments for the core subject of the command (the file to process); use options for everything else. Arguments cannot have help text in `--help` — prefer options for discoverability.

---

## Multiple values

```python
# Accept the same option multiple times: --tag foo --tag bar
@click.option("--tag", multiple=True)
def run(tag):
    print(tag)  # tuple: ("foo", "bar")

# Accept a fixed number of values in one option: --size 640 480
@click.option("--size", nargs=2, type=int)
def render(size):
    w, h = size
```

---

## Prompting for input

```python
# Prompt if option not provided on command line
@click.option("--name", prompt="Your name", help="Your name.")

# Prompt with confirmation (useful for passwords)
@click.option("--password", prompt=True, hide_input=True, confirmation_prompt=True)

# Manual prompt anywhere in a command body
value = click.prompt("Enter threshold", type=float, default=0.5)

# Yes/no confirmation
if click.confirm("This will overwrite existing results. Continue?"):
    proceed()
```

---

## Command groups (sub-commands)

Groups create a `git`-style CLI where the top-level name dispatches to sub-commands.

```python
@click.group()
def cli():
    """Anomaly detection pipeline."""
    pass

@cli.command()
@click.option("--config", type=click.Path(exists=True), default="config/config.yaml")
def train(config):
    """Train a new model."""
    click.echo(f"Training with config: {config}")

@cli.command()
@click.option("--run-id", required=True)
def predict(run_id):
    """Run prediction with an existing model."""
    click.echo(f"Predicting with run: {run_id}")

if __name__ == "__main__":
    cli()
```

```bash
python app.py train --config config/config.yaml
python app.py predict --run-id abc123
python app.py --help        # shows group help + list of commands
python app.py train --help  # shows sub-command help
```

---

## Passing context between commands (`pass_context`)

`Context` carries state between a group and its sub-commands without global variables.

```python
@click.group()
@click.option("--verbose", is_flag=True)
@click.pass_context               # injects ctx as first argument
def cli(ctx, verbose):
    ctx.ensure_object(dict)       # initialise ctx.obj if not set
    ctx.obj["verbose"] = verbose

@cli.command()
@click.pass_context
def run(ctx):
    if ctx.obj["verbose"]:
        click.echo("Verbose mode on")
```

Alternatively use `@click.pass_obj` to receive only `ctx.obj` directly (skips the full context object).

---

## Invoke a command programmatically

Useful when one command needs to call another, or in tests.

```python
@cli.command()
@click.pass_context
def all(ctx):
    """Run train then predict."""
    ctx.invoke(train, config="config/config.yaml")
    ctx.invoke(predict, run_id="latest")
```

---

## `invoke_without_command`

Makes the group body run even when no sub-command is given — useful for printing status or a default action.

```python
@click.group(invoke_without_command=True)
@click.pass_context
def cli(ctx):
    if ctx.invoked_subcommand is None:
        click.echo("No command given. Run `app --help` for usage.")
```

---

## Callbacks & eager options

Callbacks run when an option is parsed. `is_eager=True` makes the option run before all others — used for `--version` and `--help` style flags that should exit immediately.

```python
def print_version(ctx, param, value):
    if not value or ctx.resilient_parsing:
        return
    click.echo("Version 1.0.0")
    ctx.exit()

@click.option("--version", is_flag=True, is_eager=True,
              expose_value=False, callback=print_version,
              help="Show version and exit.")
```

---

## Error handling

```python
# Raise a user-facing error (shows nicely, exits with code 2)
raise click.BadParameter("Must be between 0 and 100.", param_hint="'--percentile'")

# Generic usage error
raise click.UsageError("Config file not found.")

# Exit with a specific code
raise click.ClickException("Something went wrong.")  # exit code 1, message printed

# Abort silently (e.g. user said "no" to a confirm prompt)
raise click.Abort()
```

---

## Styling output

```python
# Colours (stripped automatically if output is not a TTY / piped)
click.echo(click.style("ERROR", fg="red", bold=True) + ": file not found")
click.echo(click.style("OK", fg="green") + " Training complete.")

# Shorthand
click.secho("WARNING: low memory", fg="yellow")

# Pager (for long output — opens less/more automatically)
with click.echo_via_pager(long_text):
    pass  # or: click.echo_via_pager(long_text)

# Progress bar
with click.progressbar(items, label="Processing") as bar:
    for item in bar:
        process(item)
```

---

## Entry point in pyproject.toml

Wire a Click function as a console script so users run it by name:

```toml
[project.scripts]
anomaly-detection = "danieli_anomaly_detection.cli:cli"
```

After `uv sync` (or `pip install -e .`):

```bash
anomaly-detection train --config config/config.yaml
anomaly-detection --help
```

The string format is `"package.module:function"`.

---

## Testing with `CliRunner`

`CliRunner` invokes commands in-process without spawning a subprocess, capturing stdout/stderr as strings.

```python
from click.testing import CliRunner
from myapp.cli import cli

def test_train_command():
    runner = CliRunner()
    result = runner.invoke(cli, ["train", "--config", "config/config.yaml"])

    assert result.exit_code == 0
    assert "Training complete" in result.output

def test_missing_config():
    runner = CliRunner()
    result = runner.invoke(cli, ["train", "--config", "nonexistent.yaml"])
    assert result.exit_code != 0

# Test with a temporary filesystem
def test_with_files():
    runner = CliRunner()
    with runner.isolated_filesystem():
        with open("config.yaml", "w") as f:
            f.write("model:\n  key: ae\n")
        result = runner.invoke(cli, ["train", "--config", "config.yaml"])
        assert result.exit_code == 0
```

---

## Key notes

- Decorators apply **bottom-up**: the decorator closest to `def` is applied first.
- `click.echo()` always flushes; `print()` may buffer — use `echo` in CLI tools.
- `click.Path(exists=True)` raises a user-friendly error automatically if the path is missing; no need for manual `os.path.exists()` checks.
- `multiple=True` options return a `tuple`, not a list — use `list(tag)` if you need list methods.
- `--help` is added automatically to every command and group; you don't need to define it.
- Option names with hyphens (`--model-key`) become Python identifiers with underscores (`model_key`).
- `ctx.ensure_object(dict)` is idempotent — safe to call in both group and sub-commands.
- Always test with `CliRunner` rather than `subprocess` — it's faster and gives you full stack traces on failures.
