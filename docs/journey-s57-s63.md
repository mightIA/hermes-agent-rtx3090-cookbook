# Journey — 5 modèles, 5 verdicts KO, un fix qui débloque tout

> Récit chronologique de l'application de l'audit méthodologique 2 sur la stack **Hermes Agent + Ollama + RTX 3090 24 GB + MCP Home Assistant**, fin avril 2026. Sept sessions de travail (numérotées S57 à S63 ci-dessous, sans S61) pour passer de « 5 modèles KO, peut-être faut-il upgrader le GPU » à « `hermes update` règle tout, Qwen 3.5 27B Q5 marche très bien ».
>
> Ce document est le **récit**. Pour la méthodologie réutilisable, voir [`audit-methodologique.md`](audit-methodologique.md). Pour les fixes ciblés, voir [`troubleshooting.md`](troubleshooting.md). Pour les configs, voir [`configs.md`](configs.md).

## Contexte

L'objectif de l'agent local : piloter Home Assistant via MCP (lectures de capteurs, déclenchement de scripts, écritures d'état, notifications) en **complément** d'un LLM cloud (utilisé pour les tâches interactives complexes). Le cas d'usage principal était un **« Mode Réactif »** : 1-3 alertes par jour à classifier et router, latence acceptable 60 s à 5 min/alerte.

Hardware utilisé tout au long :
- NVIDIA RTX 3090 24 GB
- Intel i9-9900K (8 cœurs)
- 32 GB RAM DDR4
- Windows 11 Pro + WSL2 Ubuntu
- Hermes Agent v0.11.0 (premier lancement le 23/04/2026)
- Ollama dernière version stable
- Add-on Home Assistant `homeassistant-ai/ha-mcp` v7.3.0+

## S57 — Qwen 3:32b « stabilisé » via Modelfile durci (audit niveau 1)

### État de départ
Premier modèle local testé sérieusement : `qwen3:32b` (Q4, ~20 GB). Le tool calling sous Hermès était cassé avec un symptôme étrange : **« Model returned empty after tool calls — nudging to continue »** suivi d'une boucle qui consommait le contexte sans réponse utile.

### Investigation (audit communautaire — niveau 1)
Recherche méthodique sur GitHub Issues Ollama et r/LocalLLaMA. Cinq issues ouvertes documentent **exactement** le même symptôme :
- [Issue #11662](https://github.com/ollama/ollama/issues/11662) — Qwen3 tool calling broken
- [Issue #14601](https://github.com/ollama/ollama/issues/14601) — Tool calling /api/chat malformed
- [Issue #11135](https://github.com/ollama/ollama/issues/11135)
- [Issue #14745](https://github.com/ollama/ollama/issues/14745)
- [Issue #14617](https://github.com/ollama/ollama/issues/14617) — Disable thinking via Modelfile

Le pattern commun : le mode reasoning natif Qwen 3 (`<think>` blocks) interfère avec le format tool calling Ollama, et Hermès ne sait pas gérer correctement la réinjection du `reasoning_content`.

### Fix appliqué (Modelfile durci)

```Modelfile
FROM qwen3:32b
PARAMETER num_ctx 32768
PARAMETER temperature 0.3
PARAMETER top_p 0.9
PARAMETER top_k 20
PARAMETER repeat_penalty 1.05
SYSTEM """You are <agent>, the master agent.
Tool calls MUST be emitted as <tool_call>{...}</tool_call>
Do NOT narrate before calling a tool.
Do NOT use <think> blocks before tool calls.
"""
```

Voir [`configs.md` §2](configs.md#2-modelfile-durci--qwen-3x-fallback) pour le détail.

### Résultat S57
- **Lecture HA via MCP** : ✅ fonctionne (test `ha_search_entities` → 5 entités vraies retournées avec états réels)
- **Écriture HA via MCP** : ⚠️ partiellement
  - Le tool call structuré est émis (plus de « empty after tool calls »)
  - Mais la précision sémantique est dégradée : pour la demande « crée une notification persistante », le modèle a généré un **script HA YAML** au lieu d'appeler directement le service `persistent_notification.create`

### Verdict S57
- Audit communautaire niveau 1 = **utile mais insuffisant**.
- Le Modelfile durci corrige la **fiabilité** du tool calling (boucle nudge, format), pas la **précision sémantique**.
- Hypothèse posée : il faut un modèle plus capable pour fiabiliser l'écriture. À tester en S58.

### Leçon retenue
**Toujours croiser un verdict KO avec les Issues GitHub upstream avant d'éliminer un modèle.** Ce sera le pattern d'audit niveau 1 (cf. [`audit-methodologique.md`](audit-methodologique.md#hiérarchie-des-patterns-daudit) §Hiérarchie).

## S58 — Phase B étapes 1 et 2 (mistral-nemo + Llama 3.3 70B Q3)

### Étape 1 — `mistral-nemo:12b`

Hypothèse : modèle plus petit (7 GB), plus rapide, et avec format `[TOOL_CALLS]` natif. Pourrait corriger l'imprécision sémantique de Qwen 3.

Test scénario notification persistante via ha-mcp.

**Résultats observés** :
- **Inversion de rôle** : le modèle répond comme s'il était l'utilisateur (*« Bonjour, pouvez-vous m'aider à installer Node.js ? »*). Aucun tool call émis.
- **SYSTEM ignoré** : test discriminant « qui es-tu ? » → réponse générique d'agent CLI, pas Jarvis.
- **Mélange linguistique** mineur (mots EN dans réponse FR).

**Cause racine identifiée** : template chat Mistral mal mappé sous Ollama quand SYSTEM non trivial + catalogue tools simultanés. Famille bug Ollama #11662 / #14601.

**Verdict** : `mistral-nemo:12b` **KO comme moteur principal**. Reste OK comme `auxiliary.compression.model` Hermès.

### Étape 2 — `llama3.3:70b-instruct-q3_K_M` (~34 GB)

Hypothèse : modèle plus gros, devrait gérer mieux les enchaînements multi-step. Quantisation Q3 pour rentrer en hardware (offload CPU obligatoire sur RTX 3090 24 GB).

#### Pré-requis — `.wslconfig`

Premier essai : Ollama refuse avec **`HTTP 500: model requires more system memory (20.2 GiB) than is available (18.2 GiB)`**.

Cause : WSL2 alloue **50 % de la RAM Windows par défaut** = ~15 Gi seulement sur 32 GB physiques. Le calcul Ollama inclut les 10 GB de couches offloadées CPU + KV cache + overhead.

Fix : `C:\Users\<user>\.wslconfig` :
```ini
[wsl2]
memory=28GB
swap=8GB
```
puis `wsl --shutdown` depuis PowerShell. Voir [`configs.md` §4](configs.md#4-wsl2--wslconfig-28-gb-pour-70b-q3).

#### Test scénario notification persistante (post-fix RAM)

**Résultats observés** :
- **Tool call émis dans le `content`** comme texte JSON brut (au lieu d'un appel structuré reconnu par Hermès).
- **JSON malformé** : mélange de `name` et `arguments` de deux outils différents.
- **Narration explicite** (« Pour appeler l'outil, voici la commande : ») qui viole les règles SYSTEM Modelfile durci.
- **Latence catastrophique** : **10 min 52 s par tour** pour ~80 tokens (soit ~0.12 tok/s effectif).

**Cause racine identifiée** : template chat Llama 3.x sous Ollama mal câblé pour le format tool calling natif Llama (`<|python_tag|>`). Plus offload CPU lourd.

**Verdict** : `llama3.3:70b-instruct-q3_K_M` **KO sévère**. Inutilisable même avec Mode Réactif relâché (latence > 5 min/tour).

### Bilan S58
Deux modèles testés, deux verdicts KO différents. Hypothèse en cours de formation : « tool calling local cassé sur RTX 3090 ».

## S60 — Tentative d'audit bugs Haiku (partiel)

Tentative parallèle de tester un cloud LLM (Anthropic Claude Haiku 4.5) via OpenRouter comme moteur Hermès, pour comparer. Phase d'exploration interrompue (problèmes de configuration OpenRouter, scope hors-cookbook).

**Insight retenu** : Hermès supporte bien les providers cloud OpenAI-compatible. C'est une voie de fallback si le local échoue. Mais on n'abandonne pas le local trop tôt.

## S62 — Qwen 3.5 27B Q5, voies A à G

Sixième modèle testé : `qwen3.5:27b` Q5 (~17 GB poids). Plus grand contexte natif que Qwen 3, meilleure qualité sémantique sur l'instruction, sortie début 2026.

### Voies testées (variantes de config)
Sept variantes (A→G) :
- Variantes de `num_ctx` (16k, 32k, 65k)
- Variantes de `enable_tool_search` côté add-on ha-mcp (ON / OFF)
- Variantes de SYSTEM Modelfile (minimal / durci)
- Variantes de `provider` Hermès (`custom` / native)

### Symptômes récurrents
- **Dérive sémantique** : « crée notification » → « crée script Python » ou « génère YAML d'automation »
- **Hallucination MCP** : `write_file /tmp/ha_notification.yaml` ou `curl -X POST http://homeassistant.local:8123/...` au lieu d'`ha_call_service`
- **Latence dégradée** : 10-20 min/tour avec `num_ctx 65536` et offload CPU 20/80
- **Pattern « empty after tool calls »** réapparaît malgré le Modelfile durci

### Piège ressorti
Pull `qwen3.5:27b` avec `num_ctx 65536` directement sans calculer la VRAM. Résultat : 28 GB total = offload CPU sur RTX 3090 24 GB → latence catastrophique. Voir [`configs.md` §6](configs.md#6-tableau-vram--num_ctx--kv-cache) pour la règle de calcul.

### Verdict S62
- Sept variantes testées, **toutes KO** avec des nuances.
- Total cumulé fin S62 : **5 modèles testés, 5 KO différents**.
- Verdict transversal apparent : « le tool calling local est cassé sur RTX 3090, il faut un cloud LLM ou plus de VRAM ».
- Tentation forte : valider l'upgrade hardware (~2400 € pour Ryzen 9 7950X + 64 GB DDR5 + carte mère + boîtier — pas la VRAM, mais on espérait que CPU + RAM aideraient).

## Le tournant — décider d'auditer **avant** d'acheter

À la fin de S62, deux options sur la table :

| Option | Coût | Probabilité de résoudre | Réversibilité |
|---|---|---|---|
| Upgrade hardware (CPU + RAM) | **~2400 €** | Faible (le bottleneck est la VRAM 24 GB, pas la RAM) | Faible (matériel acheté) |
| Audit méthodologique 2 (cf. [`audit-methodologique.md`](audit-methodologique.md)) | 0 € | Inconnue mais asymétrie favorable (5 min phase 1) | Totale |

L'option 2 a été retenue. La logique : **la phase 1 de l'audit (`hermes update`) coûte 5 minutes**. Si elle ne résout rien, on a juste perdu 5 min. Si elle résout, on a économisé 2400 €. C'est l'asymétrie risk/reward parfaite.

## S63 — `hermes update` règle tout

### Phase 1 — l'update

```bash
hermes --version
# → Hermes Agent v0.11.0 (2026.4.23)
# → 131 commits behind upstream
```

Cent trente-et-un commits derrière le tronc en quelques jours. Le projet Hermes Agent était sorti deux mois plus tôt (02/2026, Nous Research, MIT) et était en développement intense (~40 commits/jour observés).

```bash
hermes update
# → 131 commits ingérés, web UI rebuild, dépendances Python+Node updated
```

### Audit `git log` ciblé

```bash
cd ~/.hermes/hermes-agent
git log --oneline -131 \
  | grep -iE "(tool|ollama|provider|mcp|function|empty|qwen|hermes|reasoning)" \
  | head -60
# → 29 commits matchés
```

Quatre commits **HAUTE** priorité repérés (mots-clés `reasoning_content`) :

| Hash | Message |
|---|---|
| `5ae60815` | `fix: remove has_reasoning guard — inject empty reasoning_content for DeepSeek/Kimi tool_calls unconditionally` |
| `f93d4624` | `Merge PR #15749 — fix copy-reasoning-content-ordering and cross-provider-isolation` |
| `63bf7a29` | `fix(run_agent): prevent reasoning_content regression in DeepSeek/Kimi tool-call replay` |
| `9daa0620` | `fix(agent): ordering fix in _copy_reasoning_content_for_api — cross-provider reasoning isolation` |

Les messages sont **explicites** et collent au symptôme :
- `remove has_reasoning guard` → le guard cassé qui injectait/retirait mal le `reasoning_content`
- `for DeepSeek/Kimi tool_calls` → applicable aussi à Qwen 3.x (mode reasoning natif similaire)
- `prevent regression` → fix d'un fix précédent qui régresse
- `ordering fix in _copy_reasoning_content_for_api` → ordre de copie qui causait des doublons

### Décision : voie A — retest direct du modèle qui paraissait le moins KO

Plutôt que continuer la phase 2 d'audit doc (~60 min), retester directement `qwen3.5:27b` post-update (~25 min). Asymétrie favorable : retest gratuit, et les commits cités sont **précisément** dans le périmètre du symptôme observé.

### Préparation retest
- Modelfile : revenir à `num_ctx 65536` (V1 S62)
- Add-on ha-mcp : `enable_tool_search: false` (87 outils en clair)
- `~/.hermes/config.yaml` : provider `custom` + `default: <agent-name>` + `context_length: 65536`
- `ollama create <agent-name> -f ~/Modelfile` (rebuild ~2 s, blobs poids 17 GB réutilisés)

### Tests scénarios

| Test | Scénario | Tool calls | Latence | Sémantique | Verdict |
|---|---|---|---|---|---|
| **A** | Action simple : create notification HA persistante | 1 (`ha_call_service`) | **5 m 06 s** | OK direct | ✅ |
| **B** | Lecture multi-sensors + comparaison sémantique (recherche entité « température ») | 1 (`ha_search_entities`) | **2 m 40 s** | Excellent — compare 3 sensors trouvés et justifie le choix du sensor « cuisine » comme « température ambiante » | ✅ |
| **C** | Multi-step : lis temp → condition stricte `> 22 °C` → notification | 2 (read + write) | **2 m 57 s** | Parfait — cite la valeur exacte `22.12 °C`, évalue la condition explicitement, agit conditionnellement | ✅ |

### Comparatif S62 V1 vs S63 Test A (config strictement identique sauf Hermès updated)

| Indicateur | S62 V1 (pré-update) | S63 Test A (post-update) | Variation |
|---|---|---|---|
| Latence | 20 m 01 s | **5 m 06 s** | **−75 %** |
| Tool calls | 1 | 1 | identique |
| Tokens consommés | 53.1 K / 65.5 K (81 %) | 53.1 K / 65.5 K (81 %) | identique |
| Sémantique | OK | OK | identique |
| Réponse FR | OK | OK + format puces | identique++ |
| Dérive | aucune | aucune | identique |
| VRAM | 28 GB offload CPU 20/80 | 28 GB offload CPU 20/80 | identique (hardware non modifié) |

**Seul changement** : update Hermès. Pas de patch Modelfile, pas de patch config, pas de patch hardware. C'est un signal fort que la cause racine était bien dans le harnais d'orchestration, pas dans les modèles ou le hardware.

### Verdict S63 (= verdict global)

| Sujet | Verdict |
|---|---|
| Hypothèse #1 (Hermès commits behind = fix critique) | ✅ **VALIDÉE** |
| Hypothèses #2-#7 audit méthodologique 2 | ⚪ obsolètes (rendues sans objet par #1) |
| Phases 2-5 audit | ⏭️ skip |
| Phase 6 (Hermes 4 70B) | ⏭️ skip (Qwen 3.5 27B Q5 suffit) |
| Hardware upgrade ~2400 € | 🟢 **NO-GO confirmé** |
| Bascule cloud LLM (Haiku) définitive | 🔴 **annulée** — local viable |
| Plan PDF v2 75 % local | ✅ **viable** |

## Leçons apprises

### Méthodologie
- **Pattern niveau 1 (audit communautaire)** : utile dès le 1er KO. Sans S57, on n'aurait pas identifié les Issues Ollama. Mais insuffisant seul (Modelfile durci ne traitait pas la cause racine).
- **Pattern niveau 2 (audit méthodologique 2)** : devient indispensable dès **5+ KO consécutifs** sur la même stack. La phase 1 (`hermes update`) doit toujours être tentée en premier — coût marginal, probabilité haute, surtout sur des projets jeunes en dev intense.
- **Heuristique d'asymétrie risk/reward** : si l'option « bricolage » coûte 5 min et l'option « hardware » coûte 2400 €, toujours essayer la première d'abord, même si la probabilité paraît faible.

### Technique
- **Bug `has_reasoning guard`** Hermès était transversal à tous les modèles avec mode reasoning natif (Qwen 3.x, DeepSeek-R1, Kimi). Un seul fix débloque cinq verdicts KO.
- **Modelfile durci** était une rustine côté client utile mais pas curative. À garder comme **fallback** si le bug réapparaît dans une régression future.
- **`enable_tool_search`** reste un compromis : ON pour modèles ≤ 14B, OFF pour ≥ 27B Q4-Q5 capables de gérer 87 outils en clair.
- **Calcul VRAM/KV cache** doit précéder tout pull de modèle 27-32B Q5. La règle `num_ctx × 0.0001875 GB` permet de prévoir l'offload CPU avant d'investir 30-90 min de tests.

### Mentale
- **Les verdicts transversaux apparents (« le tool calling local est cassé ») sont presque toujours faux.** Ils blâment le maillon qu'on contrôle le moins (modèle, hardware) au lieu de la couche d'orchestration où le bug est statistiquement plus probable.
- **Un projet en dev intense (~40 commits/jour) doit être maintenu à jour avant tout debug.** `hermes update` aurait dû être la première action, pas la dernière.
- **Les commits avec mots-clés `fix:`, `regression`, `replay`, `guard`, `ordering`** sont des signaux forts à grepper en priorité dans un `git log` upstream récent.

## Ce que l'on a évité

- 2400 € de hardware upgrade (Ryzen + RAM + carte mère + boîtier) qui n'aurait probablement rien réglé (le bottleneck était logiciel, pas hardware).
- Bascule définitive sur un cloud LLM payant pour le cas d'usage Mode Réactif, alors que le local couvre largement.
- Conclusion erronée « le tool calling local n'est pas mature en 2026 », qui aurait orienté la roadmap dans la mauvaise direction.

## Ce qu'il reste à faire (P3 — optionnel)

- **Retests post-update** des 4 modèles antérieurs (`mistral-nemo:12b`, `llama3.3:70b-instruct-q3_K_M`, `qwen3:32b`, `qwen2.5-coder:32b`). Critère = latence vs 2 m 40 s `qwen3.5:27b`.
- **Optimisation latence Mode Réactif** : tester `num_ctx 32k` (KV cache 6 GB → 100 % GPU possible sur 27B Q5). Cible < 90 s/tour. Voir [`configs.md` §6](configs.md#6-tableau-vram--num_ctx--kv-cache).
- **Test Hermes 4 70B Q3** (modèle « MAISON » Nous Research, HuggingFace) en bonus comparatif qualité.
- **Retest `enable_tool_search: true` post-update** : la dérive 27-32B Q5 observée était-elle uniquement due au bug `has_reasoning guard` ?

## Pour aller plus loin

- [`audit-methodologique.md`](audit-methodologique.md) — pattern réutilisable
- [`troubleshooting.md`](troubleshooting.md) — symptômes / fixes
- [`configs.md`](configs.md) — configs reproductibles
- [README](../README.md) — vue d'ensemble du cookbook
