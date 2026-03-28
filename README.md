# Curity Identity Server – Plugin SDK Skill for Coding Agents

This repository contains a **skill** for coding agents (Claude Code, GitHub Copilot, Gemini, and others) that provides guidance for developing plugins for the **Curity Identity Server** using the publicly available SDK.

The skill gives agents:

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

**Option 1: Install as a plugin (Recommended)**

Install using the GitHub repository URL:

```bash
claude /plugin install https://github.com/curityio/coding-agent-instructions
```

Or clone locally and run with the plugin flag:

```bash
git clone https://github.com/curityio/coding-agent-instructions.git
claude --plugin-dir ./coding-agent-instructions
```

**Option 2: Install as a personal skill**

Clone the repository and symlink the skill folder:

```bash
git clone https://github.com/curityio/coding-agent-instructions.git ~/src/coding-agent-instructions
mkdir -p ~/.claude/skills
ln -s ~/src/coding-agent-instructions/skills/curity-plugin-sdk ~/.claude/skills/curity-plugin-sdk
```

Once installed, Claude Code will automatically use this skill when working on Curity plugin development tasks, or you can invoke it directly with `/curity-plugin-sdk`.

### GitHub Copilot

GitHub Copilot can use the skill documentation as custom instructions.

**Option 1: Copy as project instructions (Recommended)**

Clone the repo and copy `SKILL.md` plus the `agent-docs/` directory into your plugin project's `.github/` directory:

```bash
git clone https://github.com/curityio/coding-agent-instructions.git /tmp/coding-agent-instructions
mkdir -p .github/instructions
cp /tmp/coding-agent-instructions/skills/curity-plugin-sdk/SKILL.md .github/instructions/curity-plugin-sdk.md
cp -r /tmp/coding-agent-instructions/skills/curity-plugin-sdk/agent-docs .github/instructions/agent-docs
```

Copilot in VS Code and JetBrains IDEs automatically picks up files from `.github/instructions/`.

**Option 2: Use as a Git submodule**

Add as a submodule to your plugin project:

```bash
git submodule add https://github.com/curityio/coding-agent-instructions.git docs/curity-agent-docs
```

Then create a `.github/copilot-instructions.md` that points to it:

```markdown
See [Curity Plugin SDK Guide](../docs/curity-agent-docs/skills/curity-plugin-sdk/SKILL.md) for plugin development instructions.
```

**Option 3: VS Code global instructions**

1. Open VS Code Settings (`Cmd+,` or `Ctrl+,`)
2. Search for "Copilot Instructions"
3. Add the path to `skills/curity-plugin-sdk/SKILL.md` or paste its contents

### Other Agents

Point your agent at the `skills/curity-plugin-sdk/SKILL.md` file and the `skills/curity-plugin-sdk/agent-docs/` directory.

## Structure

```
coding-agent-instructions/
├── .claude-plugin/
│   └── plugin.json                      # Plugin manifest for Claude Code
├── skills/
│   └── curity-plugin-sdk/              # Main skill directory
│       ├── SKILL.md                     # Skill definition and agent instructions
│       └── agent-docs/                  # Topic-specific references
│           ├── plugin-system.md         # Plugin system concepts & lifecycle
│           ├── plugin-types.md          # Overview of available plugin types
│           ├── plugin-type-*.md         # Details for specific plugin types
│           ├── request-handlers.md      # Multi-step flow implementation
│           ├── sdk-services.md          # SDK services (credentials, SMS, etc.)
│           ├── configuration.md         # Configuration annotations & nesting
│           ├── templating.md            # Velocity templating system
│           ├── attributes.md            # Attributes and authentication results
│           ├── testing.md               # Unit tests with Spock Framework
│           ├── build-and-deployment.md  # Build configuration and deployment
│           ├── recipes.md               # End-to-end examples and patterns
│           └── quick-reference.md       # Task-oriented index
├── LICENSE
└── README.md
```

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

## License

Apache License 2.0 - See LICENSE file for details.
