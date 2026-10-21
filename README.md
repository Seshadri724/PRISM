# PRISM

Single-file visualizers for CI/CD pipelines. Run a simulated demo, or watch real GitHub Actions runs.

| File | What it is |
|---|---|
| `agent-factory.html` | **PRISM: Agent Factory** — a 3D factory showing AI coding agents' pull requests moving through CI. Simulated demo, or live GitHub data for any public repo. |
| `pipeline3d.html` | **PRISM: Pipeline Yard** — a 3D, game-like warehouse view of one pipeline, with insights, replay and AI help. Live GitHub data. |
| `pipeline.html` | A flat 2D graph with runner cards and a log |

## Run it

`pipeline.html` opens by double-clicking. The two 3D pages load Three.js as an ES module, so serve the folder (needs internet for the CDN):

```bash
python -m http.server 8765
```

Then open `http://localhost:8765/agent-factory.html` or `http://localhost:8765/pipeline3d.html`.

## PRISM: Agent Factory (3D)

Answers three questions about teams that use AI coding agents: **which agent's PRs break the build most, what each agent costs in CI minutes, and which PRs are waiting on a human review.**

It has two modes, always labelled in the top bar:

- **SIMULATED (default).** Made-up PRs and CI results. Each agent's quality is shuffled on every page load, so the demo never implies a real ranking of products.
- **LIVE · owner/repo.** Type a repo (or paste its GitHub URL) and press **Watch live**, or open `agent-factory.html?repo=owner/name`. Works for public repos with no setup; add a token for private repos or faster refresh. The link never contains the token.

### Live mode: how it reads GitHub

