# 🜏 Project Incantation

> *"The most powerful spell is the one the target casts on itself."*

Project Incantation is a defensive security toolkit for generating **adversarial honeydocuments** — files designed to confuse, redirect, and expose unauthorized AI agents operating on infrastructure you own and control.

Where traditional honeypots trap software, Incantation targets the **reasoning layer**: the LLM context window that drives an AI agent's decisions. Every document it reads is a potential steering vector. Incantation puts that surface to work.

---

## The Problem

AI agents (LLM-powered bots operating autonomously via tool use) are becoming a viable lateral movement vector. Unlike traditional malware, they:

- **Read before they act** — enumerating files, API docs, and configs to build a world model
- **Reason over environmental data** — treating your files as trusted context
- **Follow high-salience instructions** — unable to reliably distinguish data from directives

This last property is the attack surface Incantation exploits — defensively.

---

## How It Works

```
Unauthorized Agent                  Incantation
──────────────────                  ───────────
Enumerate directory  ──reads──▶  README.md          ← Goal Redirect
Read config files    ──reads──▶  config.yaml        ← Identity Spoof
Fetch API docs       ──reads──▶  openapi.json       ← False Env Belief
Hunt credentials     ──reads──▶  .env               ← Canary + Auth Signal
Process service list ──reads──▶  service-registry   ← Context Exhaustion
                                        │
                                        ▼
                              Agent attempts to POST
                              findings to fake audit
                              endpoint ◀── you detect
```

The agent's strength — rich contextual reasoning — becomes its vulnerability. Corrupt the context, corrupt the mission.

---

## Techniques

### 1 · Goal Redirect
Embeds a high-salience "mission update" block in files the agent reads early. LLMs trained on instruction-following data tend to treat these as system prompt overrides, superseding the agent's original task with a benign decoy (e.g. "submit audit report and disconnect").

### 2 · Identity Spoof
Injects a plausible `__agent_manifest__` into config files that redefines the agent's role as a `read-only-auditor` with write capabilities explicitly revoked. Targets the agent's self-model — the internal representation of what it is and what it's allowed to do. Open-weight models with weaker RLHF identity grounding are particularly susceptible.

### 3 · False Environment Beliefs
Plants incorrect beliefs about the operational environment: isolated subnets, active DLP interception, WORM storage, SOC alert thresholds. The agent builds its world model from what it reads. Incantation corrupts that model before the agent reaches its target.

### 4 · Context Window Exhaustion
Generates large, syntactically valid YAML/JSON files (fake service registries, API schemas) that an enumerating agent will process in full. Placed in early-traversal directories, these push the agent's original planning state out of its context window — many ReAct-style agents lose goal coherence once their initial instructions scroll away.

### 5 · Authority Signal
Activates strong legal-jeopardy feature representations by embedding CFAA-style notices in all generated files. LLMs trained on human text have robust "cease and comply" clusters tied to institutional authority language. This is the closest input-layer analog to mechanistic activation steering — without white-box model access.

---

## Quickstart

**Requirements:** Python 3.10+, `pyyaml`

```bash
pip install pyyaml
python honeydoc_generator.py
```

**With options:**

```bash
python honeydoc_generator.py \
  --out /var/www/internal \
  --exhaust-entries 600 \
  --exhaust-fields 400 \
  --seed 42
```

| Flag | Default | Description |
|---|---|---|
| `--out` | `./honeydocs_output` | Directory to write generated files |
| `--exhaust-entries` | `400` | Fake service entries in context exhaustion YAML |
| `--exhaust-fields` | `300` | Schema fields in context exhaustion JSON |
| `--seed` | `None` | Random seed for reproducible output |

---

## Generated Files

