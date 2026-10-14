# Command reference

This reference covers the Fugu one-line install, the flags for the installer and the `codex-fugu` launcher, and how your Codex config is backed up, restored, and protected.

## The one-line install

The one-line install runs a small bootstrap script served at `https://sakana.ai/fugu/install`. The script clones this repository into `~/.fugu`, then asks what to set up, **Codex** (`codex-fugu`), **Claude Code** (`claude-fugu`), or **both**, and runs the matching installer: `~/.fugu/scripts/install.sh` (pins Codex, deploys the Fugu config, stores your API key) and/or `~/.fugu/scripts/install-claude.sh` (installs the Claude launcher and its settings file, reusing the same key). Choose non-interactively with a `--codex`/`--claude`/`--both` flag or `FUGU_INSTALL_TARGET`; any other flag after `bash` is forwarded to the selected installer, so every installer flag below also works through the one-line command.

```bash
curl -fsSL https://sakana.ai/fugu/install | bash
```

The bootstrap reads a few environment variables of its own. The installers' own variables (`SAKANA_API_KEY`, `CODEX_HOME`, `CODEX_INSTALL_DIR`, and the rest) pass straight through.

| Variable | Default | Purpose |
| --- | --- | --- |
| `FUGU_REPO_URL` | `https://github.com/SakanaAI/fugu.git` | clone source for the repo |
| `FUGU_HOME` | `~/.fugu` | directory the repo is cloned into |
| `FUGU_INSTALL_TARGET` | interactive menu (`codex` when headless) | which agent to install: `codex`, `claude`, or `both` (also `1`/`2`/`3`) |

Common forms:

| Situation | Command |
| --- | --- |
| Standard install (menu) | `curl -fsSL https://sakana.ai/fugu/install \| bash` |
| Claude Code only | `curl -fsSL https://sakana.ai/fugu/install \| bash -s -- --claude` |
| Both | `curl -fsSL https://sakana.ai/fugu/install \| bash -s -- --both` |
| Pass installer flags | `curl -fsSL https://sakana.ai/fugu/install \| bash -s -- --yes` |
| Non-interactive or CI | `curl -fsSL https://sakana.ai/fugu/install \| SAKANA_API_KEY=your_key FUGU_INSTALL_TARGET=both bash -s -- --yes` |
| Custom clone source | `curl -fsSL https://sakana.ai/fugu/install \| FUGU_REPO_URL=<url-or-path> bash` |

`bash -s --` passes the arguments that follow to the script. The bootstrap consumes `--codex`/`--claude`/`--both` to choose the target; every other flag (like `--yes`) goes to the selected installer. Place any environment variable right before `bash` so it reaches the bootstrap rather than `curl`. Re-running the command reuses an existing `~/.fugu`, and ongoing updates are handled by `codex-fugu`, so a re-run is rarely needed.

**If your first install failed partway through**, re-running the one-liner will not help on its own: it reuses the `~/.fugu` it already cloned, so it runs the same copy of the installer again. Remove the clone and start clean:

```bash
rm -rf ~/.fugu && curl -fsSL https://sakana.ai/fugu/install | bash
```

To run the same steps without the hosted endpoint (cloning the repository yourself and seeing exactly what runs), clone it once and run the installer for what you want:

```bash
git clone https://github.com/SakanaAI/fugu.git ~/.fugu
bash ~/.fugu/scripts/install.sh          # Codex (codex-fugu)
bash ~/.fugu/scripts/install-claude.sh   # Claude Code (claude-fugu)
```

To set up both, run `install.sh` first; it stores the API key that `install-claude.sh` then reuses.

## Installer flags

Two installers ship in the repo: `install.sh` for Codex and `install-claude.sh` for Claude Code. Run either with no flag to install.

### Codex (`install.sh`)

`bash ~/.fugu/scripts/install.sh [flag]`. Run with no flag to install and deploy.

| Flag | What it does |
| --- | --- |
| (none) | Install and pin the Codex CLI, then deploy the Fugu config |
| `--set-key` | Re-prompt for and store the Sakana API key, no redeploy |
| `--remove-config` | Cleanly undo the deployed config |
| `--pinned-version X.Y.Z` | Pin a specific Codex version instead of the default |
| `--force` | Deploy even if the installed Codex version does not match the target |
| `--dry-run` | Show what would happen and change nothing |
| `-y`, `--yes` | Assume yes, for non-interactive use |
| `-h`, `--help` | Full list of flags and environment variables |

