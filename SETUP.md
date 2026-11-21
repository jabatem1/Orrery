# Setup — Mac, start to finish (~10 minutes)

You need: this folder's files, Terminal (Cmd+Space → "Terminal"), and your Claude **Pro/Max** account
(Claude Code usage is included with those plans; free accounts can't authenticate it).

## 1. Install Claude Code

The native installer is the recommended path (auto-updates, no Node.js required):

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Then **open a new Terminal window** (so PATH updates apply) and verify:

```bash
claude --version
```

If `claude` isn't found, add the install dir to your shell and retry:
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Alternatives that also work: `brew install --cask claude-code`, or the legacy npm route
(`npm install -g @anthropic-ai/claude-code`, requires Node 18+). Docs:
https://docs.claude.com/en/docs/claude-code/overview

## 2. Create the project folder and drop these files in

```bash
mkdir -p ~/Projects/orrery && cd ~/Projects/orrery
unzip ~/Downloads/orrery-claude-code-transfer.zip -d ~/Projects/orrery
ls   # should show: CLAUDE.md  PROMPTS.md  SETUP.md  README.md  night-sky-3d.html
```

(If the unzip created a nested folder, `mv` its contents up so `CLAUDE.md` sits at the project root —
Claude Code auto-reads `CLAUDE.md` from the root of wherever you launch it.)

Initialize git — Claude Code works dramatically better with checkpoints it can diff and revert:

```bash
git init && git add -A && git commit -m "Baseline: single-file app + project brief"
```

## 3. First launch

```bash
cd ~/Projects/orrery
claude
```

First run opens your browser to sign in — choose your **Claude account** (Pro/Max) OAuth, not an API
key. Back in Terminal you'll get the `>` prompt. Claude Code has already read `CLAUDE.md`, so it
knows the whole project: the engine, the hard-won bugs, the design system, the roadmap.

## 4. Run the migration

Open `PROMPTS.md`, copy **Prompt 1** verbatim, paste it at the `>` prompt. Expect ~15–30 minutes of
supervised work. Claude Code will ask permission before running commands and editing files — read the
asks; "yes, and don't ask again for this session" is fine once you trust the pattern.

Essential driving controls:
- **Shift+Tab** → cycles into **plan mode**: Claude proposes before touching files. Use it for every
  roadmap feature.
- **Esc** interrupts mid-task; **Esc Esc** jumps back to edit an earlier message.
- `/clear` wipes the conversation between unrelated tasks (keeps CLAUDE.md knowledge — it's re-read
  every session). `/help` lists the rest.
- Ask it to commit at every green checkpoint: "commit this" — cheap insurance.

## 5. The deploy loop (keeps your existing link alive)

After migration, every release is:

```bash
npm run build        # emits dist/night-sky-3d.html — one self-contained file
```

Then upload `dist/night-sky-3d.html` to your GitHub repo exactly as you do today
(Add file → Upload files → same filename → Commit). Same filename ⇒ same URL ⇒ your shared link and
the phone compass keep working. Nothing about hosting changes.

Optional upgrade once you're comfortable: tell Claude Code *"set up a GitHub Action that builds and
publishes dist/night-sky-3d.html to Pages on every push"* and the manual upload step disappears too.

## 6. Troubleshooting quick hits

- **"command not found: claude"** → new Terminal window; then the PATH line in step 1.
- **Login loop / wrong account** → `/logout` inside Claude Code, then `claude` again.
- **It edited something you didn't want** → `git checkout -- <file>` or ask it to revert; this is why
  step 2's git init matters.
- **Gatekeeper complaints running the installer** → System Settings → Privacy & Security → Allow;
  or use the Homebrew route instead.