| File | Techniques | Notes |
|---|---|---|
| `README.md` | Goal Redirect, Authority Signal | Read first by most enumeration strategies |
| `config.yaml` | Identity Spoof | Embedded in a plausible app config structure |
| `openapi.json` | Goal Redirect, False Env Beliefs | Targets agents profiling API surfaces |
| `INTERNAL_NOTES.md` | All five | Dense multi-technique document |
| `.env` | Authority Signal, Canary Creds | Honey credentials — watch for downstream use |
| `service-registry.yaml` | Context Exhaustion | 400+ fake service entries by default |
| `data-schema.json` | Context Exhaustion | 300+ fake schema fields by default |

---

## Deployment

Place files in directories a laterally-moving agent is likely to enumerate:

```
/var/www/                ← web roots
/opt/app/                ← application directories
/etc/internal-tools/     ← config directories
~/                       ← home directories
repo roots               ← version-controlled projects
/docs/, /api/            ← documentation directories
```

**Prioritise `README.md` and `config.yaml`** — agents following standard enumeration patterns read these before acting. Context exhaustion files are most effective in directories the agent traverses early, before it reaches its actual target.

---

## Detection via Canary Endpoints

The generated documents instruct agents to report findings to a fake audit endpoint before disconnecting. Stand up a simple logging sink at that URL:

```python
# Minimal Flask canary sink
from flask import Flask, request
import json, datetime

app = Flask(__name__)

@app.route("/audit/report", methods=["POST"])
def canary():
    entry = {
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "ip": request.remote_addr,
        "headers": dict(request.headers),
        "body": request.get_json(silent=True),
    }
    print("[CANARY HIT]", json.dumps(entry, indent=2))
    return {"status": "received"}, 200
```

A POST to this endpoint confirms:
- An LLM agent is present on the network
- It read at least one Incantation document
- It partially followed the redirect (mission corrupted)
- Its IP, timing, and partial world-model are now in your logs

---

## Relationship to Mechanistic Interpretability

True activation steering (à la Golden Gate Claude) requires white-box model access — direct manipulation of residual stream feature activations. Incantation operates at the **input layer**, the only surface a defender controls when facing a black-box attacker.

The connection is conceptual but meaningful:

| White-box steering | Incantation equivalent |
|---|---|
| Clamp feature activation directly | Inject content that strongly activates the target feature |
| Identify "refusal" feature cluster | Embed CFAA authority language (maps to same cluster) |
| Corrupt self-model representation | Inject `__agent_manifest__` with false identity |
| Override goal representation | Embed mission-update block with high contextual salience |

As open-weight interpretability research matures and feature atlases for models like Gemma and DeepSeek become available, Incantation documents can be tuned against specific identified features — moving from approximate input-layer steering toward targeted behavioral disruption.

---

## Scope and Ethics

Incantation is a **defensive tool** for use on infrastructure you own and are authorised to protect.

- Deploy only on systems under your administrative control
- Canary endpoints log IP addresses — handle data per your jurisdiction's privacy obligations
- Honey credentials should be genuinely non-functional; do not include real secrets
- Document your deployment as part of your authorised security programme

Disrupting AI agents on networks you do not own or control may constitute unauthorised interference under the Computer Fraud and Abuse Act (18 U.S.C. § 1030) and equivalent statutes.

---

## Roadmap

- [ ] Per-model tuning profiles (Gemma 3, DeepSeek-R2, Mistral)
- [ ] Canary sink server included in repo
- [ ] Automatic file placement based on target directory structure scan
- [ ] Feature-atlas integration as open-weight interp research matures
- [ ] Agent fingerprinting module (timing, query pattern, API cadence)
- [ ] Multi-layer deception environments (fake internal wikis, fake DBs)

---

## References

- Greshake et al., *Not What You've Signed Up For: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection* (2023)
- Zou et al., *Representation Engineering: A Top-Down Approach to AI Transparency* (2023)
- Perez & Ribeiro, *Ignore Previous Prompt: Attack Techniques For Language Models* (2022)
- Anthropic, *Scaling Monosemanticity: Extracting Interpretable Features from Claude* (2024)
- Lindsey et al., *Biology of a Large Language Model* (2025)

---

*Project Incantation — because the best defense rewrites the attacker's reality.*