Non-interactive install (for CI or provisioning):

```bash
SAKANA_API_KEY=your_key bash ~/.fugu/scripts/install.sh --yes
```

### Claude Code (`install-claude.sh`)

`bash ~/.fugu/scripts/install-claude.sh [flag]`. This installs the `claude-fugu` launcher, stores a Sakana API key, and writes the launcher's settings file; it does not install Codex or deploy any Codex config. With no flag it configures the key (reusing an existing one from `~/.claude/.fugu-env`, or from `~/.codex/.env` if you already set up Codex, otherwise prompting), installs the launcher to `~/.local/bin/claude-fugu`, then writes its settings file `~/.claude/.fugu-settings.json`.

| Flag | What it does |
| --- | --- |
| (none) | Configure the Sakana API key, install the `claude-fugu` launcher, then write its settings file |
| `--set-key` | Reconfigure the stored Sakana API key only; does not install the launcher |
| `--reconfigure` | During install, overwrite an existing stored key instead of keeping it |
| `--remove` | Remove the installed `claude-fugu` launcher (the stored key and the settings file are kept) |
| `--dry-run` | Show what would happen and change nothing |
| `-y`, `--yes` | Assume yes, for non-interactive use (a key must already be available) |
| `-h`, `--help` | Full list of flags and environment variables |

It has no `--remove-config`, `--pinned-version`, `--force`, or `--no-backup`: it deploys no config bundle, never installs or version-pins Codex, and always backs up a settings file that differs from the defaults before resetting it.

## Launcher flags

> **The flags below are `codex-fugu`-only.** The `claude-fugu` launcher has one flag of its own, recognized only as the first argument: `claude-fugu --refresh-settings` resets its settings file `~/.claude/.fugu-settings.json` to the defaults (backing up your copy first if it differs) and exits without starting Claude Code. Otherwise every argument is forwarded straight to Claude Code, after that settings file (`claude --settings ~/.claude/.fugu-settings.json "$@"`). To reconfigure the Claude key or reinstall its launcher, use `install-claude.sh` (`--set-key`, `--reconfigure`, `--remove`).

`codex-fugu` runs `codex -p fugu` and, at most once an hour, checks this repo for config updates and offers to apply them. It never blocks launch, and any arguments you pass go straight to Codex.

| Flag | What it does |
| --- | --- |
| `--status` | Show the installed version, the pinned target, and update state |
| `--set-key` | Rotate the stored Sakana API key |
| `--check` | Check for a config update now instead of waiting for the hourly check |
| `--recheck` | Re-enable update prompts you previously dismissed, then check |
| `--no-update` | Skip the update check for this launch |

Set `CODEX_FUGU_NO_UPDATE=1` to turn update checks off for good.

### Launch notices

Now and then the launcher shows a short notice before it starts Codex, with two choices: "acknowledge and continue" (the default) and "acknowledge and never remind me again". Choosing the second one hides that notice for good. There is at most one notice at a time. Set `CODEX_FUGU_NO_NOTICE=1` to turn notices off entirely.

### Passing Codex arguments

The launcher flags above are read only when they come first. From the first other argument onward, everything is forwarded unchanged to `codex -p fugu`, so any Codex command, subcommand, prompt, or flag generally works through `codex-fugu`. The commands below are just examples:

```bash
codex-fugu resume                  # codex -p fugu resume
codex-fugu "fix the failing test"  # start a new session with a prompt
codex-fugu --no-update resume      # skip the update check, then resume
```

Because forwarding starts at the first non-launcher argument, put any launcher flag before the Codex arguments. Plain Codex flags such as `--model` or `--help` pass straight through, since the launcher only reacts to its own flags listed above.

## Version management and session resume

### Codex (`codex-fugu`)

`codex-fugu` and the installer also manage your Codex version. The Fugu configs are verified against a specific Codex version, so on a mismatch the installer offers to switch your Codex binary to that version, and the launcher offers the same reconcile at most once an hour. A switch happens only with your consent, either an interactive yes or `--force`.

