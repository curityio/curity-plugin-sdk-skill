
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
  - `plugin-type-*.md` – Details and skeletons for specific plugin types
  - `ui-and-screens.md` – How to show screens and handle user interaction
  - `sdk-services.md` – How to interact with SDK services
  - `templating.md` – How the templating system works
  - `configuration.md` – How plugins define and use configuration
  - `recipes.md` – Common patterns and example flows

## How to Use in Other Repositories

When creating a new Curity plugin project:

1. Copy `INSTRUCTIONS.md` and the `agent-docs/` folder into the root of the plugin repo.  
2. Optionally copy `.github/copilot-instructions.md` into `.github/` in that repo.  
3. Use your IDE’s coding agent (Copilot, Gemini, etc.) as usual. It will automatically
   read and use these documents as context when generating code.

This creates a standard “knowledge pack” that improves the quality and consistency
of generated plugins across projects.
