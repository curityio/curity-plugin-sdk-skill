
# Curity Identity Server – Coding Agent Instructions for Plugin Development

This repository contains **machine-focused documentation** intended for use by coding agents
such as GitHub Copilot, Gemini, and other AI assistants when developing plugins for the
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

## Structure

- `INSTRUCTIONS.md`  
  Main behavior and high-level rules for coding agents working on Curity plugins.

- `.github/copilot-instructions.md`  
  Short version for GitHub Copilot; loaded with high priority by Copilot in supported IDEs.

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

## How to Use in Other Repositories

When creating a new Curity plugin project:

1. Clone this repository
2. Create a new plugin using a prompt, or clone an existing plugin to a subfolder to this repo. These can not be committed.
3. If you create a new plugin, initialize a git repository in the created subfolder
