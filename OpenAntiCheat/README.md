# Open Anti-Cheat (OAC)

Open Anti-Cheat (OAC) is a community-driven, open standard for fair multiplayer gaming.  
It protects players from cheaters **without kernel drivers, spyware, or added latency**.

## Why OAC?

Traditional anti-cheat systems are black boxes:
- Kernel-level drivers that compromise security and stability.  
- Silent patches that sweep exploits under the rug.  
- Little transparency, little accountability, limited cross-platform support.  

**OAC flips the model:**
- Open source & auditable — no black boxes.  
- Cross-platform — Windows, Linux, macOS, Steam Deck.  
- Lightweight — minimal CPU/network cost, no invasive hooks.  
- Modular — plug-in architecture for rules, challenges, and attestation.  
- Evidence-first — verifiable bundles for bans and appeals.  

## Current Draft

The full v0.1-DRAFT specification is available here:  
➡️ [specifications.md](specifications.md)

Highlights include:
- Pluggable client/server module interfaces.  
- Cryptographic challenge–response protocol.  
- Baseline anomaly heuristics (rules-based).  
- **Optional AI-assisted anomaly module** for post-match analysis.  
- Privacy-first posture: no process scans, screenshots, or invasive drivers.  

## Roadmap

- **v0.1** — Baseline rules + transport checks (done, draft available).  
- **v0.2** — Optional TPM/IMA attestation and trust tiers.  
- **v0.3** — ML anomaly spec under the same pluggable interface.  
- **v1.0** — Reference implementations + compliance tests.  

## Get Involved

This project is designed to become an **open standard**, not a closed product.  
Indie developers, researchers, and players are welcome to contribute, audit, and refine.  

Together, we can make anti-cheat something players trust.
