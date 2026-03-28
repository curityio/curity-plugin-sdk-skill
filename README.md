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

GitHub Copilot automatically loads instructions from `.github/copilot-instructions.md` in your repository.

**Option 1: Add to your plugin project (Recommended)**

Copy the instructions file to your Curity plugin project:

```bash
# In your plugin project directory
mkdir -p .github
curl -o .github/copilot-instructions.md https://raw.githubusercontent.com/curityio/coding-agent-instructions/main/.github/copilot-instructions.md
```

**Option 2: Install as a skill in ~/.copilot/skills**

Clone the repository and symlink the skill folder:

```bash
git clone https://github.com/curityio/coding-agent-instructions.git ~/src/coding-agent-instructions
mkdir -p ~/.copilot/skills
ln -s ~/src/coding-agent-instructions/skills/curity-plugin-sdk ~/.copilot/skills/curity-plugin-sdk
```

**Option 3: Use as a Git submodule**

Add as a submodule to your plugin project:

```bash
git submodule add https://github.com/curityio/coding-agent-instructions.git docs/curity-agent-docs
```

Then reference the instructions in your own `.github/copilot-instructions.md`:

```markdown
See [Curity Plugin SDK Guide](../docs/curity-agent-docs/skills/curity-plugin-sdk/INSTRUCTIONS.md) for detailed instructions.
```

**VS Code Settings (Optional)**

You can also configure Copilot to use custom instructions globally in VS Code:

1. Open VS Code Settings (`Cmd+,` or `Ctrl+,`)
2. Search for "Copilot Instructions"
3. Add the path to `skills/curity-plugin-sdk/INSTRUCTIONS.md` or paste its contents

The `.github/copilot-instructions.md` file is automatically loaded with high priority by Copilot in supported IDEs (VS Code, JetBrains).

### Other Agents

Reference the `skills/curity-plugin-sdk/INSTRUCTIONS.md` file or `skills/curity-plugin-sdk/agent-docs/` directory in your agent's configuration.

## Structure

```
coding-agent-instructions/
├── .claude-plugin/
│   └── plugin.json                      # Plugin manifest for Claude Code
├── .github/
│   └── copilot-instructions.md          # Short version for GitHub Copilot
├── skills/
│   └── curity-plugin-sdk/              # Main skill directory
│       ├── SKILL.md                     # Skill definition with frontmatter
│       ├── INSTRUCTIONS.md              # Detailed coding guidelines
│       └── agent-docs/                  # Topic-specific references
│           ├── plugin-system.md         # Plugin system concepts & lifecycle
│           ├── plugin-types.md          # Overview of available plugin types
│           ├── plugin-type-*.md         # Details for specific plugin types
│           ├── request-handlers.md      # Multi-step flow implementation
│           ├── sdk-services.md          # SDK services (credentials, SMS, etc.)
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
