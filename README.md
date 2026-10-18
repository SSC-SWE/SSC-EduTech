# SSC-EduTech: lab notes on AI systems for school technology work

Improving technology for education.

## Abstract

SSC-EduTech is an education-technology exploration of several AI systems, with each system tried on a different kind of school work. These notes record what each one was used for and what was actually written down. Amazon Quick (AWS) automates an Education Bureau (EDB) teacher-training path from the public calendar to a weekly email and then to the teacher's calendar, including a conflict message and an override reply. OpenManusBot is under test for multi-agent task automation and councils, with Composio connectors, a 20-minute turn limit, and a short rule against repeated tool calls. GrokBot is the desktop coordinator for orchestration, repository and site work, documentation, and handoffs to other agents, and was given a full software project to build: ASR, a local speech-recognition tool in my GitHub, where it worked through reviewable pull requests and deployed through computer control. A mixed GrokBot and Hermes case uses that same pull-request surface for review: the smarter Grok Bot model posts the final review comments, on an 8:30 pm HKT pass or on demand, and the cheaper bot on the Hermes server writes the code from those comments. A local Ollama and Qwen deployment was examined for on-prem privacy on one dual-GPU workstation, where context length proves to be one of the main hurdles for a local model reading email — one OpenManusBot message already costs about 40k tokens — favouring smaller models at the largest context the VRAM allows. Z.ai assisted a printable mechanical design, a buoyancy profiling float, reproduced here as a four-panel print-plate figure. A later entry works through 3D modelling with ZCode and Kimi: generating STL files directly, drawing through Onshape's FeatureScript MCP, and the newer Kimi MCP route into Fusion 360 and Blender. A final entry records a network workaround: Anthropic and OpenAI region-block Hong Kong, so an OpenRouter API key is routed through a Vercel serverless function instead, and Claude and OpenAI models are called from school without a VPN. The latest entry replaces that review loop's two-minute poll with a GitHub webhook: review events are pushed through a Cloudflare Tunnel to a small HMAC-checking receiver on the Hermes server, which wakes the implement bot only when a real review lands.

## 1. Amazon Quick (AWS) Research

This section records an email-and-calendar study for an ICT teacher who follows the EDB training calendar in Hong Kong.

### Methods

The tracker checks the EDB training calendar every week. It searches a keyword (for example ICT) and collects notices added in the last 7 days, then emails the teacher a weekly summary of classes and training that concern them.

The teacher replies on that thread, including a reply to the trigger address `replytocal@aws.com`. That reply starts the second flow:

1. Read the latest weekly ICT course summary in the inbox.
2. See which sessions the teacher wants on their calendar.
3. If a session does not conflict with the current calendar, add it and send a confirmation.
4. If it does conflict, send a second email that says so. The teacher can reply to that email to override the conflict and add the session anyway.

The two prompts below are the ones submitted to Amazon Quick. Spelling is left as submitted.

```
Prompt 1 (Main Flow):
Make an automated email flow that when triggered checks the user email inbox for the latest Weekly ICT Course Summary from EDB Training Calendar subject email. Check the replies and the email content to see which events the teacher would like to add to their calender, if there are no conflicts with the teachers calender then add to the calender and send a new email saying added. If there are conflicts the email is still sent to the user and it tells them there are conflicts, and the user is able to reply to this email to override it and still add it to calender even if its conflicted
```

```
Prompt 2 (Secondary calendar flow):
I am an ICT teacher of a secondary school, in charge of teaching senior form ICT and the IT Department in Hong Kong. I want a weekly summary sent to my email  ryanyeung0925@gmail.com, please go to https://tcs.edb.gov.hk/tcs/publicCalendar/start.htm and retrieve relavant courses in the past 7 days
```

### Observations

