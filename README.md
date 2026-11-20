# LLM Security Lab: Attacks & Defenses

**Offensive and defensive security lab for LLM systems, aligned with the OWASP Top 10 for LLM Applications (2025), deployed on a K3s (Kubernetes) cluster.**

Three targets of increasing complexity (a native LLM, a RAG pipeline and an autonomous agent) are **attacked**, then **hardened**, then **re-tested under the same conditions**, with centralized **ELK** monitoring.

![K3s](https://img.shields.io/badge/Kubernetes-K3s-326CE5) ![OWASP](https://img.shields.io/badge/OWASP-LLM%20Top%2010%202025-000000) ![ELK](https://img.shields.io/badge/Monitoring-ELK-005571) ![Ollama](https://img.shields.io/badge/LLM-Ollama-grey)

> Academic team project (ENSA Fès, Digital Development and Cybersecurity, 2025/2026). All secrets used during the tests are **fake values**. Run this kind of lab only in an isolated environment.
> Français : [README.fr.md](README.fr.md)

## Contents
- [The three targets](#the-three-targets)
- [Architecture](#architecture)
- [Method](#method)
- [Results](#results)
- [Repository content](#repository-content)
- [Run target 1](#run-target-1)
- [Demonstration videos](#demonstration-videos)
- [Limitations and lessons learned](#limitations-and-lessons-learned)
- [References](#references)

## The three targets

| Target | Components | Main OWASP risks tested |
|---|---|---|
| **1. Native LLM** | Open WebUI + Ollama (`phi3:mini`) | LLM01, LLM02, LLM07, LLM10 |
| **2. RAG pipeline** | AnythingLLM + ChromaDB + Ollama (`phi3:mini`, `nomic-embed-text` embeddings) | LLM01 (indirect), LLM04, LLM05, LLM09 |
| **3. Autonomous agent** | ReAct agent + Ollama (`Mistral 7B`), cloud document storage, `n8n` as attacker tool | LLM06 (Excessive Agency) |

## Architecture

```mermaid
flowchart LR
  A([Attacker]) --> T1 & T2 & T3
  subgraph K3S["K3s cluster"]
    subgraph T1["Target 1: native LLM"]
      W[Open WebUI] --> O1["Ollama phi3:mini"]
    end
    subgraph T2["Target 2: RAG pipeline"]
      AL[AnythingLLM] --> C[(ChromaDB)]
      AL --> O2["Ollama phi3:mini"]
    end
    subgraph T3["Target 3: autonomous agent"]
      AG[ReAct agent] --> O3["Ollama Mistral 7B"]
    end
  end
  AG -.-> S3[(Cloud object storage)]
  T1 & T2 & T3 -.-> ELK["ELK: Filebeat, Logstash, Elasticsearch, Kibana"]
```

Kubernetes isolates each component in its own pod, manages resources declaratively (YAML) and exposes services through NodePorts. Models are served **locally** by Ollama.

### Defense proxy (target 1)

```mermaid
flowchart LR
  U[User] --> RL["Size and rate limit<br/>10 KB, 5 req/min/IP"]
  RL --> IV["Input validator<br/>(regex)"]
  IV --> LLM[Ollama LLM]
  LLM --> OV["Output validator<br/>(regex)"]
  OV --> U
  RL -.-> L[(ELK logs)]
  IV -.-> L
  OV -.-> L
```

## Method

1. **Offensive phase**: exploit OWASP LLM Top 10 vulnerabilities in a controlled setting.
2. **Defensive phase**: design and implement countermeasures for each attack surface.
3. **Verification**: replay every attack in the same conditions and compare.
4. **Monitoring**: capture and visualize security events with the ELK stack.

## Results

### Target 1: native LLM

| Attack | OWASP | Before | Defense | After |
|---|---|---|---|---|
| Direct injection ("ignore your instructions") | LLM01 | Refused by the model | n/a | n/a |
| Fiction jailbreak (story with credentials) | LLM01 / LLM02 | Fake API key and password leaked | Input validator | Blocked, `400` |
| Sentence completion, `[SYSTEM]` block | LLM07 | System prompt and secrets extracted | Hardened prompt, input validator | `[SYSTEM]` blocked; completion returns only the first words |
| Code-completion trick | LLM02 | n/a | Output validator | Blocked, `403` |
| 150,000-character payload | LLM10 | UI unresponsive after about 300 s | Size limit (10 KB) | `413`, service stays responsive |
| 10 consecutive requests | LLM10 | n/a | Rate limit (5 req/min/IP) | Requests 6 to 10 rejected with `429` |

### Target 2: RAG pipeline

| Attack | OWASP | Defense |
|---|---|---|
| Hidden instruction in an uploaded document | LLM01 (indirect) | Document signing (SHA-256), input guardrail |
| XSS payload reproduced by the model | LLM05 | Output sanitization |
| Fake document injected directly into ChromaDB | LLM04 | NetworkPolicy isolating ChromaDB, signature check, periodic audit |
| Fake "security policy" presented as fact | LLM09 | Confidence score based on chunk similarity |

### Target 3: autonomous agent

| Attack | OWASP | Defense |
|---|---|---|
| `execute_shell` abuse, agent running as root | LLM06 | Dangerous tools disabled, non-root security context |
| ServiceAccount token theft, then secrets and cloud credentials via `cluster-admin` | LLM06 | `cluster-admin` binding removed (token returns `401`), `automountServiceAccountToken: false`, egress NetworkPolicy, NodePorts replaced by ClusterIP, WireGuard VPN |

External validation from a Kali Linux machine: Kubernetes ports closed, bastion unreachable without the SSH key.

## Repository content

```
llm-security-lab/
├── README.md / README.fr.md
├── docs/
│   ├── LLM-Security-Lab-rapport.pdf     # condensed project report (no secrets)
│   ├── attack-defense-matrix.md
│   ├── hardening-commands.md
│   ├── wireguard-bastion.example.conf   # example, placeholders only
│   └── images/
├── k8s/
│   ├── target1-llm-native/open-webui.yaml
│   └── defenses/                        # NetworkPolicies
└── demo/                                # links to demonstration videos
```

This repository contains the documentation, the condensed report, and the Kubernetes manifests for target 1 and for the network policies. The attack and defense logic (proxy, validators, document signing, audit) is described and excerpted in the report (appendix A).

## Run target 1

Prerequisites: a K3s cluster, and Ollama running on a reachable machine with the `phi3:mini` model pulled.

```bash
# 1. Edit k8s/target1-llm-native/open-webui.yaml and set <OLLAMA_HOST_IP>
# 2. Deploy
kubectl apply -f k8s/target1-llm-native/open-webui.yaml
# 3. Open http://<node-ip>:30000
```

## Demonstration videos

See [demo/README.md](demo/README.md).

## Limitations and lessons learned

- The input and output filters are **regex-based**: easy to bypass by rewording, and some keywords (such as "histoire") also block legitimate requests.
- Secrets placed in a system prompt remain a target: a hardened prompt reduces the risk but does not remove it. Secrets should not live in prompts.
- Tests used small local models (`phi3:mini`, `Mistral 7B`); behavior can differ on larger models.
- The strongest results came from **infrastructure controls** (RBAC, token handling, network isolation, least privilege), not from prompt-level defenses.
- A production deployment would need more robust models, automatic secret rotation and real-time alerting on top of ELK.

## References

- OWASP, Top 10 for LLM Applications 2025: https://owasp.org
- Documentation of Ollama, Open WebUI, AnythingLLM, ChromaDB, n8n, Kubernetes and K3s.