Codex keeps a per-version session index, so `codex resume` lists different past sessions after a version switch. Your session transcripts under `~/.codex/sessions` are never deleted by a switch. Only which sessions `codex resume` enumerates changes.

Before any switch the installer saves your current session index (the `state`, `memories`, and `goals` `.sqlite` files) into the backup described below. To bring back your earlier `codex resume` list you can either run the Codex version that wrote those sessions, or restore the saved index from a backup:

```bash
cp -p ~/.codex-backups/codex-config-<timestamp>/*.sqlite* ~/.codex/
```

### Claude Code (`claude-fugu`)

`claude-fugu` does not manage the Claude Code version or touch session state. It checks that `claude` is on your `PATH` (and exits with an install hint if it is not), creates its settings file `~/.claude/.fugu-settings.json` if it is missing (otherwise it leaves your edits alone and only updates the lines it owns, such as the API key when your key changes), then execs Claude Code with that file. It never pins or checks the Claude Code version, runs no update check, and keeps no session index; Claude Code manages its own `/model` selection and session resume, so there is no version-switch resume caveat like Codex's.

Because the launcher does not auto-update, you pick up new Fugu support for Claude Code (such as new Fugu models) by re-running the installer, which reinstalls the launcher and resets its settings file to the new defaults (backing up your copy first if it differs):

```bash
bash ~/.fugu/scripts/install-claude.sh
```

## Config backup, restore, and protection

### Codex (`install.sh`)

Before switching the Codex version or making its first edit to `config.toml`, the installer saves a timestamped copy of your existing config to `~/.codex-backups/codex-config-<timestamp>/`. This location sits outside `~/.codex`, so a backup survives even a full `rm -rf ~/.codex`. Each backup holds your `config.toml`, any `*.config.toml`, `auth.json`, other catalog `*.json`, and `*.md` files, the session index (`state`, `memories`, and `goals` `.sqlite` files), plus a `MANIFEST.txt` and a `SHA256SUMS` for verification. The 10 most recent backups are kept. Use `CODEX_BACKUP_KEEP` and `CODEX_BACKUP_ROOT` to change the count and location, or `--no-backup` to skip the step.

To restore a backup, copy it back over your config directory and re-check it:

```bash
rsync -a --exclude MANIFEST.txt --exclude SHA256SUMS ~/.codex-backups/codex-config-<timestamp>/ ~/.codex/
codex doctor   # expect: config.toml parse: ok
```

Your provider settings go into `config.toml` inside managed `# >>> fugu:... >>>` markers, so a re-deploy replaces only that block and leaves the rest of your config untouched. After each edit the installer re-parses the file with `codex doctor`, and if it no longer parses, the change is rolled back automatically. The stored `auth.json` is kept at mode `0600` so your credentials stay private.

### Claude Code (`claude-fugu`)

`install-claude.sh` deploys no config bundle and never edits `~/.claude/settings.json` or any other Claude Code config except the launcher's own settings file. Claude Code is pointed at Fugu through the `ANTHROPIC_*` environment variables the launcher sets at launch and that settings file, which the launcher passes with `--settings`, so your own Claude Code settings are never modified. Under `~/.claude` the installer writes two files of its own, both mode `0600`: the API-key store `~/.claude/.fugu-env` (in a `0700` directory) and the settings file `~/.claude/.fugu-settings.json`, which holds the Fugu endpoint, model list, default model and effort level, and a copy of your API key. When the key store already exists, only the `SAKANA_API_KEY` line is rewritten and any other lines are preserved. If no key is stored yet, an existing Codex key in `~/.codex/.env` is reused, read-only, and that file is never modified.

The settings file is yours to edit after install: a normal launch leaves your changes in place and only rewrites the lines the launcher owns, such as the API key if your key has changed. Re-running the installer, or running `claude-fugu --refresh-settings`, resets it to the launcher's defaults; if your copy differs, it is first saved as `~/.claude/.fugu-settings.json.bak.<timestamp>.<random>`, and copying that file back over `~/.claude/.fugu-settings.json` restores it. That backup is automatic, so there is no `--no-backup` flag. `install-claude.sh --remove` deletes only the launcher and keeps the key store and the settings file. Both of these files, and any settings backups, contain your API key, so to remove the key too, run `rm -f ~/.claude/.fugu-env ~/.claude/.fugu-settings.json*`.