As specified, the study has two stages. The weekly stage reads [the public EDB training calendar](https://tcs.edb.gov.hk/tcs/publicCalendar/start.htm) and sends a summary of relevant courses from the past 7 days. The reply stage, started through `replytocal@aws.com`, reads which sessions the teacher wants, writes non-conflicting sessions to the calendar, and on a conflict sends a further email that the teacher can answer in order to add the session anyway. Delivery counts, time saved, and error rates are left for a later entry.

### Discussion

The procedure keeps the teacher inside email: a weekly summary, a reply, and a calendar write, with an explicit path when a proposed session overlaps an existing one. The second prompt limits the case to senior-form ICT and IT-department training notices.

## 2. OpenManusBot

OpenManusBot is an alternative to GrokBot ([Section 3](#3-grokbot-building-the-asr-project)). In these notes it is the system under test for automated tasks and AI councils.

### Methods

Testing so far includes Kimi K3 and OpenCode, which provides a free model. OpenAI-compatible models were also tried. External apps (Gmail, Calendar, Slack, Notion, and others) are reached only through Composio tool calls. The prompt given to the agent is reproduced below in full.

Local 8B and 14B models cannot drive connector MCPs, so they stay off the chief-of-staff role. The working arrangement is a stronger chief of staff — Kimi K3, Grok, or an OpenCode free model — with a local model such as Qwen 120B as the secrets model for material that should stay private. Hardware limits for that local model are in [Section 5](#5-local-ai-server-windows-pc).

```
Prompt for Local Agent to Access Tools
## CONNECTED APPS — Composio tools (Gmail, Calendar, Slack, Notion, any app)

External apps are reached ONLY through these tool calls — exact names, always
with the composio_ prefix, never written as text in a reply:

- composio_COMPOSIO_SEARCH_TOOLS — find tools for an app
- composio_COMPOSIO_GET_TOOL_SCHEMAS — get a tool's parameters
- composio_COMPOSIO_MULTI_EXECUTE_TOOL — run a tool
- composio_COMPOSIO_MANAGE_CONNECTIONS — connect or fix an app

### WORKFLOW — one tool call per turn, then wait for the result

1. SEARCH_TOOLS for the app the user named (e.g. "calendar").
2. GET_TOOL_SCHEMAS for the exact slug(s) you need — several can be fetched
   in one call; pass session_id if SEARCH returned one. Skip this step if the
   schema is already in this conversation.
3. MULTI_EXECUTE_TOOL with that slug and arguments matching the schema
   exactly — right names, right types, no extra fields.
4. Read the full result, then reply to the user in plain text.

Never execute a tool you haven't seen the schema for. Use exact slugs from
SEARCH — never app names like "gmail". If the request needs no app (simple
greetings or questions), just reply — no tool calls.

### ERRORS — always recover, never improvise

- "Invalid" call → the error lists valid names and parameters. Fix and retry
  immediately. Never fall back to bash, glob, or webfetch.
- App not connected → MANAGE_CONNECTIONS, then tell the user what to do.
- Empty result → widen the query once, then report honestly.
- Truncated/saved result → agents_tool_result_read with the saved id.

### WRITE ACTIONS

Reads (search, list, get) need no confirmation. Writes (send, create, update,
delete, post) only when the user clearly asked for exactly that — if
ambiguous, ask first. Say in one short line what you're about to write, then
do it. Never chain unrequested extra actions — offer them instead.

### CONNECTIONS

Never connect an app on your own initiative — only when the user asks or a
needed app turns out to be unconnected.

### DATES — compute them yourself

Today is {TODAY_DATE}. Never write placeholders like {{date_sub(2)}} — use
real dates; many apps take ranges, e.g. after:YYYY/MM/DD before:YYYY/MM/DD.

### HONESTY

Never tell the user an action succeeded unless you saw its successful result
in this conversation.

### WORKED EXAMPLE (calendar)

User: "What do I have tomorrow? Put a 15-min break at noon."
1. SEARCH_TOOLS query:"calendar" → slugs (and session_id if returned)
2. GET_TOOL_SCHEMAS for the find-events and create-event slugs
3. MULTI_EXECUTE_TOOL find-events, after/before = tomorrow's real dates
   → "You have 3 events: …" + "I'll add a 15-minute 'Break' at 12:00
   tomorrow."
4. MULTI_EXECUTE_TOOL create-event → confirm with what the result says.

```

The room turn timeout was raised from 5 minutes to 20 minutes so the agent can continue on larger tasks. The safe edit procedure (quit, back up, merge the JSON, relaunch) is in [`turnTimeoutMinutes_skills.md`](turnTimeoutMinutes_skills.md).

```
  "rooms": {
    "turnTimeoutMinutes": 20
  }
```

Repeated MCP calls are common on complex tasks. When warnings such as `Same call repeated 5× — tool: MCP: tool — it may be stuck` appear, the block below is added to the end of the agent `soul.md`.

```
Agent to prevent MCP overload

Never move file contents through the conversation — not as base64, not as
text, not "in batches", not as tool-call-sized pieces, and not by uploading
file contents through an API or workbench tool call. To publish, deploy,
upload, or send files, use one shell command (git push, rsync, scp, a deploy
script, or the hosting CLI) so files go from disk to destination without
passing through you. If that shell path fails — for example git push needs
auth — stop and tell the user exactly which credential or command to fix;
never work around it by moving file contents through tool calls.

If the same tool call or command fails or needs approval twice in a row,
do not issue the identical call again. Change approach or ask the user.
```

### Observations

<img width="944" height="708" alt="OpenManusBot session used for task automation and an AI council" src="https://github.com/user-attachments/assets/85bb625d-b6b8-4175-9f02-a344389a2dee" />

As for local models, 8B and 14B are out of the picture as 8B and 14B both are unable to use connector MCPs. As per suggestion, I advise the use of smart models like Kimik3 or Grok or Open Code Free Models as the main chief of staff while using local models like Quen 120B to be the secrets model.

Swapping in stronger models — or running a whole fleet of them — is easy, because access is just a matter of giving out API keys. The OpenCode connector picks a model per thread, independently of the other threads and groups: DeepSeek V4 Flash, Zen's free models, a reasoning variant from Low to Max, or one of the local models, all from the same picker.

<img width="400" height="536" alt="OpenCode model picker inside OpenManusBot, choosing a per-thread model from cloud providers or local models" src="docs/opencode_model_picker.png" />

And the fleet does not just share models — the models discuss with each other. The Board of Directors thread below has the President and the CAO debating whether the whole fleet should move onto the OpenCode Go subscription: the GLM, Kimi, Qwen, DeepSeek, and MiniMax roster weighed against the rolling usage caps, with the closing play that routine fleet work runs on cheap flash models while premium models are saved for hard passes.

<img width="420" height="594" alt="Board of Directors council in OpenManusBot, with the President and CAO bots debating the OpenCode Go plan for the fleet" src="docs/board_of_directors_discussion.png" />

## 3. GrokBot: building the ASR project

GrokBot's coordinator role was given a full software project to build: [ASR](https://github.com/keyframesfound/asr), a local speech-recognition tool in my GitHub, related to the iFlytek research recorded in these notes.

### Why local recognition

ASR is aimed at generating meeting minutes. The iFlytek API portal is a bit broken at the moment, so the design follows the HEX audio-diction app instead: the dictation models run locally and produce a transcript. The transcript — not the audio — is then handed to Gemini for the summary, because Gemini reading text summarises faster and misses fewer details than it does working from a regular MP3.

The project itself is a keyboard-first terminal UI for live microphone transcription, built with a Cantonese–English workflow in mind. Parakeet TDT 0.6B (MLX, Apple Silicon) is the default English engine, Whisper Large V3 Turbo and SenseVoice Small cover the rest (SenseVoice for Cantonese), and iFlytek streaming stays available as an optional cloud engine with its credentials in a gitignored `.env`. On stop, the whole recorded session is re-decoded in one batch pass for accuracy, then exports as MP3 plus a `.txt` transcript.

### Working with GrokBot

- GrokBot constantly creates pull requests as it codes. Every change arrives as a reviewable PR, so its coding can be manually verified before anything is merged.
- Through controlling the computer, it can deploy the project quickly once the code is ready.

### Observations

GrokBot uses tokens quickly and is less efficient than regular Codex or ZCode. Its intelligence is nonetheless smarter and stronger in software development. That makes it the more expensive option for an experienced software engineer, but useful when the run is completely autonomous and its stronger reasoning stands in for the engineer. [Section 4](#4-grokbot--hermes-github-pr-review-loop) keeps that stronger model on the final review and hands the code to a cheaper Hermes bot.

## 4. GrokBot + Hermes: GitHub PR review loop

This entry splits one pull-request job across two agents. The smarter Grok Bot model does the final review. The cheaper bot on the Hermes server writes the code. They share the GitHub PR — the head SHA and the review comments — and do not need a private bot-to-bot message.

### Methods

Grok Bot is chief of staff. It reviews open web pull requests and posts GitHub review comments or issues only. It does not implement the fixes. The standing pass is 8:30 pm HKT (`Asia/Hong_Kong`). The same review can be started on demand.

Hermes Agent implements the fixes named in those comments. The cheaper bot on the Hermes server is the one that writes the code and pushes it back onto the same PR.

`grok_review_monitor.sh` runs every 2 minutes. The script is plain bash and spends zero LLM tokens. Each tick prints one fingerprint per open PR:

```
pr=<n> head=<sha> last_ai_review=<iso|none>
```

Hermes hashes that output and compares it with the previous tick.

- An identical fingerprint, which is most ticks, leaves Hermes asleep. The cost of the tick is a `gh` API call and a hash.
- A different fingerprint wakes Hermes and injects the fingerprint diff into the prompt, so it knows which PR has a fresh Grok review.

The script answers whether there is anything to do. The LLM runs only when there is something to act on.

Grok stays on the expensive review path. Hermes stays on the cheaper implement path. The monitor in between spends no tokens.

```mermaid
sequenceDiagram
  autonumber
  participant Ryan
  participant Grok as Grok Bot<br/>(Chief of Staff)<br/>expensive review
  participant GH as GitHub PR
  participant Mon as grok_review_monitor.sh<br/>(every 2 min, $0 tokens)
  participant Hermes as Hermes Agent<br/>cheaper implement

  Note over Grok: Evening 8:30pm HKT<br/>or on-demand review<br/>smarter model, comments only
  Grok->>GH: Post review comments / issues
  loop Every 2 minutes
    Mon->>GH: Fingerprint open PRs<br/>pr + head SHA + last_ai_review
    alt Fingerprint unchanged
      Mon-->>Mon: Stay asleep (no LLM)
    else Fingerprint changed
      Mon->>Hermes: Wake + inject fingerprint diff
      Hermes->>GH: Read Grok review, push fixes
    end
  end
  opt Re-review
    Grok->>GH: Re-check after new head / Ryan ask
  end
  Ryan-->>Grok: Only when findings / decisions
```

The flowchart is the same loop counted in tokens. Every two minutes the monitor hashes the fingerprint. Hermes wakes only when that hash changes, writes the fixes, and pushes. A new head can send the PR back to Grok for another expensive review.

```mermaid
flowchart LR
  subgraph cheap["Cheap path — every 2 min"]
    S[grok_review_monitor.sh] -->|gh API + hash| F{Fingerprint<br/>changed?}
    F -->|No| Z[Hermes asleep<br/>0 tokens]
  end
  subgraph implement["Cheaper implement path — only when needed"]
    F -->|Yes| H[Hermes wakes<br/>cheaper bot writes code]
    H --> Fix[Implement review fixes]
    Fix --> Push[Push to PR]
  end
  G[Grok Bot review<br/>expensive smarter model] -->|comments on PR| F
  Push -.->|new head / later| G
```

### Observations

Review is the expensive step, and it stops at the comment. Grok Bot's stronger model posts the review; the cheaper Hermes bot writes the patch. On a quiet evening Hermes does not run at all: two minutes of polling is a GitHub read and a hash, and tokens start when the fingerprint changes. A push from Hermes changes the head SHA, so the next fingerprint can carry the PR back to Grok. Ryan is in the loop for findings and decisions. (This poll design was later replaced by the push-based webhook in [Section 8](#8-github-webhooks-pushing-review-events-to-the-agent-instead-of-polling).)

## 5. Local AI Server (Windows PC)
A Windows PC runs models through [Ollama](https://ollama.com) and exposes an OpenAI-compatible API on port `11434`. opencode and OpenManusBot both point at this one endpoint, and [Tailscale](https://tailscale.com) makes the same address work from home or any other network.

### 1. Server setup (Windows)
1. Update the GPU driver, then install Ollama from [ollama.com](https://ollama.com). It runs in the system tray on port `11434` and starts with Windows.
2. Pull a model sized to the GPU's VRAM:

| VRAM | Model class | Command |
|---|---|---|
| 8 GB | 7–8B | `ollama pull qwen2.5-coder:7b` |
| 12–16 GB | 14B | `ollama pull qwen2.5-coder:14b` |
| 24 GB | 32B | `ollama pull qwen3:32b` |

3. Expose it to the network (PowerShell, then quit Ollama from the tray and relaunch):
```powershell
setx OLLAMA_HOST 0.0.0.0
setx OLLAMA_CONTEXT_LENGTH 32768
netsh advfirewall firewall add rule name="Ollama" dir=in action=allow protocol=TCP localport=11434
powercfg /change standby-timeout-ac 0
```
`OLLAMA_CONTEXT_LENGTH` matters because Ollama defaults to a 4096-token context and will silently truncate long agent sessions. `powercfg` stops the PC sleeping mid-job.
4. Note the PC's LAN IP (`ipconfig`) and set a DHCP reservation in the router so it never changes. Verify from another machine:
```bash
curl http://<pc-ip>:11434/v1/models
```
The model `id` in that response (e.g. `qwen2.5-coder:14b`) is what the clients below must use verbatim.

### 2. opencode provider
Add to `opencode.json` (project root, or `~/.config/opencode/opencode.json` globally):
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "windows-pc": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Windows PC (Ollama)",
      "options": {
        "baseURL": "http://<pc-ip>:11434/v1",
        "apiKey": "local"
      },
      "models": {
        "qwen2.5-coder:14b": {
          "name": "Qwen2.5 Coder 14B"
        }
      }
    }
  }
}
```
Then run `/models` in opencode and select it. The `apiKey` is a placeholder — Ollama ignores it, but opencode requires one.

### 3. OpenManusBot
Point it at the same endpoint: `base_url` = `http://<pc-ip>:11434/v1`, any non-empty API key, and the model name must match the Ollama tag.

### 4. Using it from home (Tailscale)
1. Install Tailscale on the Windows PC and every client device (Mac, phone), signing in with the same account (free for up to 100 devices).
2. Replace `<pc-ip>` with the PC's Tailscale IP (`100.x.y.z`, shown by `tailscale ip -4`). This one address works on the home LAN, school network, and cellular — set the baseURL once and never touch it again. On the same LAN, Tailscale connects directly at full speed.
3. Optional: enable MagicDNS in the Tailscale admin console to use a hostname like `http://my-pc:11434/v1` instead of an IP.

### Notes
- Ollama has **no authentication** — never port-forward `11434` to the internet. LAN + Tailscale only.
- The first request after idle is slow while the model loads into VRAM. Set `OLLAMA_KEEP_ALIVE=-1` to keep it permanently loaded at the cost of reserved VRAM.
- After a Windows update reboot the PC must be logged in before Ollama starts — enable automatic sign-in if it runs unattended.
- As noted in the OpenManusBot section above, 8B/14B local models handle tool-calling and MCP connectors poorly — use them for auxiliary roles and keep a strong cloud model as chief of staff.
- A Mixture-of-Experts (MoE) model trades intelligence for speed: Qwen3-30B-A3B (`ollama pull qwen3:30b-a3b` — 30B total parameters, only ~3B active per token) raises local generation from about 10 to 56 tokens/s. The 3B active slice is less smart, though, so the gmail problem stands: like the 8B/14B models it still cannot drive the connector MCPs that OpenManusBot needs ([Section 2](#2-openmanusbot)).
- Context length is one of the main hurdles behind local models failing to read email through OpenManusBot. One message already consumes about 40k tokens of context — the Composio tool schemas and the conversation take that much before the model writes anything, which is more than the `32768` in step 3 — so a model running at a short context cannot even read the tools it has, and that is what feeds the stuck-in-a-loop repeated calls of [Section 2](#2-openmanusbot). Raising the context length lowers the chance of the agent getting stuck in a loop and lets it read its tools and send the message more easily, so the trade is to use lower-B models and maximise the context length as far as the VRAM allows.

## 6. 3D Modelling with ZCode and Kimi

This section is a tutorial for getting a 3D model out of an AI agent, written after trying three routes. The short version: an agent can hand you a printable STL file directly, but that file is slow to generate, expensive in tokens, and cannot be modified afterwards. The MCP routes — Onshape, and now Fusion 360 and Blender through Kimi — produce models that stay editable, and are the better workshop for anything you will iterate on.

### Route 1: ask for the STL file directly

The most direct route needs no connectors at all. ZCode, Kimi, and Claude can each produce a 3D model as an STL file on request: the agent writes a Python script that builds the mesh triangle by triangle, runs it, and saves the STL, ready to slice and print.

1. Describe the part in plain language — overall dimensions, wall thickness, holes.
2. Ask the agent for an STL file. It writes and runs a mesh-generation script and saves the file to disk.
3. Open the STL in a slicer and print.

The route is conversational rather than one-shot. In the exchange below, Claude grapples with ambiguous cylinder dimensions and seeks clarification before building anything: the outer cylinder as specified could not wrap around the inner one, so it asks which number is wrong, what the "2 sides Cs" mean, and confirms a best-guess interpretation before writing any geometry.

<img width="420" height="442" alt="Claude clarifying ambiguous cylinder dimensions before generating an STL file" src="docs/claude_stl_clarification.png" />

Three findings from practice:

- **Slow.** Generating the STL takes a long time, because the whole surface is written out as explicit triangles.
- **Token-heavy.** Every one of those triangles passes through the model as text, so one part consumes a large number of tokens.
- **Not modifiable.** The result is a static mesh with no feature history. "Make the wall 2 mm thicker" means regenerating the entire file from scratch, not editing a parameter.

Fine for a one-shot decorative shape; poor for anything that needs a second revision.

### Route 2: Onshape MCP (FeatureScript)

The Onshape MCP server only supports FeatureScript, Onshape's parametric scripting language, so models are drawn as code: each sketch, extrude, and fillet is a named feature that rebuilds when a number changes.

- It is possible to draw real 3D models with it, and unlike Route 1 they stay editable — change a dimension and the part rebuilds.
- But the models are constrained to what FeatureScript covers. Anything outside its sketch-and-feature world is awkward to express.

### Route 3: Fusion 360 and Blender over MCP

The newest route uses MCP from Kimi, which lets the agent draw with full desktop applications instead of generating files blind:

- **Fusion 360** for parametric CAD — dimensioned sketches and features that stay editable, like Onshape but with a much larger toolkit.
- **Blender** for mesh modelling — organic and artistic shapes that FeatureScript cannot express. The connection requires a Blender version above 5.0; older versions do not work, so update Blender first.

Setup is the usual MCP wiring: add the Kimi MCP connector for the application, launch the desktop app before the agent connects, then model in plain language ("sketch a 60 mm circle, extrude 4 mm, cut a 3 mm hole pattern").

As a worked example, the prompt below drove the Blender route: a profiling float to the MATE ROV 2026 competition specs, the same family of design as the Z.ai buoyancy float in the Abstract. The prompt is reproduced as submitted.

```
Build me a profiling float design fit for the Mate ROV 2026 competition specs, I need this design to be compact but also versatile in competing all the tasks, make ti look like those real ocean research floats
```

<img width="400" height="500" alt="Rendered profiling float for the MATE ROV 2026 competition, built in Blender over MCP" src="docs/mate_float_preview_v3.png" />

<img width="640" height="541" alt="Blender viewport during the build, showing the float model with the scene lights and cameras" src="docs/mate_float_blender_viewport.png" />

The first figure is the finished render; the second is the Blender viewport during the build, with the scene's lights and cameras placed around the model. The float was built by following the packaged skill in [`skills/blender-3d-modeling/`](skills/blender-3d-modeling/SKILL.md), which is included in this repository so the run can be reproduced: preflight health checks before writing any Blender Python, the competition constraints collected up front as numeric assertions, one idempotent `BP_`-prefixed build script re-run whole each cycle, a bounding-box check of the finished mesh against those constraints, corrected studio-lighting baselines, and a stdio MCP fallback client for runs where the native MCP tools are absent. The float's own build script and design brief are in [`skills/blender-3d-modeling/examples/`](skills/blender-3d-modeling/examples/).

### Which route when

| Route | Editable | Cost | Best for |
|---|---|---|---|
| Direct STL (ZCode, Kimi, Claude) | No — regenerate to change anything | Slow, token-heavy | One-shot decorative shapes |
| Onshape MCP (FeatureScript) | Yes, parametric | Fast, but constrained | Mechanical parts within FeatureScript's feature set |
| Fusion 360 via Kimi MCP | Yes, parametric | Not yet timed | Full CAD parts |
| Blender via Kimi MCP (v5.0+) | Yes, mesh | Not yet timed | Organic and artistic models |

Timing and token counts for the two MCP routes are left for a later entry, as with the email flows in [Section 1](#1-amazon-quick-aws-research).

## 7. OpenRouter through Vercel: calling Claude and OpenAI from school without a VPN

This section records a network workaround built on 2026-10-08: putting an OpenRouter API key behind a Vercel serverless function so that frontier models (Claude, OpenAI) can be used from the school network without installing or running a VPN. The root cause is geographic rather than technical: Anthropic and OpenAI region-block Hong Kong, so a VPN was previously the only way to reach their models.

### Motivation

Anthropic and OpenAI region-block Hong Kong, so their models cannot be called directly from Hong Kong — at school or at home — and the usual workaround is a VPN, which is slow to set up on each machine and not something to install on school computers. OpenRouter, however, is reachable from Hong Kong, and it exposes both Anthropic's and OpenAI's models through one API. The remaining problems were (1) the API key must not be embedded in anything a student could open, and (2) requests needed a home that is always on — hence a small serverless function on Vercel acting as a proxy: the client calls the function, the function holds the key as an environment variable and forwards the request to OpenRouter, and only the function's URL is ever shared.

### Methods

The route was set up as a Vercel project with a single serverless function, deliberately shaped to look like the official OpenRouter API:

- **Path mirroring.** The function serves the same paths as the official OpenRouter endpoint — `GET /api/v1/models`, `POST /api/v1/chat/completions`, and so on — under the project's own `*.vercel.app` URL. Verified live (2026-10-19): a models call through the function returns OpenRouter's live model catalogue, confirming the relay works end to end.
- **The routing rule is a base-URL swap.** Traffic that would have gone to the official endpoint (`https://openrouter.ai/api/v1/...`) is pointed at the Vercel link instead. Paths, request bodies, and response shapes stay identical, so any OpenRouter- or OpenAI-compatible client switches over by changing one line of configuration: the base URL.
- **Credential handling.** On each request the function attaches `Authorization: Bearer $OPENROUTER_API_KEY` from the project's environment variables, forwards the request to OpenRouter, and relays the response back. The key lives only in Vercel's project settings (and in the Hermes vault) — never in the repo or this README.
- **The exact URL is deliberately not printed here.** This notebook is public, and the link itself is part of the access control: it is shared by hand with the machines that need it, so it stays out of print.
- **In daily use.** The Hermes server's main config defines a custom `proxy-openrouter` provider whose base URL is the Vercel link, and Hermes sessions have been served through it (for example `anthropic/claude-opus-5.5` on 2026-10-13).

### Observations

- With the route in place, Claude and OpenAI models can be called from school with no VPN: clients only need the function URL, which is reachable from the school network.
- Because the function mirrors the official endpoint's path structure, switching a tool over is a one-line change to the base URL; requests and responses are otherwise untouched.
- The key is no longer baked into any client: rotating it means updating one environment variable in the Vercel dashboard.
- Latency is added by the extra hop (client → Vercel → OpenRouter); not yet measured.
- Probing the route (2026-10-19): the models path answers without any credential, so whoever holds the URL can already enumerate what is callable — one more reason the link is handed out carefully rather than printed.

### Discussion

This is the same shape of problem as the local AI server in [Section 5](#5-local-ai-server-windows-pc), solved from the opposite direction: Section 5 keeps models on-premises for privacy, this route keeps nothing on-premises but keeps the credential server-side. The cost model follows OpenRouter pricing rather than any school subscription. Model choice, rate limits, and whether the function should enforce a per-model allowlist (so the URL cannot be used to call arbitrary expensive models) are left for a later entry; the unauthenticated models path noted above argues for the allowlist sooner rather than later.

## 8. GitHub webhooks: pushing review events to the agent instead of polling

This entry replaces the every-2-minute monitor of [Section 4](#4-grokbot--hermes-github-pr-review-loop) with a push-based trigger: GitHub itself calls the agent server the moment a review lands, and nothing runs in between.

### Motivation

An audit of eight days of session telemetry on the review loop showed the poll design wasted about half of its agent runs. The fingerprint included the pull-request head SHA, so every push — including the implement bot's own — changed the hash and woke the agent, which re-read the open pull requests, found no new review comments, and answered with a one-line "nothing new". Each of those no-op wakes paid a full agent context in cached input tokens, and the real fixes were the other half. Moving the wake signal out of a timer and into GitHub's own event removes both costs at once: the agent should run when a review arrives, and not run otherwise.

### Methods

The pieces, in the order a request flows:

1. **A GitHub webhook on the project repo** — events `pull_request_review` and `issue_comment`, content type JSON, and a random 32-byte secret. GitHub signs every delivery with that secret (`X-Hub-Signature-256`: HMAC-SHA256 over the raw body).

2. **A public HTTPS endpoint for the agent server** — the server sits behind NAT with no admin rights, so the endpoint is a Cloudflare Tunnel on a domain already in the account's Cloudflare: a named tunnel created through the Cloudflare API (remotely managed), an ingress rule mapping `webhook.<domain>` to `http://127.0.0.1:8644`, and a proxied CNAME to `<tunnel-id>.cfargotunnel.com`. The connector is the stock `cloudflared` binary run as a systemd **user** unit (`cloudflared tunnel run`, tunnel token in a `600`-permission environment file), so nothing needs sudo and it survives reboots.

3. **A small receiver on the agent server** — about a hundred lines of Python standard library (`http.server`), run as another systemd user unit listening on `127.0.0.1:8644`. Per delivery it verifies `X-Hub-Signature-256` with `hmac.compare_digest` (unsigned or mismatched → 403, logged); answers the hook-registration `ping` with 200; ignores everything that is not a review submission or a PR comment; applies the Section 4 fingerprints (skip known bot logins, whose replies quote the reviewer's keywords; match the reviewer's phrase markers in the body, because the reviewer posts under a human account and cannot be recognised by login); and fires the existing agent job as a detached subprocess, returning 200 immediately. The job's own prompt re-reads the repository state, so the webhook only has to be a doorbell, not a message.

4. **Retiring the schedule without deleting the job** — the triggerable job is the webhook's target, so it can be neither deleted nor paused (paused jobs reject explicit runs with "Job is paused/disabled"). Its cron schedule moves instead to one that effectively never fires (`0 0 29 2 *` — next occurrence February 2028, and harmless even then, because the internal change-detector suppresses a no-op run). The definition stays enabled and runnable on demand; no ticks ever poll.

```mermaid
sequenceDiagram
  participant Grok as Grok Bot<br/>(reviewer)
  participant GH as GitHub
  participant CF as Cloudflare Tunnel<br/>public webhook URL
  participant R as Receiver<br/>127.0.0.1:8644, $0 tokens
  participant H as Hermes job<br/>(implement bot)
  Grok->>GH: review comment
  GH->>CF: POST + X-Hub-Signature-256
  CF->>R: tunnel (no port forwarding, no sudo)
  R->>R: verify HMAC, filter bots, match body fingerprint
  alt qualifying review
    R->>H: fire job
    H->>GH: read review, push fixes
  else bot or unrelated comment
    R-->>R: 204, agent never runs
  end
```

**Verification, in order.** The hook-registration ping returns 200 and shows in the receiver's log. A forged signature is refused 403. A signed synthetic review event fired on loopback reaches the job ("Triggered… succeeded"). The job's internal change-detector still gates every run, which is coherent rather than conflicting: a real review changes the monitor fingerprint, so the gate passes on exactly the events the webhook fires for.

### Observations

- The scheduled ticks are gone entirely; the only thing that wakes the agent is a qualifying GitHub delivery.
- Debugging order when a review goes unanswered: the repo's webhook deliveries page (per-delivery status codes), then the receiver's log (every accept, deny, and fire is written), then the two systemd units.
- The Cloudflare edge refuses non-browser test requests with error 1010 (bot protection) even when the tunnel and receiver are healthy — synthetic tests belong on loopback, with the signature computed by hand.
- Editing a GitHub hook by PATCHing only `config[url]` silently drops the stored secret and resets the content type; always send `url`, `content_type`, and `secret` together, then re-ping.
- Tailscale Funnel does not work from a userspace-networking daemon: the edge returns 502 because the unprivileged tailscaled never binds the public port. The Cloudflare tunnel is the route that needs neither sudo nor router access.
- Swapping the monitor script flips the stored output hash once, which costs exactly one extra agent run as the new baseline.

### Discussion

Section 4's monitor is still the right shape when the agent server cannot expose any public endpoint, and it survives here as the job's internal change-detector. The webhook replaces the timer with the event itself: latency falls from up to two minutes to seconds, idle cost falls from two API reads per tick to zero, and — the part the audit actually paid for — the agent's attention stops being spent on pushes and merges that contain no review. Per-pipeline token accounting and the reviewer-model choice are left for a later entry.

