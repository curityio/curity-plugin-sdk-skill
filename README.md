# Curity Identity Server – Coding Agent Instructions for Plugin Development

This repository contains **machine-focused documentation** intended for use by coding agents
such as Claude Code, GitHub Copilot, Gemini, and other AI assistants when developing plugins for the
**Curity Identity Server**.

The goal is to give agents:

- A consistent mental model of how the **plugin system** works
- Clear rules for how to implement **different plugin types**
- Guidance for how to **show UI/screens** to end-users
- Rules for how to **interact with SDK services**
- An understanding of how the **templating system** works
- Example **recipes** and patterns they can reuse

> These documents are primarily written for **AI coding agents**, not humans, but humans are
> welcome to read them as well.

## Installation

### Claude Code

Install as a plugin using the GitHub repository URL:

```bash
claude /plugin install https://github.com/YOUR_USERNAME/curity-plugin-development
```

Or clone locally and install from the directory:

```bash
git clone https://github.com/YOUR_USERNAME/curity-plugin-development.git
claude --plugin-dir ./curity-plugin-development
```

Once installed, Claude Code will automatically use this skill when working on Curity plugin development tasks.

### GitHub Copilot

GitHub Copilot automatically loads instructions from `.github/copilot-instructions.md` in your repository.

**Option 1: Add to your plugin project (Recommended)**

Copy the instructions file to your Curity plugin project:

```bash
# In your plugin project directory
mkdir -p .github
curl -o .github/copilot-instructions.md https://raw.githubusercontent.com/YOUR_USERNAME/curity-plugin-development/main/.github/copilot-instructions.md
```

**Option 2: Clone and work inside this repository**

Clone this repository and create your plugin as a subdirectory:

```bash
git clone https://github.com/YOUR_USERNAME/curity-plugin-development.git
cd curity-plugin-development
# Create your plugin in a subdirectory (add to .gitignore)
```

**Option 3: Use as a Git submodule**

Add as a submodule to your plugin project:

```bash
git submodule add https://github.com/YOUR_USERNAME/curity-plugin-development.git docs/curity-agent-docs
```

Then reference the instructions in your own `.github/copilot-instructions.md`:

```markdown
See [Curity Plugin Development Guide](../docs/curity-agent-docs/INSTRUCTIONS.md) for detailed instructions.
```

**VS Code Settings (Optional)**

You can also configure Copilot to use custom instructions globally in VS Code:

1. Open VS Code Settings (`Cmd+,` or `Ctrl+,`)
2. Search for "Copilot Instructions"
3. Add the path to `INSTRUCTIONS.md` or paste its contents

The `.github/copilot-instructions.md` file is automatically loaded with high priority by Copilot in supported IDEs (VS Code, JetBrains).

### Other Agents

Reference the `INSTRUCTIONS.md` file or `agent-docs/` directory in your agent's configuration.

## Structure

- `INSTRUCTIONS.md`
  Main behavior and high-level rules for coding agents working on Curity plugins.

- `.github/copilot-instructions.md`
  Short version for GitHub Copilot; loaded with high priority by Copilot in supported IDEs.

- `.claude-plugin/plugin.json`
  Plugin manifest for Claude Code installation.

- `skills/curity-plugin-development/SKILL.md`
  Skill definition for Claude Code with frontmatter and instructions.

- `agent-docs/`
  Topic-specific references and examples:
  - `plugin-system.md` – How the plugin system works (concepts + lifecycle)
  - `plugin-types.md` – Overview of available plugin types
  - `plugin-type-*.md` – Details and skeletons for specific plugin types (e.g., authenticator)
  - `request-handlers.md` – How to implement request handlers for multi-step flows
  - `sdk-services.md` – How to interact with SDK services (credentials, accounts, SMS, etc.)
  - `templating.md` – How the Velocity templating system works
  - `attributes.md` – Working with attributes and authentication results
  - `testing.md` – How to write unit tests with Spock Framework
  - `build-and-deployment.md` – Build configuration and deployment steps
  - `recipes.md` – Complete end-to-end examples and common patterns
  - `quick-reference.md` – Task-oriented index for fast lookup

## Usage

Once installed, the agent will automatically use this skill when:

- Creating new Curity plugins
- Implementing authenticators or authentication actions
- Working with Curity SDK APIs
- Writing Spock tests for plugins
- Configuring Gradle builds for plugins

### Example Prompts

**Create a new authenticator:**
```
Create a backchannel authenticator for Vipps CIBA integration
```

**Implement request handler:**
```
Add a multi-screen OTP authenticator with SMS verification
```

**Add tests:**
```
Create Spock tests for the authentication handler
```

**Configure build:**
```
Add Gradle tasks for deploying to local Curity server
```

## How to Use in Other Repositories

When creating a new Curity plugin project:

1. Clone this repository
2. Create a new plugin using a prompt, or clone an existing plugin to a subfolder to this repo. These can not be committed.
3. If you create a new plugin, initialize a git repository in the created subfolder

## License

Apache License 2.0 - See LICENSE file for details.
