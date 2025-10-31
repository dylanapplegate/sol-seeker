# Gemini Workspace Context

## Project Overview

This is a Python project for a local-first, multi-agent AI system for automating job searches. We use `uv` for all dependency management and project execution.

## Required Libraries

The following Python libraries are required for this project:

- `pyautogen`
- `chromadb`
- `duckduckgo-search`
- `requests`
- `beautifulsoup4`

## Directives for Python Scripts

When generating or working with Python scripts, follow these rules:

1.  **Always use `uv`**: Use `uv run` to execute scripts or manage environments. Avoid using `pip`, `venv`, or `python` directly unless absolutely necessary for system-level checks.
2.  **Inline Dependencies**: For single-file scripts, use `uv`'s inline metadata format to specify dependencies within a `# /// script` block.
3.  **Example Script Format**:

    ```python
    # /// script
    # requires-python = ">=3.12"
    # dependencies = [
    #   "requests",
    #   "rich",
    # ]
    # ///
    import requests
    from rich.console import Console

    console = Console()
    console.print(f"Using requests version: {requests.__version__}")
    ```

4.  **Project Environment**: For multi-file projects, assume a project structure managed by `pyproject.toml` and use `uv run` or `uv sync` to manage the project environment. The environment will be located in a `.venv` directory.
5.  **Tool Usage**: If a task requires a specific Python library, suggest writing a script using the format above and executing it with `uv run`.

## Available Tools

The system has the `uv` command-line tool installed and available in the PATH. You can use the following `uv` commands:

- `uv run <script_name.py>`: Executes a Python script with its inline dependencies.
- `uv init`: Initializes a new project with a `pyproject.toml` file.
- `uv add <package>`: Adds a package to the project's `pyproject.toml` dependencies.
- `uv sync`: Creates or updates the project's virtual environment based on `pyproject.toml`.
- `uv pip install <package>`: Installs a package directly into the active environment (use sparingly, prefer `uv add`).

Ensure all generated code and instructions align with the best practices of the `uv` tool.
