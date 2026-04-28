# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`shcmanager` is a Bash-based environment configuration manager. It presents an interactive TUI menu where the user selects plugins to install or uninstall. It has no build step, no tests, and no external dependencies beyond Bash and the tools each plugin uses (e.g. `brew`, `git`).

## Running the script

```bash
./shcmanager                    # open the interactive menu
./shcmanager create <name>      # scaffold a new plugin template in WORK_DIR/plugins/
```

The script must always be run in Bash. It re-execs itself under Bash when launched from zsh.

## Configuration

`.shcmanager.config` in the repo root sets `WORK_DIR`, the directory where managed config files and plugins live. The script prompts the user to create this file if it is missing.

```
WORK_DIR=~/shcmanager/managed
```

`WORK_DIR/plugins/` is where plugins are loaded from at runtime. `WORK_DIR` itself is typically a separate git repository for cross-machine config tracking.

## Architecture

### Plugin system

Each file in `WORK_DIR/plugins/` is sourced by the main script. A plugin must define two functions named after the file:

```bash
install_<plugin_name>()   { ... }
uninstall_<plugin_name>() { ... }
```

The main script collects plugin names, renders the menu, then calls the appropriate function for each selected item in the order they appear in the menu.

### Adapters (`adapters/`)

Adapters are sourced inside plugins via `$SHCMANAGER_ADAPTERS_DIR`, which the main script exports before loading plugins.

| Adapter | Functions |
|---|---|
| `symlink` | `install_symlink file target`, `uninstall_symlink file target`, `install_symlinks "file\|target;..."`, `uninstall_symlinks "..."` |
| `brew_package` | `install_brew_package pkg [tap]`, `uninstall_brew_package pkg [tap]`, `install_brew_packages "pkg\|tap;..."`, `uninstall_brew_packages "..."` |
| `repository` | `install_repository src path`, `uninstall_repository src path`, `install_repositories "src\|path;..."`, `uninstall_repositories "..."` |

Batch functions (`install_brew_packages`, etc.) take a semicolon-delimited string where each item's fields are separated by `|`. Plugins typically build an array, then join it with `IFS=';'` to produce this format.

### Plugin authoring pattern

```bash
source "$SHCMANAGER_ADAPTERS_DIR/brew_package"
source "$SHCMANAGER_ADAPTERS_DIR/symlink"

BREW=("tmux" "gitmux|arl/arl")
CSV_BREW=$(IFS=';'; echo "${BREW[*]}")

install_myplugin() {
  install_brew_packages "${CSV_BREW[*]}"
  install_symlink "$HOME/managed/.myconf" "$HOME/.myconf"
}

uninstall_myplugin() {
  uninstall_brew_packages "${CSV_BREW[*]}"
  uninstall_symlink "$HOME/managed/.myconf" "$HOME/.myconf"
}
```

### Menu navigation

`k` up · `l` down · `i` mark for install (green) · `u` mark for uninstall (red) · `Enter` confirm · `q` quit

Items selected for install run first (in menu order), then items selected for uninstall.
