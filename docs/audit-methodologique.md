# Pattern méthodologique : audit niveau 2

> **Quand appliquer ?** Lorsque tu as testé **5 modèles ou plus** sous Hermes Agent + Ollama avec des verdicts KO différents (dérive sémantique, hallucination de tool calls, JSON malformé, latence inacceptable, output vide après tool call, etc.) et que tu commences à conclure « le tool calling local est intrinsèquement cassé ».
>
> **Heuristique** : ce verdict transversal est presque toujours **faux**. Avant de claquer 2400 € en hardware ou de basculer définitivement sur un cloud LLM, dérouler ce plan en 6 phases.

## Pourquoi un pattern méthodologique « niveau 2 » ?

L'écosystème LLM agent (fin 2025 / début 2026) bouge vite : Hermes Agent a vu **environ 40 commits/jour** sur la période d'observation, Ollama a des Issues GitHub ouvertes sur le tool calling depuis des mois (#11662, #14601, #11135, #14745, #14617), les Modelfiles par défaut des modèles « reasoning » (Qwen 3, Qwen 3.5, DeepSeek-R1, Kimi) ont des comportements inattendus (`<think>` non désactivé, `reasoning_content` mal injecté).

Quand on enchaîne les KO sur des modèles différents, le réflexe naturel est de blâmer **le maillon qu'on contrôle le moins** (le modèle, le hardware). C'est rarement la bonne réponse. Les vraies causes racines sont presque toujours **dans la couche d'orchestration ou la config par défaut** :

- bug du harnais (Hermes Agent lui-même)
- bug Ollama API tool calling
- Modelfile par défaut inadapté au cas d'usage agent
- catalogue MCP trop large pour le modèle (>50 outils ⇒ hallucination)
- mode `<think>` actif qui pollue le tool calling
- contexte trop court vs taille du prompt système (87 outils en clair, c'est ~46 K tokens)

L'audit niveau 2 force à **investiguer ces couches en priorité**, dans un ordre coût/probabilité décroissant.

## Le plan en 6 phases

| # | Action | Temps | Probabilité résolve | Coût |
|---|---|---|---|---|
| 1 | `hermes update` + audit `git log` ciblé tool calling / Ollama / provider | ~5 min | 🔴 **HAUTE** | gratuit |
| 2 | Vérifier qu'on utilise le provider `ollama` natif Hermès (vs `custom` + base_url) | ~10 min | 🟠 Moyenne-Haute | gratuit |
| 3 | Tester `tools.include` / `tools.exclude` ciblé (10 outils MCP précis au lieu de 87) | ~20 min | 🟠 Moyenne | gratuit |
| 4 | Recherche communautaire Reddit + GitHub Issues ciblée (Hermès + Ollama + ta taille de modèle) | ~15 min | 🟠 Moyenne | gratuit |
| 5 | Tester un scénario simple isolant (ex. lis un état de sensor seul, sans multi-step, sans condition) | ~10 min | 🟡 Moyenne | gratuit |
| 6 | Modèle « MAISON » Hermes 4 70B HuggingFace (dernier recours, offload CPU obligatoire 70B Q3 sur 24 GB) | ~60-90 min | 🟡 Faible | offload CPU lourd |

### Heuristique de stop

**Si la phase 1 résout le problème, SKIP les phases 2 à 5.** C'est exactement ce qui s'est passé lors de la validation du pattern (voir [`journey-s57-s63.md`](journey-s57-s63.md) §S63) : un commit `fix: remove has_reasoning guard` ingéré via `hermes update` a débloqué tous les modèles, validation 3 scénarios variés, audit terminé.

Pourquoi cet ordre :

1. **Phase 1 — coût marginal (5 min) avec proba HAUTE → toujours commencer là.** Le fait que ça soit gratuit et rapide, combiné à la fréquence des fixes upstream sur Hermès (~40 commits/jour), rend cette phase quasi obligatoire avant tout autre debug.
2. **Phase 2** vérifie une hypothèse architecturale (provider `custom` qui réimplémente partiellement la logique vs provider `ollama` natif qui peut bénéficier de fixes upstream différents). À faire avant la phase 3 car la config `tools.include` suppose que l'architecture est déjà OK.
3. **Phase 3** réduit la surface du bug. Si le tool calling marche avec 10 outils mais pas 87, le problème est lié au catalogue (token budget, hallucination sur grand espace de tools), pas au modèle.
4. **Phase 4** = pattern d'audit communautaire (niveau 1, voir hiérarchie ci-dessous). Utile si les phases 1-3 n'ont pas résolu mais ne remplace **pas** la phase 1.
5. **Phase 5** isole l'hypothèse complexité scénario. Si « create notification » échoue mais « lis état d'un sensor » marche, le problème est dans le multi-step ou l'écriture, pas dans le tool calling lui-même.
6. **Phase 6** est une montée en puissance modèle, **coûteuse** car offload CPU obligatoire pour 70B Q3 sur 24 GB. À ne pas faire avant d'avoir validé que les modèles plus petits ne marchent pas pour des raisons modèle (et non config).

## Hiérarchie des patterns d'audit

| Niveau | Nom | Quand l'appliquer | Référence |
|---|---|---|---|
| **1** | Audit communautaire avant verdict | Dès **1-2 KO** : ne pas éliminer un modèle après 1-2 tests sans patcher la config et croiser avec issues GitHub upstream | Origine méthodologique, à appliquer systématiquement |
| **2** | Audit méthodologique 2 *(ce document)* | Dès **5+ KO consécutifs** sur Phase B Hermès local | Validé par expérience |
| **3** | Audit hardware / OS / driver *(spéculatif, jamais déclenché)* | Si phases 1-6 du niveau 2 sont KO ensemble | À documenter le jour où ça arrive |

L'idée : monter en puissance graduellement. Sauter directement au niveau 3 (« upgrade hardware ») dès qu'on a 5 KO est l'erreur classique qui coûte cher. Le niveau 2 doit être épuisé d'abord.

## Quand NE PAS appliquer ce pattern

- Moins de 3 modèles testés → utilise plutôt le pattern niveau 1 (audit communautaire ciblé sur le modèle qui pose problème).
- Tu travailles avec un LLM **propriétaire cloud** (Claude, GPT-4, Gemini) → la phase B Hermès local ne s'applique pas, et tes problèmes sont probablement liés aux clés API, rate limits, ou prompt engineering.
- Le symptôme est isolé sur **un seul modèle** sur 5 testés → debug ciblé sur ce modèle, pas audit transversal.
- Tu as **moins de 12 GB VRAM** → l'audit ne va pas changer le fait que tu ne pourras pas faire tourner les modèles 27B+ requis. Vise plus petit (Llama 3.1 8B, Mistral 7B, Qwen 2.5 7B) et accepte les limitations.

## Exemple concret d'application réussie

Voici comment l'audit a déroulé en conditions réelles, fin avril 2026 :

### État avant audit
- 5 modèles testés sous Hermes Agent v0.11.0 + Ollama + add-on Home Assistant MCP exposant 87 outils :
  - `mistral-nemo:12b` → KO (template chat cassé, inversion de rôle)
  - `llama3.3:70b-instruct-q3_K_M` → KO (JSON tool call dans `content`, latence 10 m 52 s/tour)
  - `qwen3:32b` → KO (« Model returned empty after tool calls »)
  - `qwen2.5-coder:32b` → KO (dérive sémantique)
  - `qwen3.5:27b` → KO (5 voies A→G testées, latence 20 min, dérive)
- Verdict transversal apparent : « le tool calling local est cassé sur RTX 3090 »
- Tentation : claquer 2400 € en upgrade hardware (Ryzen + plus de VRAM)

### Phase 1 du plan (5 min)

```bash
# Dans WSL2
hermes --version
# → Hermes Agent v0.11.0 (2026.4.23)
# → 131 commits behind upstream

hermes update
# → 131 commits ingérés

cd ~/.hermes/hermes-agent
git log --oneline -131 \
  | grep -iE "(tool|ollama|provider|mcp|function|empty|qwen|hermes|reasoning)" \
  | head -60
# → 29 commits matchés
# → 4 commits HAUTE priorité repérés (mots-clés "reasoning_content"):
#   5ae60815  fix: remove has_reasoning guard — inject empty reasoning_content for DeepSeek/Kimi tool_calls unconditionally
#   f93d4624  Merge PR #15749 — fix copy-reasoning-content-ordering and cross-provider-isolation
#   63bf7a29  fix(run_agent): prevent reasoning_content regression in DeepSeek/Kimi tool-call replay
#   9daa0620  fix(agent): ordering fix in _copy_reasoning_content_for_api — cross-provider reasoning isolation
```

### Décision
**Voie A (retest direct du modèle qui paraissait le moins KO, post-update)** plutôt que continuer la phase 2 d'audit doc. Asymétrie risque/bénéfice favorable : retest gratuit (modèle déjà installé), et les commits cités sont **explicitement** dans le périmètre du symptôme observé (`reasoning_content` injecté lors du tool calling).

### Validation post-update (15 min)

| Test | Scénario | Latence | Verdict |
|---|---|---|---|
| A | Action simple (create notification HA) | **5 m 06 s** (vs 20 m 01 s pré-update sur config strictement identique, **−75 %**) | ✅ |
| B | Lecture multi-sensors + comparaison sémantique | **2 m 40 s** | ✅ |
| C | Multi-step (lecture → condition `>22 °C` → notification) | **2 m 57 s** | ✅ |

**Conclusion** : phases 2-5 inutiles (skip). Phase 6 (Hermes 4 70B) inutile (Qwen 3.5 27B Q5 suffit).

### Verdict
- Hardware upgrade 2400 € : **NO-GO** (économie confirmée)
- Bascule cloud LLM Haiku définitive : **annulée** (local viable)
- Plan PDF v2 75 % local : **viable**

L'audit niveau 2 a remplacé une décision à 2400 € par 5 min de `hermes update`. C'est exactement le genre d'asymétrie qu'on cherche.

## Mots-clés à grepper dans `git log` Hermès

Quand tu fais l'audit `git log --oneline -<N> | grep -iE "..."`, voici les mots-clés à chercher en priorité (ordre de probabilité décroissante d'impact sur ton problème de tool calling) :

```regex
(reasoning_content|has_reasoning|reasoning regression|copy.reasoning)  # le plus haut signal
(tool.call|tool_call|tool_calls)
(provider|custom_provider|ollama)
(empty|empty.after|return.*empty)
(deepseek|kimi|qwen)
(replay|injection|guard)
(mcp|mcp_server|tools.include|tools.exclude)
```

Les commits qui matchent **plusieurs** de ces patterns sont des candidats sérieux. Si tu vois `fix:` ou `fix(`, c'est encore plus probable que ce soit pertinent pour ton symptôme.

## Pré-requis pour appliquer ce pattern

- WSL2 ou Linux natif avec accès `git` au repo Hermès cloné automatiquement par l'installation (`~/.hermes/hermes-agent/.git/`).
- Capacité à **rebuild** un Modelfile Ollama localement (`ollama create`).
- Patience : le retest post-update prend ~15 min pour 3 scénarios sur un modèle 27B Q5 + offload CPU.
- Bénéfice secondaire : tu apprends à lire un `git log` ciblé sur un repo upstream, compétence transposable à beaucoup d'autres projets en dev intense.

## Pour aller plus loin

- [`troubleshooting.md`](troubleshooting.md) — symptômes spécifiques et leur fix
- [`journey-s57-s63.md`](journey-s57-s63.md) — récit complet de l'application du pattern
- [`configs.md`](configs.md) — configs reproductibles validées post-update
