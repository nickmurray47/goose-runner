# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **Goose-based automation system** for overnight task execution. Goose is a CLI tool that executes recipes (YAML files) to automate development workflows.

The system follows this workflow:
1. Daily todos are created in `todos/` as README files (format: `todo_DD_MM_YY.README`)
2. The `generate-from-todo.yaml` recipe transforms a todo README into an executable Goose recipe
3. The generated recipe is saved to `recipes/overnight.yaml` (overwriting previous versions)
4. The `run-overnight.yaml` runner executes the generated overnight recipe at midnight

## Architecture

### Directory Structure

- **`todos/`**: Contains daily todo files that define tasks to be automated
- **`recipes/`**: Contains Goose recipe YAML files
  - `generate-from-todo.yaml`: Recipe generator that converts todos into executable recipes
  - `overnight.yaml`: The generated recipe (created daily, gitignored typically)
- **`runner/`**: Contains the orchestration recipe
  - `run-overnight.yaml`: Main runner that executes the overnight recipe

### Recipe System

All recipes follow Goose YAML format with these key sections:
- `version`: Recipe format version (currently "1.0.0")
- `title` & `description`: Metadata
- `prompt`: Natural language prompt for the Goose agent (**REQUIRED** - always use this)
- `parameters`: Input parameters with validation
- `extensions`: Tools/MCP servers available to the recipe (e.g., developer, filesystem)
- `sub_recipes`: References to other recipes
- `response`: Expected structured output (JSON schema)
- `activities`: High-level task descriptions

**CRITICAL:** Always use `prompt` field. Never use `instructions` field - it will silently fail in this system.

### Key Recipes

**generate-from-todo.yaml**: Reads a todo file and generates a concrete Goose recipe
- Parameters: `todo_file` (path to todo), `output_recipe` (default: `./recipes/overnight.yaml`)
- Uses filesystem MCP server via `npx @modelcontextprotocol/server-filesystem`
- Returns JSON: `{saved: boolean, path: string}`

**run-overnight.yaml**: Executes the generated overnight recipe
- References `overnight` subrecipe at `../recipes/overnight.yaml`
- Exits with actionable error if subrecipe is missing

## Common Commands

### Running Recipes

```bash
# Generate overnight recipe from today's todo
goose run recipes/generate-from-todo.yaml --parameter todo_file=./todos/todo_27_10_25.README

# Execute the overnight runner
goose run runner/run-overnight.yaml
```

### Creating Todo Files

Todo files should be named `todo_DD_MM_YY.README` and contain numbered task lists:
```
1. Task description here
2. Another task description
```

## Development Workflow

1. Create a new todo file in `todos/` with today's date
2. Run `generate-from-todo.yaml` to create the overnight recipe
3. The generated `recipes/overnight.yaml` will contain executable instructions
4. `run-overnight.yaml` executes the generated recipe (scheduled for midnight)

## Important Notes

- The `overnight.yaml` file is generated daily and overwritten each time
- Recipes must include either `instructions` or `prompt` (or both)
- When tasks require specific tools, add them to the recipe's `extensions` section
- Generated recipes should prefer explicit commands over vague instructions
- File paths in generated recipes should be project-relative

## Claude Code Provider on macOS

When using the Claude Code provider (`GOOSE_PROVIDER=claude-code`), there are important permission considerations for macOS:

### File Write Permissions

The Claude Code provider delegates file operations to the `claude` CLI tool, which has its own permission system. To enable file write operations:

**Set Goose Mode to "auto":**
```bash
# Via config file
echo "GOOSE_MODE: auto" >> ~/.config/goose/config.yaml

# Or via environment variable
export GOOSE_MODE=auto

# Or via goose configure command
goose configure
# Select: Goose Settings → Goose Mode → Auto Mode
```

Without `GOOSE_MODE=auto`, the claude CLI runs in restrictive mode and will require manual approval for all file writes, causing permission errors on macOS.

### Structured Output Limitation

**Important:** Recipes with `response` fields (structured JSON output) are **not compatible** with the Claude Code provider because:
- Claude Code filters out Goose's tool ecosystem (including the `final_output` tool)
- This causes the agent to never call the required tool, resulting in repeated "You MUST call the `final_output` tool NOW" messages

**Solution:** Remove the `response` section from recipes when using Claude Code, or switch to the direct Anthropic provider (`GOOSE_PROVIDER=anthropic`) if structured output is required.
