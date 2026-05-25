# AGENT_INSTALL.md — agent-driven install of `rare-disease-consult`

> **Note on naming:** this file is distinct from `AGENTS.md` (the open standard for always-on, every-session project context). `AGENT_INSTALL.md` is a one-time install runbook — fetched once during install, then discarded. Not always-on context. The `AGENT_` prefix signals it's agent-readable; the file lives separately from any `AGENTS.md` the repo may also carry.

You are helping a non-technical user install this Claude Code plugin. They are running Claude Code in their own working directory — likely a folder with medical records or symptom notes. **Your job: get the plugin installed and verified without making the user run shell commands themselves.**

## Surface the disclaimer FIRST

Before you do anything else, tell the user this — verbatim:

> This plugin is research assistance. It can help with one step in a long medical journey — giving you something concrete to bring to a doctor. It is **not a diagnosis**, **not medical advice**, and **not a substitute for a physician**.
>
> Specifically: it's often wrong — its top candidate is right roughly 6 of 10 times. About 1 in 4 cases, the correct answer isn't on the top-5 list at all. And because it's purpose-built for *rare* disease, it can confidently surface rare-disease lists for symptoms actually caused by something common (depression, thyroid, anemia, sleep apnea, medication side effects). Your doctor will rightly consider those first.
>
> Use this for an **informed conversation with your doctor**, not as a replacement for one.
>
> Ready to continue?

Wait for the user to acknowledge before installing.

## Verify prerequisites

Then check three things, in order:

### 1. Claude Code
The user is talking to you, so this is satisfied. Skip.

### 2. `uv` (Python package manager, for the HPO MCP server)

```bash
which uv
```

If missing:
- **macOS / Linux:** offer to run `curl -LsSf https://astral.sh/uv/install.sh | sh` (a single-binary install, no admin needed). Confirm with the user before running.
- **Windows:** tell the user to install from https://docs.astral.sh/uv/ — don't try to auto-install.

### 3. `node` + `npx` (for the PubMed MCP server)

```bash
which node && which npx
```

If missing:
- **macOS with Homebrew:** offer `brew install node`. Confirm first.
- **Linux:** ask the user how they'd like to install Node (system package manager, nvm, or direct download). Don't auto-install — Node is a larger install and the user should know it's happening.
- **Windows:** point at https://nodejs.org/ for the installer.

If the user is on Windows or otherwise can't install one of these, **stop and explain that the plugin won't work without both** — don't proceed to install a half-working plugin.

## Hand off the install to the user

You are **not** editing `.claude/settings.local.json`, `.claude/settings.json`, or any file under `~/.claude/` on the user's behalf. Plugin install is intentionally a user-confirmed action in Claude Code — the user clicking through the `/plugin` UI is the supported path, and trying to short-circuit it via direct settings writes both trips the auto-mode safety classifier *and* leaves the install half-registered. Your role stops at telling the user what to do.

Tell the user, verbatim or close to it:

> Prerequisites are good. The rest of the install is in Claude Code's `/plugin` UI — I'll walk you through it but you'll be the one clicking. Plugin code runs on your data, so the install is intentionally yours to confirm.
>
> **Step 1 — open the plugin UI.** In this Claude Code session, run:
>
> ```
> /plugin
> ```
>
> A UI opens with tabs across the top: `Plugins | Discover | Installed | Marketplaces | Errors`. Use the keys shown in the footer: **Enter** to select, **Esc** to go back, **u** to update, **d** to remove.
>
> **Step 2 — add the marketplace.** Switch to the **`Marketplaces`** tab. At the top of the list there's a row labeled **`+ Add Marketplace`** — select it and paste this as the source when prompted:
>
> ```
> fryanpan/rare-disease-consult-plugin
> ```
>
> Confirm. Claude Code fetches the marketplace definition from GitHub.
>
> **Step 3 — install the plugin.** Switch to the **`Plugins`** tab. Search for `rare-disease-consult`, select it, and on the install screen pick **local scope** — the option labeled "Install for you, in this repo only". That scopes the plugin to this working directory only, so it won't be active in your other Claude Code sessions on unrelated work. (Avoid "user scope" — that turns the plugin on everywhere — and avoid "project scope" — that's for sharing with collaborators on a shared git repo.) When the install succeeds you'll see this in the transcript:
>
> ```
> ✓ Installed rare-disease-consult. Run /reload-plugins to apply.
> ```
>
> **Step 4 — reload.** Run `/reload-plugins`. The plugin's MCP servers (HPO + PubMed) boot on reload — no full Claude Code restart needed. If `/reload-plugins` isn't available in your version, exit and start a new Claude Code session from this same directory instead.
>
> Let me know when you've done all four and I'll verify.

