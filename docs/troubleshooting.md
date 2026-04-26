# Troubleshooting — symptômes, causes, fixes

> Catalogue des symptômes rencontrés sous **Hermes Agent v0.11.0 + Ollama + add-on `homeassistant-ai/ha-mcp`** sur RTX 3090 24 GB. Pour chaque symptôme : modèles concernés, cause racine identifiée, fix validé, méthode de vérification.
>
> 📌 **Avant de plonger ici, fais d'abord `hermes update` et vérifie `hermes --version`.** La majorité des symptômes ci-dessous étaient causés par un même bug `has_reasoning guard` côté Hermès, résolu par les commits `5ae60815` / `f93d4624` / `63bf7a29` / `9daa0620` (fin avril 2026). Si tu n'es pas à jour, ne perds pas de temps avec les autres pistes.

## Sommaire

1. [`Model returned empty after tool calls` / boucle nudge](#symptôme-1--model-returned-empty-after-tool-calls--boucle-nudge)
2. [Tool call émis comme texte JSON dans le `content` au lieu d'un appel structuré](#symptôme-2--tool-call-dans-le-content-au-lieu-dun-appel-structuré)
3. [Inversion de rôle / SYSTEM prompt ignoré](#symptôme-3--inversion-de-rôle--system-prompt-ignoré)
4. [Hallucination `write_file` / `curl` au lieu d'appeler le tool MCP](#symptôme-4--hallucination-write_file--curl-au-lieu-du-tool-mcp)
5. [Latence catastrophique sur 70B Q3 (>10 min / tour)](#symptôme-5--latence-catastrophique-sur-70b-q3)
6. [Ollama refuse de charger un modèle : `model requires more system memory than available`](#symptôme-6--ollama-http-500-model-requires-more-memory)
7. [Modèles 8-12B hallucinent face à un gros catalogue MCP (>50 outils)](#symptôme-7--modèles-8-12b-hallucinent-sur-gros-catalogue-mcp)
8. [`<think>` blocks polluent le tool calling sur Qwen 3 / Qwen 3.5 / DeepSeek-R1](#symptôme-8--think-blocks-dans-tool-calling)
9. [Hermès `mcp add` ne persiste pas la config](#symptôme-9--hermès-mcp-add-ne-persiste-pas)

---

## Symptôme 1 — `Model returned empty after tool calls` / boucle nudge

### Modèles concernés
- `qwen3:32b` (Q4)
- `qwen3.5:27b` (Q5)
- `qwen2.5-coder:32b` (Q4-Q5)
- Tous les modèles avec **mode reasoning natif** (Qwen 3.x, DeepSeek-R1, Kimi)

### Symptôme observé
Hermès appelle correctement le tool MCP, l'outil renvoie un résultat valide, puis le modèle « retourne vide ». Hermès affiche `Model returned empty after tool calls — nudging to continue`, le modèle re-tente, vide à nouveau, boucle. Le contexte est consommé sans réponse utile à l'utilisateur. Latence 5-12 min, finit en timeout ou réponse incohérente.

### Cause racine
Bug `has_reasoning guard` dans Hermes Agent v0.11.0 **avant** les commits ci-dessous. Le guard décidait selon le modèle si on injectait ou pas le `reasoning_content` dans le replay du tool-call. Sur les modèles avec reasoning natif, le guard provoquait des regenérations parasites Ollama (boucles `<think>`, injection/retrait répété de `reasoning_content`).

### Fix
```bash
# Dans WSL2
hermes update

# Vérifier
hermes --version
# Doit afficher "Up to date" et un hash upstream récent dans le titre TUI
```

Les commits qui apportent le fix :
- `5ae60815` — `fix: remove has_reasoning guard — inject empty reasoning_content for DeepSeek/Kimi tool_calls unconditionally`
- `f93d4624` — `Merge PR #15749 — fix copy-reasoning-content-ordering and cross-provider-isolation`
- `63bf7a29` — `fix(run_agent): prevent reasoning_content regression in DeepSeek/Kimi tool-call replay`
- `9daa0620` — `fix(agent): ordering fix in _copy_reasoning_content_for_api — cross-provider reasoning isolation`

### Vérification
Refaire un test action simple HA (création de notification persistante par exemple). Latence avant fix : 20 min. Latence après fix : ~5 min. Le tool call reste structuré, la réponse texte arrive en français.

### Si le fix ne résout pas
- Vérifier que tu es vraiment sur un commit ≥ `087e74d4` (hash upstream visible dans le titre TUI Hermès post-update).
- Voir [Symptôme 4](#symptôme-4--hallucination-write_file--curl-au-lieu-du-tool-mcp) pour la dérive résiduelle.
- Voir [Symptôme 8](#symptôme-8--think-blocks-dans-tool-calling) pour les `<think>` blocks parasites.

---

## Symptôme 2 — Tool call dans le `content` au lieu d'un appel structuré

### Modèles concernés
- `llama3.3:70b-instruct-q3_K_M`
- Potentiellement d'autres modèles Llama 3.x sous Ollama

### Symptôme observé
Le modèle « répond » par un texte JSON brut qui ressemble à un tool call (`{"name": "...", "arguments": {...}}`) directement dans le `content` du message assistant, au lieu d'émettre un vrai appel structuré que Hermès puisse intercepter. Hermès ne le détecte pas comme tool call et l'affiche tel quel à l'utilisateur. Souvent accompagné de :
- JSON malformé (mélange de `name` et `arguments` de plusieurs outils différents)
- Narration explicite (`"Pour appeler l'outil, voici la commande :"`) qui viole les règles SYSTEM
- Latence > 10 min/tour pour ~80 tokens (chargement modèle + offload CPU lourd)

### Cause racine
Template de chat Llama 3.x sous Ollama mal câblé pour le format tool calling natif Llama (`<|python_tag|>` ou `<|tool_call|>`). Le modèle « sait » qu'il doit faire un tool call mais ne sait pas l'émettre dans le format attendu, donc il écrit du JSON dans le `content`. Famille bug Ollama [#11662](https://github.com/ollama/ollama/issues/11662).

S'ajoute pour les Q3 70B sur RTX 3090 24 GB : offload CPU obligatoire (~10 GB RAM en plus du KV cache), latence ~0.12 tok/s effectif.

### Fix
- **Pas de fix simple côté Ollama+Hermès direct.** Ce modèle, dans cette stack, est inutilisable.
- Alternatives :
  - Tester via **vLLM** avec un template chat custom (extraction du format Llama natif)
  - Passer par **OpenRouter** (cloud) pour Llama 3.3 70B en mode agent
  - Choisir un autre modèle local : Qwen 3.5 27B Q5 marche bien post-update fix Symptôme 1

### Vérification
```bash
ollama show llama3.3:70b-instruct-q3_K_M
# Regarder la section Template — si elle ne câble pas correctement le rôle "tool"
# et n'utilise pas <|python_tag|>, le bug est confirmé côté Ollama
```

### Note hardware
**Le hardware upgrade ne résout pas ce symptôme.** Même avec 64 GB RAM et CPU plus rapide, le facteur limitant reste la VRAM 24 GB de la RTX 3090 (offload CPU obligatoire pour 70B Q3) **ET** le template Ollama cassé. Il faudrait une carte 32+ GB VRAM (RTX 5090) **et** un fix Ollama pour Llama 3.3, ou changer de stack.

---

## Symptôme 3 — Inversion de rôle / SYSTEM prompt ignoré

### Modèles concernés
- `mistral-nemo:12b` (7.1 GB, 128k ctx natif)
- Potentiellement d'autres Mistral récents sous Ollama

### Symptôme observé
Sur un scénario test typique (ex. `Crée une notification persistante via ha-mcp dire "Test"`) :
- Le modèle répond comme s'il était l'utilisateur : *"Bonjour, pouvez-vous m'aider à installer Node.js ?"*
- Aucun tool call n'est émis
- Le SYSTEM prompt « tu es Jarvis, agent qui pilote Home Assistant via MCP » est totalement ignoré (test discriminant `"qui es-tu en une phrase ?"` → réponse générique d'agent CLI, pas Jarvis)
- Mélange linguistique mineur (mots EN dans réponse FR)

### Cause racine
Template de chat Mistral mal mappé sous Ollama quand un SYSTEM non trivial **et** un catalogue de tools sont présents simultanément. Le modèle ne reçoit pas correctement la séparation des rôles `system` / `user` / `assistant` / `tool`. Hermès n'a pas non plus d'adapter Mistral récent capable d'extraire le format `[TOOL_CALLS]` natif de mistral-nemo. Famille bug Ollama #11662 / #14601.

### Fix
- **Pas de fix simple en moteur principal.** Ne pas retenter `mistral-nemo:12b` comme moteur agent tant qu'un upgrade Ollama ou Hermès dédié n'est pas testé sur ce point précis.
- **Mistral-nemo reste OK comme `auxiliary.compression.model`** dans Hermès — pas de tool calling, pas de catalogue MCP, juste un résumé text-to-text. Override possible dans `~/.hermes/config.yaml` :

  ```yaml
  auxiliary:
    compression:
      model: mistral-nemo:12b   # OK ici, pas en principal
  ```

- Pour utiliser Mistral en agent : considérer un autre frontend (vLLM, llama.cpp direct avec template custom) ou attendre un fix Ollama.

### Vérification (test discriminant)
Avant tout test scénario tool calling, exécuter :
```
qui es-tu en une phrase ?
```
- Si la réponse ressemble au SYSTEM (ex. *"Je suis Jarvis, ton agent..."*), le SYSTEM est respecté → on peut tester un scénario tool calling.
- Si la réponse est générique (*"Modèle d'agent conversationnel exécuté dans votre terminal..."*), le SYSTEM ne passe pas → inutile de tester du tool calling.

---

## Symptôme 4 — Hallucination `write_file` / `curl` au lieu du tool MCP

### Modèles concernés
- `qwen3:32b` (Q4-Q5)
- `qwen3.5:27b` (Q5)
- `qwen2.5-coder:32b` (Q4-Q5)
- Modèles 27-32B Q4-Q5 globalement, surtout en mode `enable_tool_search: true`

### Symptôme observé
Au lieu d'appeler le tool MCP attendu (ex. `ha_call_service`, `ha_set_entity`), le modèle hallucine :
- `write_file /tmp/notification.yaml` (alors que ce tool n'existe pas dans le catalogue MCP)
- `curl -X POST http://homeassistant.local:8123/...` (texte brut sans tool call)
- Dérive sémantique : la demande « crée notification » devient « crée script Python » ou « génère un YAML d'automation »

### Cause racine
Identique au [Symptôme 1](#symptôme-1--model-returned-empty-after-tool-calls--boucle-nudge) : `has_reasoning guard` cassé Hermès. Le `<think>` Qwen non-géré pollue le contexte, le modèle se perd et hallucine des alternatives plausibles à partir de son training (write_file, curl).

### Fix
1. **D'abord** : `hermes update` (cf. Symptôme 1). Cela résout la majorité des cas.
2. **Si persistant après update** : durcir le Modelfile pour désactiver le mode `<think>` et forcer le format tool call (cf. [`configs.md`](configs.md) section *Modelfile durci*).
3. **Si persistant avec Modelfile durci** : tester en `enable_tool_search: false` (87 outils en clair). Le pattern dérive a été observé surtout en `enable_tool_search: true`. Compromis : 87 outils en clair = ~46 K tokens de contexte idle.

### Vérification
Test multi-step typique : `Lis la température de la cuisine puis si > 22 °C crée une notification`. Si le modèle :
- Cite la valeur exacte lue (ex. `22.12 °C`)
- Évalue la condition explicitement
- Émet **deux** tool calls structurés (lecture puis écriture)
- Répond en FR concis

… alors le fix est validé.

---

## Symptôme 5 — Latence catastrophique sur 70B Q3

### Modèles concernés
- `llama3.3:70b-instruct-q3_K_M` (~34 GB)
- Tout modèle 70B Q3 sur GPU 24 GB

### Symptôme observé
- Cold start chargement : ~4 min (mapping 34 GB vers VRAM/RAM)
- Génération warm : **10 min 52 s par tour** pour ~80 tokens (~0.12 tok/s effectif)
- KV cache 32k contexte ajoute ~5-7 GB RAM/VRAM (Llama 3.3 GQA 8 KV heads)

### Cause racine
70B Q3 (34 GB) ne tient pas en VRAM 24 GB pure → offload partiel CPU obligatoire (~10 GB en RAM). Vu Ollama, c'est ~8-15 tok/s **théorique** mais en pratique 0.12 tok/s effectif sur un tour avec catalogue 20 tools + SYSTEM dur. Le bottleneck est le bus PCIe + la latence CPU pour les couches offloadées.

### Fix
- **Pas de fix logiciel.** C'est un problème hardware fondamental.
- Alternatives :
  - **Attendre RTX 5090 32 GB** (ou équivalent) pour faire tenir 70B Q3 en VRAM pure
  - **Cloud LLM via OpenRouter** : Claude Haiku 4.5, GPT-4o-mini, Llama 3.3 70B hébergé. Coût pay-as-you-go raisonnable, latence ~5-15 s/tour
  - **Quantisation plus basse** (`q2_K`, ~24 GB) : qualité sémantique sévèrement dégradée, pas recommandé pour agent
  - **Modèles plus petits** (Qwen 3.5 27B Q5, Mistral Small 3, Qwen 3.5 14B) : ce qu'on a fait, ça marche

### Lien upgrade hardware
**Attention** : un upgrade CPU + RAM (Ryzen 9 7950X + 64 GB DDR5) **ne résout pas** ce symptôme. Le facteur limitant est la VRAM 24 GB de la RTX 3090, pas la RAM. Seul un upgrade GPU 32+ GB sauverait le 70B Q3 en VRAM pure.

---

## Symptôme 6 — Ollama HTTP 500 `model requires more memory`

### Modèles concernés
- Tout modèle Q3/Q4 70B+ sous Ollama
- Modèles 27-32B Q5 avec gros num_ctx (≥ 65k)

### Symptôme observé
```
HTTP 500: model requires more system memory (20.2 GiB) than is available (18.2 GiB)
```
ou similaire, lors du `ollama run` ou du premier appel tool calling.

### Cause racine
WSL2 alloue **50 % de la RAM Windows par défaut**. Sur une machine 32 GB physiques, ça donne ~15 Gi côté WSL2 (visible via `free -h` dans Ubuntu). Le calcul Ollama inclut les couches offloadées CPU + KV cache + overhead moteur. Total demandé > total disponible côté WSL2.

### Fix
Créer `C:\Users\<user>\.wslconfig` (côté Windows, pas dans WSL2) :

```ini
[wsl2]
memory=28GB
swap=8GB
```

Puis depuis **PowerShell Windows** (pas dans WSL2) :
```powershell
wsl --shutdown
```

Patienter ~10 s, relancer un terminal WSL2.

### Vérification
```bash
free -h
# Mem total doit afficher ~27 Gi (au lieu de 15 Gi)
# Swap doit afficher 8.0 Gi
```

### Pourquoi 28 GB et pas plus
- 32 GB physiques côté Windows
- ~4 GB réservés à Windows + apps Windows (navigateur, IDE, etc.)
- 28 GB alloués à WSL2 = compromis sain, laisse de la marge à Windows
- Si tu as 64 GB physiques, vise `memory=56GB` (laisser ~8 GB à Windows)

### Effets de bord
- `wsl --shutdown` tue **toutes** les distros WSL2 actives (Hermès meurt, Ollama meurt, processus en cours interrompus, redémarrent seuls).
- Les apps Windows ne sont pas affectées (navigateur, IDE, etc.).

---

## Symptôme 7 — Modèles 8-12B hallucinent sur gros catalogue MCP

### Modèles concernés
- `llama3.1:8b` (4.9 GB)
- `mistral-nemo:12b` (7.1 GB, 128k ctx)
- Globalement tout modèle ≤ 14B Q4 face à > 50 outils MCP

### Symptôme observé
Le modèle « comprend » qu'il doit appeler un tool (mention dans le content), mais :
- N'émet pas d'appel structuré
- Hallucine une présentation générique (« voici comment je procéderais »)
- Choisit le mauvais outil
- Affiche `I don't see any tool calls` après son propre tool call

### Cause racine
Submersion par le contexte. Les définitions JSON Schema des 87 outils chargent ~46 K tokens dans le contexte **avant** la requête utilisateur. Les modèles 8-12B saturent leur attention sur ces définitions et hallucinent au lieu de sélectionner correctement.

### Fix
**Solution privilégiée** : activer le flag `enable_tool_search: true` dans la config de l'add-on `homeassistant-ai/ha-mcp` côté Home Assistant UI :

1. Home Assistant UI → Apps (anciennement Add-ons) → `Home Assistant MCP Server`
2. Configuration → cocher `Enable tool search` → Sauvegarder
3. Stop puis Start (Restart **ne suffit pas**)

Effet : remplace le catalogue complet (~46 K tokens idle) par un système de recherche (~5 K tokens). Les outils sont trouvés via `ha_search_tools` puis exécutés via proxies catégorisés (read / write / delete).

### Solution de repli
Utiliser un modèle plus capable :
- 27-32B Q4-Q5 (Qwen 3.5 27B Q5 validé, voir [`configs.md`](configs.md))
- 70B Q3 si tu as la VRAM (sinon voir [Symptôme 5](#symptôme-5--latence-catastrophique-sur-70b-q3))
- Cloud LLM via OpenRouter (Claude, GPT, Mistral Large)

### Note
Le mode `enable_tool_search: true` introduit son propre risque (cf. [Symptôme 4](#symptôme-4--hallucination-write_file--curl-au-lieu-du-tool-mcp) pré-update). Post-update Hermès fix `reasoning_content`, à retester. Voir aussi [`configs.md`](configs.md) pour les compromis.

---

## Symptôme 8 — `<think>` blocks dans tool calling

### Modèles concernés
- `qwen3:32b`, `qwen3:14b`
- `qwen3.5:27b`, `qwen3.5:14b`
- `deepseek-r1:14b` (mais celui-ci ne supporte pas le tool calling Ollama → HTTP 400)
- Tout modèle avec mode reasoning natif activé par défaut

### Symptôme observé
Avant chaque tool call, le modèle émet un block `<think>` plus ou moins long (1-3 paragraphes de raisonnement interne). Hermès ou les renderers downstream ne savent pas toujours quoi en faire :
- Affichage texte brut `<think>...</think>` dans la réponse user
- Boucle de regenération côté Ollama (couplé au [Symptôme 1](#symptôme-1--model-returned-empty-after-tool-calls--boucle-nudge))
- Latence dégradée

### Cause racine
Mode reasoning natif activé par défaut sur Qwen 3.x. Combiné au bug `has_reasoning guard` (avant le fix), causait des regenérations parasites.

### Fix
1. **`hermes update`** d'abord (résout la majorité des cas).
2. Si tu veux explicitement désactiver le mode `<think>` côté modèle, durcir le Modelfile :

   ```Modelfile
   FROM qwen3:32b
   PARAMETER num_ctx 32768
   PARAMETER temperature 0.3
   PARAMETER top_p 0.9
   PARAMETER top_k 20
   PARAMETER repeat_penalty 1.05
   SYSTEM """You are Jarvis, the master agent. CRITICAL RULES:
   1. Tool calls MUST be emitted as <tool_call>{"name":"...","arguments":{...}}</tool_call>
   2. Do NOT narrate before calling a tool. Call it directly.
   3. Do NOT use <think> blocks before tool calls.
   4. After receiving tool results, summarize concisely in French."""
   ```

   Voir [`configs.md`](configs.md) pour le détail et le pattern complet.

### Vérification
Lancer un scénario test action HA. Si la réponse ne contient plus de `<think>...</think>` visible dans le `content`, et que le tool call s'exécute proprement, c'est OK.

### Note S63 post-update
Le Modelfile durci était une rustine côté client pour contourner le bug Hermès `has_reasoning guard`. Avec Hermès post-update, le Modelfile minimal sans hack `<think>` peut suffire (à confirmer par tests P3). Le Modelfile durci reste valable comme fallback.

---

## Symptôme 9 — Hermès `mcp add` ne persiste pas

### Versions concernées
- Hermes Agent v0.11.0

### Symptôme observé
```bash
hermes mcp add ha-mcp --url https://example.com/mcp/<SECRET>
# → l'agent tente la connexion, affiche le succès, lance un test
# → mais ne persiste rien dans ~/.hermes/config.yaml
```

Au prochain démarrage de Hermès, le MCP `ha-mcp` n'apparaît pas dans le banner.

### Cause racine
Bug ou design CLI : `hermes mcp add` lance un agent test mais n'écrit pas la config en dur.

### Fix
Édition manuelle de `~/.hermes/config.yaml`. Section `mcp_servers:` à ajouter ou compléter :

```yaml
mcp_servers:
  ha-mcp:
    type: http
    url: https://example.com/mcp/<SECRET_PATH>
    # adapter selon ton add-on / endpoint MCP
```

Sauvegarder, redémarrer Hermès. Le MCP doit apparaître dans le banner avec le nombre d'outils détectés.

### Vérification
Au démarrage de Hermès, le banner doit afficher quelque chose comme :
```
ha-mcp (http) — 87 tool(s)
```
ou (si `enable_tool_search: true`) :
```
ha-mcp (http) — 20 tool(s)
```

---

## Si rien ne marche

1. Refaire un `hermes update` (peut-être un nouveau fix critique a été poussé entre-temps — Hermes Agent voit ~40 commits/jour sur certaines périodes).
2. Lancer le pattern d'audit méthodologique 2 (cf. [`audit-methodologique.md`](audit-methodologique.md)) — il a été conçu exactement pour ces situations.
3. Ouvrir une issue sur le repo [Hermes Agent](https://github.com/NousResearch/hermes-agent/issues) avec :
   - Version Hermès (`hermes --version` complet, hash upstream)
   - Version Ollama (`ollama --version`)
   - Modèle utilisé + Modelfile complet
   - Extrait `~/.hermes/config.yaml` (caviardé)
   - Trace minimale qui reproduit
4. Croiser avec les Issues Ollama tool calling : [#11662](https://github.com/ollama/ollama/issues/11662), [#14601](https://github.com/ollama/ollama/issues/14601), [#11135](https://github.com/ollama/ollama/issues/11135), [#14745](https://github.com/ollama/ollama/issues/14745), [#14617](https://github.com/ollama/ollama/issues/14617).
