# Rebuild teardown: Lovable in 7 days

**The prompt (from the job post):** "If you had to rebuild an AI app builder, pick one (rork.com, Lovable, or Base44), from scratch in 7 days, how would you attack it?" Pick: Lovable.

## Assumptions (it's underspecified on purpose)
- "Rebuild" means a working MVP of the core loop, not feature parity. Prompt in, working full-stack web app out, editable by chat, live preview, one-click deploy.
- One stack, done well: React + Vite + Tailwind on the front, Supabase (Postgres + Auth + RLS) on the back. No multi-framework, no mobile (that's rork's lane).
- Small team: me plus coding agents. Frontier models via API, nothing trained or fine-tuned.
- Goal for day 7: one happy path that actually holds, not ten that half-work.

## Architecture (the blocks that matter)
1. **Generation orchestrator (the brain).** An agent loop that turns intent into a plan, then into code: plan, generate/edit files, run, read the errors, fix. This is the product. Everything else is plumbing around it.
2. **Execution + preview sandbox.** Every user project runs in an isolated environment that installs deps, builds with Vite, and serves a live hot-reloading preview. This is the hard infra problem, not the code generation.
3. **Versioned project model.** The project is a file tree with checkpoints (undo, branch, restore). Git underneath or a snapshotting virtual file system. Without this, iteration is Russian roulette.
4. **Diff-based edit layer.** The agent applies surgical diffs, it does not rewrite whole files. This is what keeps it fast, cheap in tokens, and stops it from breaking what already worked.
5. **Backend provisioning.** A Supabase project per app: schema, auth, RLS. Reused, not reinvented.
6. **Deploy pipeline.** Sandbox to static build plus edge functions, one click.
7. **Builder UI.** Chat, code view, embedded preview (iframe), and a DB panel.

## Where agents, where code
- **Agents (non-deterministic):** intent to plan, generating and editing code as diffs, diagnosing build/runtime errors and self-healing, drafting the DB schema.
- **Deterministic code (no LLM near it):** the sandbox and its resource limits, the file system and versioning, applying the diffs, the build/deploy pipeline, rate limiting, preview streaming. Rule: the model proposes, the code validates and executes. You never let the model touch infra directly.
- **The glue that makes it reliable:** a harness that runs generated code in the sandbox, captures the error, and feeds it back to the agent with a hard retry cap. Self-heal, but bounded, or it burns money in a loop.

## Build vs buy vs reuse
- **Reuse (don't be a hero):** frontier models for generation. Supabase for DB/auth/RLS. Vite for build. An existing sandbox runtime (E2B, WebContainers, or Firecracker microVMs) instead of hand-rolling container orchestration in week one. shadcn/Tailwind as the component library the generator assembles from. An existing host for deploy.
- **Build (this is the moat, you can't buy it):** the agent orchestrator, the diff layer, the bounded self-heal loop, the versioned project model, and the glue between chat, sandbox and preview. That is the actual product. Code generation is a commodity now; reliable iteration on top of it is not.
- **Where the money leaks:** running N warm sandboxes per user is what kills the margin. The commercial problem is not "can it generate a nice app once", it's "can it iterate without breaking, at a per-user infra cost that doesn't bankrupt you". I'd instrument sandbox cost from day one.

## What I'd cut for day 7
- Out: social auth, template marketplace, real-time multi-user collab, mobile, multiple integrations, multi-framework support.
- Out: training or fine-tuning anything. Prompting plus harness only.
- In: prompt to working app, live preview, chat edits via diffs, one-click deploy. One happy path, pulled tight.

## Honest weakest point
The Achilles heel is not code generation, the frontier models already do that well. It's **reliable iteration at scale and sandbox cost/latency**. In 7 days my real risks:
- the self-heal loop getting into expensive loops or clobbering working state,
- warm-sandbox cost per user,
- context degradation: edit a large app and the agent loses the thread and starts making it worse.

I would not have those solved in a week. I'd have them mitigated (retry caps, diffs, checkpoints, cost instrumentation), not solved, and I'd say so.

The honest part from the user side: I used Lovable once for a site and ended up rebuilding it by hand in Claude Code, because the generated code didn't survive manual changes. That's the exact crack I'd prioritize: generate code an engineer can pick up and extend without it falling apart. Clean generated architecture, not spaghetti only the tool can touch. Get that right and you own the segment that actually pays: teams that outgrow the toy.
