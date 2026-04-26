# Hermes Agent + RTX 3090 — Cookbook

> Retour d'expérience reproductible : faire tourner [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research, MIT) avec [Ollama](https://ollama.com/) en local sur **RTX 3090 24 GB** pour piloter [Home Assistant](https://www.home-assistant.io/) via MCP.

🇫🇷 **README en français pour le moment.** English translation coming later (after a thorough technical review pass).

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Hermes Agent v0.11.0+](https://img.shields.io/badge/Hermes%20Agent-v0.11.0%2B-blue.svg)](https://github.com/NousResearch/hermes-agent)
[![Ollama](https://img.shields.io/badge/runtime-Ollama-black.svg)](https://ollama.com/)
[![GPU: RTX 3090](https://img.shields.io/badge/GPU-RTX%203090%2024GB-green.svg)](https://www.nvidia.com/)

> 📅 **Publié sur GitHub le 26/04/2026** (commit racine `de7e268`, 7 fichiers, 1286 lignes, License MIT, contenu FR caviardé). Procédure d'init repo public + garde-fous email no-reply documentés dans [`docs/publication-s64.md`](docs/publication-s64.md).

---

## TL;DR

J'ai testé **5 modèles locaux différents** sous Hermes Agent + Ollama sur une RTX 3090 24 GB pour piloter Home Assistant via MCP. **Tous KO** au premier round (fin avril 2026). Le verdict transversal apparent était : *« le tool calling local est cassé sur RTX 3090, il faut un cloud LLM ou plus de VRAM »*.

C'était faux. La cause racine était un **bug `has_reasoning guard` dans Hermes Agent** lui-même, pas les modèles. Un `hermes update` (131 commits) a tout réglé en une étape :

| Métrique (Qwen 3.5 27B Q5, scénario identique) | Avant fix | Après fix | Variation |
|---|---|---|---|
| Latence Test action HA (create notification) | **20 m 01 s** | **5 m 06 s** | **−75 %** |
| Latence Test lecture multi-sensors | timeout / dérive | **2 m 40 s** | OK |
| Latence Test multi-step (read → condition → write) | échec | **2 m 57 s** | OK |
| Tool calls réussis | 0/3 | 3/3 | OK |

**Conclusion** : RTX 3090 24 GB suffit largement pour Hermes Agent + Ollama + MCP Home Assistant en mode pilotage non-interactif (Mode Réactif, automation 1 alerte/jour). **Pas besoin** de hardware upgrade à 2400 €. **Pas besoin** non plus de basculer sur un cloud LLM payant pour le cas d'usage.

---

## À qui s'adresse ce cookbook ?

- Tu as une **RTX 3090 24 GB** (ou GPU équivalent 24 GB VRAM) et tu veux faire tourner un agent LLM local avec tool calling.
- Tu utilises ou veux utiliser [**Hermes Agent**](https://github.com/NousResearch/hermes-agent) (Nous Research, MIT, sortie 02/2026) comme harnais d'orchestration multi-LLM.
- Ton backend d'inférence est **Ollama** (gguf, quantisations Q3/Q4/Q5).
- Tu veux brancher un agent local sur **MCP Home Assistant** (add-on `homeassistant-ai/ha-mcp` ou équivalent).
- Tu cherches **du concret** : configs reproductibles, latences mesurées, pièges, troubleshooting, pas une démo marketing.

---

## Stack validée

| Composant | Version / Modèle | Note |
|---|---|---|
| GPU | NVIDIA RTX 3090 24 GB | Offload CPU 20 %/80 % nécessaire pour 27B Q5 + ctx 65k |
| CPU | Intel i9-9900K (8 cœurs) ou équivalent | Compense l'offload sur les couches qui ne tiennent pas en VRAM |
| RAM | 32 GB DDR4 | Suffisant. Pour 70B Q3 → besoin de `.wslconfig` à 28 GB (voir [docs/configs.md](docs/configs.md)) |
| OS | Windows 11 Pro + WSL2 Ubuntu | Hermès et Ollama tournent dans WSL2 |
| Hermes Agent | v0.11.0 (commit ≥ `087e74d4`, post-26/04/2026) | Inclut le fix critique `reasoning_content` |
| Ollama | dernière version stable | `ollama serve` doit être actif |
| Modèle principal | `qwen3.5:27b` (Q5_K_M) + Modelfile custom `num_ctx 65536` | Validé sur 3 scénarios variés |
| MCP Home Assistant | add-on `homeassistant-ai/ha-mcp` v7.3.0+ | Avec ou sans `enable_tool_search`, voir [docs/troubleshooting.md](docs/troubleshooting.md) |

---

## Latences mesurées (post-fix)

Mesurées avec Hermes Agent en interactif (pas batch), MCP `ha-mcp` exposé avec **87 outils en clair** (`enable_tool_search: false`), Modelfile durci, num_ctx 65536, RTX 3090 24 GB + offload CPU 20/80 :

| Scénario | Tool calls | Latence | Sémantique |
|---|---|---|---|
| Action simple (create notification HA) | 1 | **5 m 06 s** | OK direct |
| Lecture multi-sensors + comparaison sémantique | 1 | **2 m 40 s** | Excellent — choisit le bon sensor parmi 3 candidats |
| Multi-step (lecture → condition `>22 °C` → notification) | 2 | **2 m 57 s** | Parfait — cite la valeur, évalue la condition, agit conditionnellement |

**Verdict** : viable pour **Mode Réactif** (1 alerte/jour, 60 s à 5 min acceptable). Pas adapté à un usage interactif synchrone (chat humain). Pour ce dernier cas, déléguer à un cloud LLM (Claude Haiku, GPT-4o-mini) reste pertinent.

---

## Index de la documentation

- 📜 **[docs/journey-s57-s63.md](docs/journey-s57-s63.md)** — Récit chronologique : 5 modèles testés, 5 KO, audit méthodologique 2, fix `reasoning_content`, validation. **Commence ici si tu veux comprendre le problème de fond.**
- 🔍 **[docs/audit-methodologique.md](docs/audit-methodologique.md)** — Pattern méthodologique réutilisable « audit niveau 2 » à appliquer dès **5+ modèles consécutifs KO**. Plan en 6 phases avec heuristique de stop. **Commence ici si tu rencontres une série de KO et veux une démarche structurée.**
- 🛠️ **[docs/troubleshooting.md](docs/troubleshooting.md)** — Catalogue symptôme → cause → fix : `has_reasoning guard`, `Model returned empty after tool calls`, dérive 27-32B Q5, KV cache OOM, template chat cassé sur certains modèles, latence Llama 3.3 70B Q3 inutilisable, etc. **Commence ici si tu as un bug précis à résoudre.**
- ⚙️ **[docs/configs.md](docs/configs.md)** — Configs reproductibles : Modelfile Qwen 3.5 durci, extrait `~/.hermes/config.yaml`, `.wslconfig` 28 GB pour 70B Q3, paramètres add-on `ha-mcp`. **Commence ici si tu veux copier-coller pour reproduire.**

---

## Reproductibilité — démarrage rapide

> ⚠️ Ce repo documente une expérience, ce n'est pas un installateur clé en main. Suis la documentation officielle Hermes Agent + Ollama + ha-mcp pour l'installation. Ce cookbook complète, ne remplace pas.

1. **Mets Hermès à jour avant tout test** : `hermes update` (le fix `reasoning_content` est le bloqueur le plus probable de tes problèmes de tool calling). Vérifie que `hermes --version` affiche `Up to date` et un hash upstream récent dans le titre TUI.
2. **Crée le Modelfile durci Qwen 3.5 27B** : voir [docs/configs.md](docs/configs.md). `ollama create qwen35-agent -f ~/Modelfile.qwen35-agent`.
3. **Configure `~/.hermes/config.yaml`** avec `provider: custom` + `default: qwen35-agent` + `context_length: 65536`. Voir [docs/configs.md](docs/configs.md).
4. **Branche le MCP Home Assistant** (add-on `homeassistant-ai/ha-mcp` recommandé, alternatives possibles).
5. **Teste 3 scénarios variés** : action simple, lecture, multi-step. Compare tes latences au tableau plus haut.
6. **Si KO** : ouvre [docs/troubleshooting.md](docs/troubleshooting.md), pas de panique, c'est probablement un détail de config.

---

## Ce que ce cookbook n'est PAS

- ❌ Pas un benchmark exhaustif (modèles, paramètres, hardware comparés méthodiquement). C'est un retour d'XP terrain.
- ❌ Pas une recommandation Hermes Agent vs alternatives (LangGraph, AutoGen, OpenHands, etc.). On a choisi Hermes Agent pour ses raisons (multi-LLM natif, MCP natif, fichier `AGENTS.md` style `CLAUDE.md`) — d'autres choix sont valides.
- ❌ Pas un tutoriel installation complète. La doc officielle Hermes/Ollama/ha-mcp couvre l'installation.
- ❌ Pas un repo maintenu activement avec issues/PRs traitées rapidement. C'est du *write-once-share-forever*. Les PRs sont bienvenues mais sans engagement de réponse.

---

## Crédits & inspirations

- [Nous Research](https://nousresearch.com/) — pour [Hermes Agent](https://github.com/NousResearch/hermes-agent) (MIT, sortie 02/2026).
- [Ollama](https://ollama.com/) — pour rendre l'inférence locale accessible.
- [`homeassistant-ai/ha-mcp`](https://github.com/homeassistant-ai/ha-mcp) — pour combler le bug DCR du `mcp_server` core de Home Assistant.
- Communauté `r/LocalLLaMA`, `r/homeassistant`, GitHub Issues Ollama (#11662, #14601, #11135, #14745, #14617) — pour avoir documenté les patterns de tool calling cassé qui ont guidé le diagnostic.

---

## Licence

[MIT](LICENSE) — réutilisation libre, mention bienvenue mais pas obligatoire.

## Contributions

Les *issues* avec retour d'expérience similaire (ou divergent !) sur d'autres GPU 24 GB (RTX 4090, A5000), d'autres backends (vLLM, llama.cpp), ou d'autres modèles (Hermes 4 70B Q3, Qwen 3 32B, Mistral Small 3) sont bienvenues. PR pour corriger des erreurs ou améliorer la doc également.