It reads the 30 newest PRs and the 100 newest workflow runs, then draws three stations: **CI → Review → Merged**. (GitHub's API doesn't give a stage graph, so the six simulated stations don't apply.)

| What you see | Where it comes from |
|---|---|
| Crate colour / agent | Guessed from the PR author (`copilot`, `devin`, `cursor`, `codex` bot accounts), branch prefix (`claude/`, `cursor/`, `codex/`, `copilot/`), or a "Generated with Claude Code" / `Co-Authored-By: Claude` line in the PR body. Other bot accounts go to **Bots**; everything else is **Humans**. |
| Running at CI / in the repair bay | The PR's latest workflow run: in progress, or failed (failure, timed out, startup failure) |
| Waiting for a human | CI is green and the PR is still open. GitHub's review state is **not** read, so "changes requested" isn't shown. |
| Merged | `merged_at` is set. Only the 12 newest merges are drawn. |
| Break % / Wasted | Failed runs ÷ all runs, and the time those failed runs took |
| Spend (est.) | Each run's wall-clock time × the $/runner-minute box. This is a **lower bound**: jobs that run in parallel are billed separately, and macOS/Windows cost more. |

**Rate limits:** without a token GitHub allows 60 requests an hour; the page makes 2 per refresh and refreshes every 150 s. With a token it refreshes every 15 s. The token is sent only to `api.github.com` and isn't stored. Anyone pasting a token into a page should use a fine-grained, read-only token.

**Not in live mode:** speed, reviewer and PR-volume controls (they only drive the simulator), and the AI triage note (the demo's is canned text; live mode links to the failed run instead).

| Factory | CI/CD |
|---|---|
| Robots on the agent dock | AI agents (Claude Code, Copilot Agent, Codex, Cursor, Devin) and Humans, each opening PRs |
| Crates | Pull requests, coloured by who opened them |
| Machines on the belt | CI stages: Lint → Test → Build → Security → Review → Deploy |
| Red repair bay | Failed PRs. Most are re-pushed and retried; after 3 tries they are abandoned. |
| Human review area | PRs waiting for a person to review, with desks for the reviewers |
| Smoking machine, "BOTTLENECK" label | The furthest-downstream stage with a backed-up queue |
| Shipped dock | PRs that reached production |

**Leaderboard** (per agent): PRs, break rate, CI spend, money wasted on runs that failed, PRs in review, PRs shipped. The top bar adds total spend, total waste, PRs in the pipeline, PRs awaiting a human, average review wait and sim time.

**Controls:** pause and 1×/2×/4× speed; add or remove reviewers (watch the review pile shrink); an agent PR volume slider; an editable $/runner-minute (default $0.008, GitHub's usual Linux price, so check current pricing); click an agent to spotlight its PRs; click a crate for its history, attempts, CI time, cost and a sample "AI triage" note for failures (canned text, not a real model call); 🎥 Cinematic hides the panels and slowly rotates the camera for screen recordings; ⟲ resets the camera. Drag to orbit, scroll to zoom.

**Adding another agent** is one line in the `AGENTS` list near the top of the script.

### Where PRISM: Agent Factory is going

Done: the view only reads crate objects, and two sources fill them (the simulator and the GitHub poller), so a new source is a small addition. Planned, not built:

1. **Host it** (GitHub Pages or similar) so `?repo=owner/name` is a shareable link. It opens fine today from any static host or local server.
2. **Better agent detection**: an optional `.agent-factory.yml` for custom author/branch mappings, and real billable minutes from GitHub's run timing data instead of wall-clock.
3. **Review state** (changes requested, reviewer names) from the reviews API.
4. **A GitHub Action** that builds the data inside the user's own CI and publishes the viewer to their GitHub Pages, so tokens never leave their account.
5. **`npx agent-factory owner/repo`** using the user's local `gh` login, for private repos.
6. **A GitHub App** for org-wide, real-time use. This is the natural paid tier.

Private repos should never need a token pasted into a website. Use the Action, the CLI or the App.

## PRISM: Pipeline Yard (3D)

| Warehouse | Pipeline |
|---|---|
| Bays on the floor | Stages, left to right (Trigger first) |
| Machines | Jobs. Ring and pad glow amber (running), green (passed), red (failed). The bar shows progress, the cubes show steps. |
| Crates on conveyors | Artifacts moving to the next stage while it runs |
| Forklifts | Runners. They drive to the job they pick up; the pool grows with parallel jobs. |
| Smoke and blinking light | A failed job. Downstream jobs go grey (skipped). |
| Truck leaving | The whole run passed |
| Gold pad | The job is on the **critical path** (shown once the run finishes) |

**Controls:** drag to orbit, right-drag to pan, scroll to zoom, click a machine for its steps. `T` toggles TV mode, `Esc` leaves it.

### Features

- **Exact job graph.** Live mode reads `needs:` from the run's workflow file, so stages and conveyors match the real dependency graph, including jobs that skip a stage. If the file can't be read it falls back to a timing guess.
- **Run insights.** Duration, billable minutes and estimated cost (GitHub-hosted Linux, $0.008/min, each job rounded up to a minute), critical path, the slowest job on it, jobs that wait on a slower sibling, runner queue wait, recent failure rate and retries (flaky hint).
- **Recent-runs bars.** Height is duration, colour is result. Click a bar to load that run.
- **Replay.** Every state change is recorded. Drag the slider or press ▶ to replay the run, including forklift movement. `LIVE` jumps back.
- **Ask Claude.** Click a failed machine → *Explain this failure*, or type a question. With an Anthropic API key it calls Claude (`claude-sonnet-5-5`) with the run state and log tail. Without a key it uses offline pattern hints. The key stays in memory and is sent only to `api.anthropic.com`.
- **Fleet.** Enter several repos separated by commas. Each gets a status chip, and clicking one switches the yard to it.
- **Alerts.** The 🔔 button turns on a chime or alarm and a desktop notification when a run finishes. Green-run streaks and fastest-run records show as toasts.
- **TV mode and share links.** 📺 hides the controls and slowly rotates the camera, for an office screen. 🔗 copies a link such as `?repo=owner/repo&tv=1&theme=dark` (a token is never put in the link).
- **Themes.** Follows the OS light/dark setting; 🌓 overrides and is remembered.

### URL parameters

| Param | Effect |
|---|---|
| `repo=a/b,c/d` | Start watching live instead of the demo |
| `tv=1` | Start in TV mode |
| `theme=light\|dark` | Force a theme |
| `selftest=1` | Run the built-in checks for the graph, layout and critical-path code; the result appears in the message line |

## Using it

- **Run demo** simulates a full run. Tick **make tests fail** to break Jest and try *Explain this failure*.
- **Watch live** takes `owner/repo` and an optional GitHub token. Without a token GitHub allows about 60 API requests per hour; a running pipeline polls every 10 seconds and a finished one every 30. The token is only sent to `api.github.com` and is never stored.

## Known limits

- **Live logs show job and step changes only.** Real log text needs an authenticated log download, which browsers can't do directly. Failed jobs include their GitHub annotations instead.
- **Runner assignment is simulated.** The page hands running jobs to forklifts itself and doesn't read real runner names.
- **Cost is an estimate** for GitHub-hosted Linux runners; self-hosted, larger or macOS/Windows runners differ.
- **The AI key lives in the browser.** Fine for personal use. A shared deployment should proxy the call through a server.
- **Matrix jobs and reusable workflows** are matched by name; unusual names can fall back to the timing layout.

## Roadmap (needs a server and credentials)

These can't be done in a static page:

1. **GitHub App with webhooks** — instant updates, no polling or rate limits, one-click install.
2. **Real log streaming** through a small backend that downloads logs with the user's token.
3. **Slack and Teams alerts** with a link into the yard.
4. **Auto-fix pull requests** — Claude proposes the patch for a failing test, lockfile or config.
5. **Multi-org dashboard** with history, flaky-test tracking over time, SSO and a self-hosted option.
6. **More CI systems** — GitLab, Jenkins, CircleCI, Buildkite, Argo.

## Tech

Plain HTML and JS. PRISM: Pipeline Yard uses [Three.js](https://threejs.org) 0.160 and js-yaml 4.1; PRISM: Agent Factory uses Three.js 0.170 with its bloom post-processing, plus Google Fonts. All come from CDNs (jsDelivr, Google Fonts). No build step and nothing to install.
