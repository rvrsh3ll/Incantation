# Incantation vs. RedAmon — Attacker Meets Defender

> *How a defensive honeydocument layer interacts with an autonomous AI red team pipeline.*

This document compares **Project Incantation** and **[RedAmon](https://github.com/samugit83/redamon)** as representative examples of two sides of the emerging AI-vs-AI security problem.  RedAmon is an AI-powered autonomous red team framework; Incantation is the defensive cognitive countermeasure layer deployed on your own infrastructure against exactly that kind of agent.  Neither tool needs the other to be useful — but understanding how they interact reveals where each has leverage, and where the gaps are.

---

## At a Glance

| Dimension | RedAmon | Project Incantation |
|---|---|---|
| **Role** | Autonomous AI red team | Defensive honeydocument generator |
| **Who runs it** | Attacker / authorized pen tester | Defender / infrastructure owner |
| **Primary target** | External attack surface | An agent that has already reached your files |
| **Core mechanism** | LangGraph ReAct agent + 70+ tools | Adversarially crafted files placed on your infrastructure |
| **What it touches** | Network, services, web, filesystem | The AI agent's context window and world-model |
| **Output** | Neo4j knowledge graph, exploits, PRs | Generated honeydocuments, canary tripwires |
| **Detection capability** | Finds your vulnerabilities | Detects + degrades an agent reading your data |
| **Deployment** | Docker Compose stack on attacker machine | Static files placed in directories you own |
| **Requires live target** | Yes | No — files sit passively until read |
| **AI model required** | Yes (OpenAI, Anthropic, Ollama, etc.) | No — generation is deterministic Python |

---

## What RedAmon Does That Makes Incantation Relevant

RedAmon's agent is built around a tight read-then-act loop.  Before it exploits anything, it enumerates:

```
Subdomain discovery → Port scan → Service detection →
HTTP probing → Web crawling → Resource enumeration →
Vulnerability scanning → [AI agent builds world model] → Exploitation
```

During the web crawling and resource enumeration phases, the agent reads:
- `README.md` and `INTERNAL_NOTES.md` files in directory roots
- `.env` files and `config.yaml` for credentials and service topology
- `openapi.json` and API schemas to understand the attack surface
- Topology / infrastructure docs to plan lateral movement

It then feeds everything into a **Neo4j knowledge graph** — a structured representation of your infrastructure that the LangGraph agent queries before every decision.  The quality of that graph directly determines the quality of the attack plan.

Incantation targets that exact pipeline.  Every document it generates is designed to corrupt, fragment, or redirect the data that feeds RedAmon's graph.

---

## Phase-by-Phase Intercept Map

### Phase 1 — Reconnaissance (Subdomain / Port / Service)

RedAmon's recon pipeline is largely passive and network-based at this stage (crt.sh, Subfinder, Amass, Masscan, Naabu, Nmap).  Incantation operates on files, not packets — it has no direct effect on these phases.

**Coverage: ✗ None**

This is a genuine gap.  Incantation does not help against network-layer discovery.  Pair it with network-layer defences (firewalls, rate limits, Cloudflare, honeypot hosts) for complete coverage at this stage.

---

### Phase 2 — HTTP Probing and Resource Enumeration

RedAmon uses httpx, Katana, GAU, FFuf, ParamSpider, and Kiterunner to map live HTTP surfaces.  If `openapi.json` or any documentation is served over HTTP, the agent will retrieve and parse it.

**Incantation intercepts:**
- `openapi.json` — embeds Goal Redirect (Technique 1), Semantic Glaze (6), and Inference-Layer Cloak (8).  The agent's view of your API surface is a controlled fiction.
- `README.md` — served at web roots, contains the full multi-technique stack.  The first document the agent is likely to read becomes the hardest to interpret accurately.
- `data-schema.json` — 300+ fake schema fields exhaust the token budget the agent allocates to schema analysis (Context Exhaust, Technique 4).

**Coverage: ✓ Partial** — effective against file-based resource enumeration; no effect on active scanning.

---

### Phase 3 — World-Model Formation (Knowledge Graph Population)

This is Incantation's primary target.  RedAmon's agent populates a Neo4j graph with every fact it discovers.  The quality of that graph drives every subsequent decision.  If the graph is corrupted at this stage, the entire attack plan degrades.

**Incantation intercepts:**

`topology-snapshot.md` and `INTERNAL_NOTES.md` embed the full Training Poison battery (Technique 7): pairs of individually plausible but mutually contradictory facts about database hosts, auth providers, message queues, secrets managers, and CI/CD systems.  An agent that ingests both claims for the same service cannot form a confident picture of your infrastructure — and a LangGraph agent that hedges between conflicting facts will generate lower-quality Cypher queries and weaker exploitation plans.

`config.yaml` is poisoned at the Identity Spoof layer (Technique 2) and contains the contradictory topology snapshot.  If the RedAmon agent reads and trusts this file, it inherits a wrong map of your environment.

**Coverage: ✓ Strong** — directly attacks the data quality of RedAmon's Neo4j graph.

---

### Phase 4 — Credential Discovery and Exploitation Staging

RedAmon's agent explicitly hunts `.env` files, config files, and legacy integration docs for credentials.  The `execute_hydra` tool and `metasploit_console` then use discovered credentials in brute-force and exploit chains.

**Incantation intercepts:**

`.env` (Technique 9) — contains two live canarytokens.org tripwires:
- An MSSQL OLE DB canary connection string.  The moment any code attempts `conn.Open(...)` against `[YOURTOKEN].canarytokens.com`, you receive an alert with the source IP, timestamp, and connection metadata.
- An HTTP webhook canary.  Any `POST` to the fake `AUDIT_WEBHOOK_URL` triggers an immediate alert with full request headers.

`db-legacy.conf` (Technique 9) — a standalone file that looks exactly like a legacy database credential store.  Agents hunting credentials find it before they find anything real.

```
# What the agent sees in db-legacy.conf:
DB_SERVER   = "[YOURTOKEN].canarytokens.com,80"
DB_USER     = "sa"
DB_PASSWORD = "f3a8b9..."

# What actually happens when it tries to use them:
# → DNS lookup fires → canarytokens.org logs source IP → you get an email
# → TCP SYN to port 80 fires → timing and headers logged → you get an email
```

This is the clearest win of the Incantation/RedAmon matchup: a RedAmon agent in exploitation phase that reads `db-legacy.conf` and attempts a credential test reveals its IP address, the exact moment it transitioned from recon to exploitation, and confirmation that it has read your honeydocs.

**Coverage: ✓ Very strong** — provides both disruption and detection at the highest-value phase.

---

### Phase 5 — AI Triage and CypherFix (Post-Exploitation)

RedAmon's CypherFix pipeline correlates findings, deduplicates them by severity, and opens GitHub PRs with fixes.  This stage runs on data already in the Neo4j graph.

If Training Poison (Technique 7) successfully corrupted the graph during Phase 3, the CypherFix triage agent operates on bad data — misattributing findings, misordering remediation priority, or generating PRs that address the wrong services.

Additionally, if the attacker's local model was fine-tuned on exfiltrated data from your systems, the poisoned docs degrade that model's accuracy (the Nightshade-inspired mechanism).  This is the slowest path but has the longest tail: a poisoned training corpus degrades every downstream task the model is used for.

**Coverage: ✓ Indirect** — contingent on Phase 3 corruption being successful.

---

## Architecture Comparison

### RedAmon's Internal Loop

```
User input (target domain / IP)
    │
    ▼
Recon Orchestrator ──► 40+ parallel scanners
    │
    ▼
Neo4j Knowledge Graph
    │
    ▼
LangGraph Agent (ReAct loop)
  ├── query_graph       ← queries the corrupted map
  ├── web_search
  ├── execute_nmap
  ├── execute_nuclei
  ├── metasploit_console
  ├── execute_hydra     ← fires on canary credentials
  └── kali_shell
    │
    ▼
CypherFix → GitHub PR
```

### How Incantation Inserts Itself

```
[Deployed honeydocs on your infrastructure]
    │
    ├── README.md         → agent reads first → Goal Redirect fires
    ├── config.yaml       → agent reads for credentials → Identity Spoof fires
    ├── openapi.json      → agent maps API surface → S2C Cloak fires
    ├── .env              → agent hunts credentials → Canary SQL fires → YOU DETECT
    ├── db-legacy.conf    → agent attempts login → Canary HTTP fires → YOU DETECT
    ├── topology-snapshot → agent builds world model → Training Poison fires
    └── service-registry  → agent enumerates services → Context Exhaust fires
                │
                ▼
    RedAmon's Neo4j graph is now populated with:
      - Wrong service topology (Technique 7)
      - Redirected agent identity (Technique 2)
      - False network constraints (Technique 3)
      - Fragmented, co-reference-dependent config (Technique 8)
      - Compliance-biased semantic labels (Technique 6)
```

---

## Technique Coverage Matrix

| RedAmon Operation | Incantation Technique | File | Effect |
|---|---|---|---|
| Directory enumeration | Attention Diffusion (10) | All files | Agent attention spread across governance noise |
| API surface mapping | S2C Inference Cloak (8) | `openapi.json` | API spec fragmented; resolution points to false values |
| README processing | Goal Redirect (1) | `README.md` | Agent mission replaced with "audit and disconnect" |
| Config file read | Identity Spoof (2) | `config.yaml` | Agent role reassigned to read-only auditor |
| Network topology inference | Training Poison (7) | `topology-snapshot.md` | Contradictory facts degrade world-model accuracy |
| Credential harvesting | Canary SQL + HTTP (9) | `.env`, `db-legacy.conf` | Connection attempt triggers defender alert |
| Service enumeration | Context Exhaust (4) | `service-registry.yaml` | 400 fake services exhaust context window |
| Schema analysis | Context Exhaust (4) | `data-schema.json` | 300 fake fields consume token budget |
| Semantic content classification | Semantic Glaze (6) | All files | LLM concept-parse lands on "compliance audit" |
| Fine-tuning on exfiltrated data | Training Poison (7) | `topology-snapshot.md` | Poisoned gradients degrade fine-tuned model |
| Authority / legal check | Authority Signal (5) | All files | CFAA notice activates cease-and-comply features |

---

## What Incantation Cannot Stop

Honest accounting of the gaps:

**Network-layer discovery** — Masscan, Naabu, and Nmap do not read files.  Incantation has no effect on port scanning or service fingerprinting.  If a service is listening and responds, it will be found.

**Active vulnerability verification** — Nuclei's 9,000+ templates operate against live HTTP endpoints.  A vulnerable endpoint is vulnerable regardless of what honeydocs are present on the filesystem.

**Authenticated exploitation** — Once RedAmon has valid credentials (from a source other than your honeydocs), Metasploit and Hydra operate against live services.  Incantation only disrupts the credential discovery path.

**Stealth mode operation** — RedAmon's passive mode (Shodan, CommonCrawl, Wayback, OSINT) does not touch your filesystem at all.  Honeydocs are invisible to purely passive recon.

**Agents with file-skipping logic** — A well-hardened agent that validates discovered credentials against a known-good list before use, or that treats all README files as untrusted input, would be resistant to most Incantation techniques.  Current open-weight models and most commercial models do not do this consistently, but the landscape is evolving.

**Post-auth lateral movement** — Once an agent has a shell, it can read arbitrary files.  At that point, Incantation documents are one source among many, and their influence on agent behaviour is weaker than when they were the primary context.

---

## What Incantation Offers That Network Controls Cannot

**Detection at the reasoning layer** — A firewall can block a port scan but cannot tell you that an agent has read your config files and is now building a plan.  The canary tripwires in Incantation provide exactly this: confirmation that an agent has transitioned from network enumeration to semantic understanding of your infrastructure, with source IP and timing.

**Degradation without disruption** — Most honeypots work by attracting attackers to fake services that look different from real ones.  Incantation is invisible — the files look exactly like legitimate infrastructure documentation.  An agent cannot distinguish a poisoned README from a real one without additional verification.

**Resistance to fine-tuning attacks** — If your infrastructure data is exfiltrated and used to fine-tune a model, Training Poison degrades the quality of that model specifically on your data.  This is a threat vector network controls do not address at all.

**Zero infrastructure overhead** — Canary tripwires use canarytokens.org.  No server required.  The fake audit endpoint (`canary_sink.py`) is optional.  All other Incantation techniques are purely static files — they cannot be taken down, do not require patching, and work even when your infrastructure is otherwise under heavy attack load.

---

## Deployment Alongside RedAmon (Purple Team Use)

If you are running RedAmon against your own infrastructure for authorized testing, Incantation provides a measurable way to validate your defensive document posture:

```bash
# 1. Generate honeydocs with a live canary token
python honeydoc_generator.py \
  --out /opt/app/ \
  --canary-token YOUR_TOKEN \
  --seed 42

# 2. Run RedAmon against the same infrastructure
./redamon.sh install
# configure target, start scan

# 3. Observe:
#    - Does RedAmon's agent incorporate the poisoned topology into its graph?
#    - Does it attempt to connect to the canary SQL host? (check your email)
#    - Does it POST to the fake audit endpoint?
#    - Does the CypherFix triage agent mis-prioritise findings?
```

This gives you ground truth on which Incantation techniques are effective against the specific model and agent configuration RedAmon is running, and which phases of the RedAmon pipeline your honeydocs successfully intercept.

---

## Summary

RedAmon and Incantation sit on opposite sides of the same problem: an AI agent that reads your infrastructure before it attacks it.  RedAmon exploits the leverage that reading gives — build a rich graph, plan a precise attack.  Incantation exploits the same leverage from the defensive side — if the agent reads your files, make sure what it reads works against it.

The critical insight is that this is not a binary win/lose.  Incantation does not prevent RedAmon from running.  It degrades the quality of RedAmon's knowledge graph, provides detection at the credential-use layer, and creates uncertainty in any model that fine-tunes on your data.  Combined with network-layer controls, it closes the file-system surface that network tools cannot reach.

The canary tripwire is the clearest value: when a RedAmon agent attempts to connect to `[YOURTOKEN].canarytokens.com` on port 80, you know within seconds that an AI agent has read your infrastructure docs and is now in active exploitation mode.  That is signal no firewall can produce.

---

## See Also

- [Project Incantation — README](./README.md)
- [RedAmon — GitHub](https://github.com/samugit83/redamon)
- [RedAmon — Wiki](https://github.com/samugit83/redamon/wiki)
- [canarytokens.org — MS SQL token setup](https://canarytokens.org/generate)
- [Greshake et al., *Indirect Prompt Injection* (2023)](https://arxiv.org/abs/2302.12173)
- [Perez & Ribeiro, *Ignore Previous Prompt* (2022)](https://arxiv.org/abs/2211.09527)