Wait for the user's confirmation. Then verify:

```bash
claude plugin list 2>&1 | grep rare-disease
```

The plugin should appear with status enabled. The marketplace name in the listing is auto-assigned by Claude Code from the source repo (typically `rare-disease-consult-plugin` or a similar derived name); whatever appears is fine.

If the plugin doesn't appear:
- Ask what the user saw in the `/plugin` UI — was there an error message? Did the marketplace-add step complete? Did the install step show the green checkmark?
- Common case: the marketplace gets added but the install step needs a second click. Have the user run `/plugin`, switch to the `Plugins` tab or the marketplace listing, find `rare-disease-consult`, and confirm it shows as installed/enabled before retrying the reload.

Once `claude plugin list` shows it enabled, tell the user:

> Installation complete. You can invoke the consultation with `/rare-disease-consult:consult`.

## First-run note

The first time `/rare-disease-consult:consult` runs, the HPO MCP server downloads ~50 MB of HPO + Orphanet annotation data via `pyhpo`. This takes ~30 seconds and only happens once. Tell the user about this delay so they don't think it's stuck.

## Troubleshooting

### `pubmed_*` tools don't appear after install

If `/mcp` or `claude plugin list` shows the plugin loaded with HPO tools (`mcp__plugin_rare-disease-consult_hpo__*`) but no PubMed tools (`mcp__plugin_rare-disease-consult_pubmed__*`), the PubMed MCP server likely failed its stdio handshake. The most common cause is the server logging to stdout during startup, which corrupts the JSON-RPC stream.

The shipped `.mcp.json` sets `MCP_LOG_LEVEL=silent` (and two other pino-style log env vars) to suppress this, and pins to a known-good version of `@cyanheads/pubmed-mcp-server`. If those defaults aren't enough on a particular machine, try:

1. **Warm the npm cache:** run `npx -y @cyanheads/pubmed-mcp-server@2.7.4 </dev/null` once at a shell. It may take 30-60s on a fresh install. Then exit Claude Code and restart it — the handshake should succeed when the server doesn't have to also pay the install cost during MCP init.
2. **Confirm the env vars stuck:** if you have `MCP_LOG_LEVEL` set globally to something verbose in your shell profile, it can override the plugin's `.mcp.json` setting. Check with `env | grep -i log`.
3. **Last resort, vendor it:** clone the PubMed MCP server locally and point `.mcp.json` at the local path. Avoids the npx-resolution variability entirely. Out of scope for default install.

If PubMed tools still don't appear, the consult skill's Phase 3 will detect the gap on its tool-availability probe and surface it to the user before producing a differential — the consultation still runs (WebSearch fills in), but the user knows the gap.

## Reading list before they start a real case

Point the user at the README for the full caveat list before they run a real consultation:

> Before you start, please read the **"Risks worth taking seriously"** section at https://github.com/fryanpan/rare-disease-consult-plugin — it covers anchoring bias, common-cause blind spot, workup-cascade harm, cyberchondria, hallucinated citations, and privacy. ~5 minutes; worth it.

## Cowork users

If the user is on **Claude Cowork** (the hosted variant, not local Claude Code): stop and explain that this plugin uses local-stdio MCP servers (`uv` and `npx`) which Cowork's hosted runtime doesn't currently support. Cowork support is planned for a future release. They'd need to install local Claude Code to use this plugin today.

## Done criteria

You've finished when ALL of these are true:

- [ ] The user has acknowledged the disclaimer
- [ ] `uv` is installed and on PATH
- [ ] `node` and `npx` are installed and on PATH
- [ ] The user added `fryanpan/rare-disease-consult-plugin` as a marketplace via the `/plugin` UI (Marketplaces tab → `+ Add Marketplace`)
- [ ] The user installed `rare-disease-consult` from that marketplace via the `/plugin` UI (transcript shows `✓ Installed rare-disease-consult. Run /reload-plugins to apply.`)
- [ ] The user ran `/reload-plugins` (or restarted Claude Code) and `claude plugin list` shows `rare-disease-consult` as enabled
- [ ] The user has been pointed at the README's "Risks worth taking seriously" section
