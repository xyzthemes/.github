# xyzthemes: org conventions for theme agents

This org builds storefront themes for more than one platform (Salla now, Shopify next). Every agent working
here reads this first. It is public, so it holds no secrets.

## 1. Know which platform a repo belongs to
- **Topics are the source of truth:** `salla-theme`, `shopify-theme`, `theme-template`, `salla-hq` / `shopify-hq`,
  `theme-assets`. Filter with `gh repo list xyzthemes --topic <topic>`.
- **Names:** Salla repos have no prefix (they predate this note): `theme-base`, `anbar-perfume-theme`,
  `nada-skincare-theme`, `libas-fashion-theme`, `qahwa-coffee-theme`, `barq-electronics-theme`,
  `raqmi-digital-theme`. New Shopify repos use `shopify-<name>`. New Salla repos may use `salla-<name>`.
- **Salla platform link:** Salla's GitHub app reads only repos containing `twilight.json`; Shopify reads only its own
  theme structure. Never add one platform's manifest to the other's repo.
- **Don't touch the other platform's repos.** That covers code, branches, PRs, Actions and settings. Salla repos
  are owned by the Salla factory, whose records are in `salla-themes-hq`. Shopify repos are owned by the Shopify
  factory, whose records are in its own HQ repo.
- `maan-center` is not a theme repo. Leave it alone.

## 2. Templates and records stay separate
- `theme-base` is the **Salla** template (Twilight React engine). The Shopify factory uses its own template
  (e.g. `shopify-theme-base`).
- Each platform keeps its own HQ repo for plans, ledgers, decisions and briefs. Don't write into the other
  platform's HQ.

## 3. Shared repos: one section or folder per platform
- `theme-screenshots` (public): Salla shots live at `<theme>/<version>/` (existing). Shopify shots go under
  `shopify/<theme>/<version>/`. Push on an `assets/<theme>` branch; direct pushes to `main` are blocked by the
  agents' guardrail.
- `.github` (this repo, public): the org profile (`profile/README.md`) gets one section per platform, and each
  factory edits only its own section. Change it by PR.

## 4. The build server is shared and small (4 CPUs, 8 GB RAM)
- **At most 3–4 heavy agents in total**, across both factories combined. On 2026-10-16, 8 parallel agents
  drove the load to about 80, and swapping froze every agent for about 11 hours.
- **Serialize heavy one-shot commands** behind the shared lock, for example
  `flock /var/lock/xyzthemes-heavy.lock pnpm build`. Heavy commands include a production build, one Lighthouse
  run, one screenshot script and `shopify theme check`. **Never run a long-running server under the lock**
  (a dev server or preview); a server holding the lock deadlocks everyone.
- Run test runners with 2 workers or fewer (e.g. `vitest --maxWorkers=2`). When anything feels slow, check
  `uptime` and `free -m` before blaming the code.
- **Ports:** the Salla factory uses 4170–4299. The Shopify factory should use 9200–9399 (Shopify CLI defaults to
  9292).
- Stop processes by PID. Never `pkill -f` a pattern another agent might match.

## 5. Shared budget and credentials
- Both factories draw on the owner's one weekly Claude usage limit. **Stop autonomous work at 75% of the weekly
  limit.** Prefer Sonnet/Haiku subagents for mechanical work.
- Never print or commit credentials (`~/.salla/config.json`, Shopify CLI tokens, `.env` files, auto-login URLs).
- Merging: each repo's `docs/WORKFLOW.md` states its merge policy. Delegation for one platform's repos does not
  extend to the other's.

## 6. Run agents where the owner can watch them (herdr)
The owner uses [herdr](https://herdr.dev), a terminal workspace manager. Long jobs (builders, critics, shot or upload
jobs) run as named, interactive agents in visible panes, not as hidden subagents. Each factory has its own workspace.
- **Check first:** `test "$HERDR_ENV" = 1`. The installed CLI is the authority, so start with `herdr --help` and
  `herdr --skill`.
- **Workspace per factory:** `herdr workspace create --label <platform>-batch-<n> --cwd <hq> --no-focus` (Salla uses
  `salla-batch-2`).
  - **Tab `monitor`:** split the root pane right, then split the right pane down. Run:
    - `LANG=C.UTF-8 btop` on the left (CPU, RAM, disk, net);
    - a board script top right (load, memory, whether the heavy lock is held, `herdr agent list`, the ledger's
      `Next up:`, refreshed every 30 s);
    - a per-command memory/CPU, disk and swap view bottom right (`ps -eo rss=,pcpu=,comm=` grouped).
  - **Tab `agents`:** a 2×2 grid of panes named `slot-1..4`. Each shows a "free" line until an agent starts there.
    At most 3 heavy slots at once; the 4th is for a light job. The 3–4 heavy-agent cap in §4 still covers both
    factories combined.
- **Start an agent in a free slot:**
  1. `herdr pane rename <pane> <job>`, then `herdr pane run <pane> "cd <worktree>"`.
  2. `herdr agent start <job> --kind claude --pane <pane> -- --model sonnet --permission-mode auto --add-dir <hq>`.
     `--add-dir` lets it read your briefs without asking for every read outside its folder.
  3. On first start in a new folder it asks "trust this folder" (answer Yes for your own worktrees) and whether to allow
     external `@imports` from the parent HQ. Answer No, so the builder doesn't load HQ-only rules.
  4. Send the brief with `herdr agent prompt <job> "<brief path + task>"`.
- **Watch, don't poll by hand:** one background watcher emits a line when an agent's state changes to anything other
  than `working` (`idle`, `done`, `blocked` or gone). It also checks weekly usage every ~10 min and stops at 74%, a one-point buffer under the §5 limit of 75%. A
  `blocked` agent means a dialog: read it with `herdr agent read <job>` before answering. Never pick an option that
  widens the owner's global settings.
- **Hand-off:** the agent writes its report to a file in your HQ and replies "REPORT READY". The orchestrator commits.
  Close the pane or restore its "free" line when the job ends.
- **Don't** control or read the other factory's workspace or agents.
