# Open Anti-Cheat (OAC)

Open Anti-Cheat (OAC) is a community-driven, open standard for fair multiplayer gaming.  
It is designed to protect players from cheaters **without invasive kernel drivers, spyware, or added latency**.

## Why OAC?

Traditional anti-cheat is a black box:
- Ring-0 drivers that compromise security.
- Hidden patch cycles that sweep vulnerabilities under the rug.
- Limited cross-platform support (Linux, macOS, Steam Deck).

OAC flips the model:
- **Open source & auditable** — no black boxes.  
- **Cross-platform** — PC, Mac, Linux, Steam Deck.  
- **Lightweight** — no kernel hooks, minimal CPU/network overhead.  
- **Evidence-first** — strong proof bundles for bans and appeals.  
- **Modular** — plug-in architecture for rules, challenges, and attestation.  

## AI-Assisted Anomaly Detection

OAC supports pluggable AI modules to assist in post-match analysis.  
This helps catch sophisticated cheats that hide within human-like thresholds, while keeping false positives low.  
The AI layer is optional, explainable, and integrated only at the evidence stage.

## Repo Layout

- `README.md` — project overview (this file).  
- `specifications.md` — the full v0.1 draft spec, including baseline heuristics and AI-assisted anomaly module.  

## Get Involved

OAC is intended as an **open standard** for the gaming community.  
Indie developers, researchers, and players are encouraged to contribute, test, and refine the approach.  

Together, we can build an anti-cheat system that players actually trust.
