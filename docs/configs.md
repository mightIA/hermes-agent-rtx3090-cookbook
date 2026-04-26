# Configs reproductibles

> Configs validées sur RTX 3090 24 GB + WSL2 Ubuntu + Hermes Agent v0.11.0 (post-update fix `reasoning_content`) + Ollama. Caviardées : remplace les `<placeholder>` par tes valeurs locales.

## Sommaire

1. [Modelfile minimal — Qwen 3.5 27B Q5 (validé post-update)](#1-modelfile-minimal--qwen-35-27b-q5)
2. [Modelfile durci — Qwen 3.x (fallback ou pré-update)](#2-modelfile-durci--qwen-3x-fallback)
3. [Hermes Agent — `~/.hermes/config.yaml`](#3-hermes-agent--hermesconfigyaml)
4. [WSL2 — `.wslconfig` 28 GB pour 70B Q3](#4-wsl2--wslconfig-28-gb-pour-70b-q3)
5. [Add-on Home Assistant `homeassistant-ai/ha-mcp`](#5-add-on-home-assistant-homeassistant-aiha-mcp)
6. [Tableau VRAM ↔ `num_ctx` ↔ KV cache (RTX 3090 24 GB)](#6-tableau-vram--num_ctx--kv-cache)

---

## 1. Modelfile minimal — Qwen 3.5 27B Q5

> ✅ **Config principale validée** sur 3 scénarios variés (action / lecture / multi-step), latence 2 m 40 s à 5 m 06 s, post-update Hermès fix `has_reasoning guard`.

Fichier `~/Modelfile.qwen35-agent` :

```Modelfile
FROM qwen3.5:27b

PARAMETER num_ctx 65536
PARAMETER num_predict 4096
PARAMETER temperature 0.3
PARAMETER top_p 0.9
PARAMETER top_k 40
PARAMETER repeat_penalty 1.05

SYSTEM """You are <agent-name>, the master agent for the home assistant.
You always respond in concise direct French.

CRITICAL RULES:
1. You MUST call MCP tools IMMEDIATELY when an action is requested.
2. NEVER narrate before a tool call. No "I will now...", no "Let me first...", no "Okay, I am going to...".
3. Call the tool directly, wait for the result, then confirm in French in 1-2 short sentences.
4. For Home Assistant actions, use the ha_* tools from the ha-mcp MCP server.
5. Tool calls take precedence over conversation. Output ONLY the tool call when an action is needed.

You operate via Hermes Agent which orchestrates MCP calls and memory.
"""
```

Construction :
```bash
ollama create <agent-name> -f ~/Modelfile.qwen35-agent
```

Vérification :
```bash
ollama show <agent-name>
ollama list | grep <agent-name>
```

**VRAM réelle observée** : 28 GB total (poids 17 GB + KV cache 65k ≈ 11 GB) → offload CPU 20 % / 80 %. Acceptable post-update Hermès. Voir [tableau §6](#6-tableau-vram--num_ctx--kv-cache) pour les autres `num_ctx`.

---

## 2. Modelfile durci — Qwen 3.x (fallback)

> ⚠️ Cette config était une rustine côté client pour contourner le bug `has_reasoning guard` Hermès (pré-update fin avril 2026). Post-update, le Modelfile minimal §1 suffit. Garder ce Modelfile durci comme **fallback** si le bug réapparaît ou si tu travailles avec un Hermès non à jour.

Fichier `~/Modelfile.qwen3-agent` :

```Modelfile
FROM qwen3:32b

PARAMETER num_ctx 32768
PARAMETER num_predict 4096
PARAMETER temperature 0.3
PARAMETER top_p 0.9
PARAMETER top_k 20
PARAMETER repeat_penalty 1.05

SYSTEM """You are <agent-name>, the master agent. CRITICAL RULES:
1. Tool calls MUST be emitted as <tool_call>{"name":"...","arguments":{...}}</tool_call>
2. Do NOT narrate before calling a tool. Call it directly.
3. Do NOT use <think> blocks before tool calls.
4. After receiving tool results, summarize concisely in French.
"""
```

Pourquoi chaque paramètre :
- `num_ctx 32768` — aligné capacité native Qwen 3 (32k natif, extensible 128k via YaRN). Au-dessus, le modèle dégrade silencieusement.
- `num_predict 4096` — marge confortable pour réponse + tool calls multiples sans tronquer.
- `temperature 0.3` — basse température = JSON tool call plus déterministe (évite les hallucinations de paramètres).
- `top_p 0.9` + `top_k 20` — recommandations communautaires r/LocalLLaMA pour tool calling fiable.
- `repeat_penalty 1.05` — léger anti-loop pour éviter les boucles « nudge » Hermès.
- SYSTEM — force le format `<tool_call>` natif Hermès et désactive `<think>` qui casse les renderers downstream.

Construction et activation :
```bash
ollama create <agent-name> -f ~/Modelfile.qwen3-agent

# Backup config Hermès AVANT modification
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak.$(date +%Y%m%d_%H%M%S)

# Bascule (adapter aux clés exactes de ta version Hermès)
sed -i 's|^  default: qwen3:32b$|  default: <agent-name>|' ~/.hermes/config.yaml
sed -i 's|^  model: qwen3:32b$|  model: <agent-name>|' ~/.hermes/config.yaml
```

Pattern transposable :
- Adapter `FROM` à d'autres modèles Qwen 3.x (`qwen3:14b`, `qwen3.5:27b`).
- Garder le `SYSTEM` quasi-identique (juste adapter le rôle / agent name).
- Adapter `num_ctx` à la capacité native du modèle cible.

Sources upstream :
- [Issue Ollama #11662 — qwen3:32b tool calling broken](https://github.com/ollama/ollama/issues/11662)
- [Issue Ollama #14601 — Qwen3 tool calling /api/chat malformed](https://github.com/ollama/ollama/issues/14601)
- [Issue Ollama #14617 — Disable thinking via Modelfile](https://github.com/ollama/ollama/issues/14617)
- [Issue Ollama #11135](https://github.com/ollama/ollama/issues/11135)
- [Issue Ollama #14745](https://github.com/ollama/ollama/issues/14745)

---

## 3. Hermes Agent — `~/.hermes/config.yaml`

Extrait minimal (caviardé) :

```yaml
default: <agent-name>

provider: custom
base_url: http://localhost:11434/v1
context_length: 65536

custom_providers:
  - name: custom
    base_url: http://localhost:11434/v1
    api_key: ollama   # placeholder, Ollama ignore le contenu
    model: <agent-name>

mcp_servers:
  ha-mcp:
    type: http
    url: https://<your-ha-mcp-domain>/<SECRET_PATH>
    # exemple générique. Voir §5 pour le détail add-on ha-mcp.

auxiliary:
  compression:
    model: mistral-nemo:12b   # 128k ctx natif, OK pour résumés text-to-text
  # Les autres sections auxiliaires (planning, etc.) héritent du modèle principal
  # par défaut. Override ici si besoin.
```

### Notes

- **Provider `custom` vs `ollama` natif** — Hermès supporte un provider `ollama` natif depuis quelques versions. La config `custom` ci-dessus reste valide et permet de pointer vers n'importe quel endpoint OpenAI-compatible (vLLM, llama.cpp, OpenRouter, etc.). Si tu veux tester `provider: ollama` natif, c'est la phase 2 de l'audit méthodologique 2.
- **`context_length: 65536`** — Hermès tronque côté client si la conversation dépasse cette valeur. Doit être **≤ ou =** au `num_ctx` du Modelfile Ollama (voir [tableau §6](#6-tableau-vram--num_ctx--kv-cache)).
- **`auxiliary.compression`** — héritage du modèle principal par défaut. Override par `mistral-nemo:12b` recommandé : 128k ctx natif suffit pour résumer une longue conversation, et c'est plus rapide qu'un 27B Q5.
- **Backup obligatoire** avant tout `sed -i` :

  ```bash
  cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak.$(date +%Y%m%d_%H%M%S)
  ```

### Pièges connus

- `hermes mcp add` ne persiste pas la config (cf. [`troubleshooting.md` Symptôme 9](troubleshooting.md#symptôme-9--hermès-mcp-add-ne-persiste-pas)). Édition manuelle obligatoire.
- Les modèles auxiliaires (`auxiliary.*`) **héritent** du modèle principal par défaut. Override section par section requis quand le contexte / les capabilities diffèrent (ex. `compression.model: mistral-nemo:12b` quand le principal est qwen3.5:27b avec 32k natif < 64k Hermès).
- Versions Hermès en dev intense : si tu vois un comportement bizarre, vérifier `hermes --version` et faire `hermes update` AVANT de débugger ta config.

---

## 4. WSL2 — `.wslconfig` 28 GB pour 70B Q3

> ⚠️ Nécessaire uniquement si tu testes des modèles 70B Q3 ou des `num_ctx` > 65k sur 27-32B Q5. Pour Qwen 3.5 27B Q5 + `num_ctx 65536`, la config par défaut WSL2 (~15 Gi RAM) suffit en théorie ; en pratique on est juste à la limite, donc 28 GB recommandé.

Fichier `C:\Users\<user>\.wslconfig` (côté Windows, pas dans WSL2) :

```ini
[wsl2]
memory=28GB
swap=8GB
```

Application :
```powershell
# PowerShell Windows (pas WSL2)
wsl --shutdown
```

Patienter ~10 s puis relancer un terminal WSL2.

Vérification :
```bash
# Dans WSL2 Ubuntu
free -h
# Mem total doit afficher ~27 Gi (au lieu de 15 Gi par défaut)
# Swap total doit afficher 8.0 Gi
```

### Pourquoi 28 GB

- 32 GB physiques côté Windows
- ~4 GB réservés à Windows + apps Windows (navigateur, IDE, autres process)
- 28 GB alloués à WSL2 = compromis sain, laisse de la marge à Windows
- Si tu as 64 GB physiques, vise `memory=56GB` (laisser ~8 GB à Windows)

### Pourquoi swap 8 GB

- Buffer pour pics inattendus (KV cache extension, multiples gros LLM en parallèle)
- 4 GB par défaut (config Microsoft) trop juste pour un 70B
- Le SSD NVMe absorbe sans problème

### Effets de bord
- `wsl --shutdown` tue **toutes** les distros WSL2 actives (Hermès, Ollama, processus en cours interrompus, redémarrent seuls).
- Les apps Windows ne sont pas affectées (navigateur, IDE, etc.).

---

## 5. Add-on Home Assistant `homeassistant-ai/ha-mcp`

Add-on : [`homeassistant-ai/ha-mcp`](https://github.com/homeassistant-ai/ha-mcp). Fournit un endpoint MCP HTTP authentifié pour Home Assistant (contourne le bug DCR du `mcp_server` core).

### Config minimale (Home Assistant UI → Apps → ha-mcp → Configuration)

```yaml
secret_path: <generate-a-strong-random-token>
enable_tool_search: false
# autres options selon version add-on
```

### Champs

- **`secret_path`** — segment d'URL aléatoire qui authentifie tous les appels. Génère **24 caractères minimum** entropie haute (ex. via `openssl rand -base64 24` puis URL-safe). À NE PAS publier, traiter comme un mot de passe.
- **`enable_tool_search`** — booléen. Compromis :
  - `false` (recommandé pour modèles ≥ 27B Q4-Q5) : 87 outils en clair, ~46 K tokens dans le contexte idle. Pas de méta-recherche, le modèle voit tout. Performances tool calling meilleures sur modèles capables.
  - `true` (recommandé pour modèles ≤ 14B Q4) : ~5 K tokens dans le contexte idle, outils trouvés via `ha_search_tools` puis exécutés via proxies (read / write / delete). Permet aux modèles plus petits de gérer.

### Endpoint résultant

```
https://<your-ha-mcp-domain>/<secret_path>
```

ou si exposé en local LAN :
```
http://<ha-host-or-ip>:9583/<secret_path>
```

### Sécurité

- L'add-on **n'a pas** d'auth standard (pas de Basic Auth, pas de token Bearer). La sécurité repose entièrement sur l'**imprévisibilité du `secret_path`**.
- Pour exposer en HTTPS public : passer par un Cloudflare Tunnel ou équivalent, **avec bypass MFA** sur le path MCP (les LLM agents ne sauront pas faire un challenge MFA).
- Régénérer le `secret_path` est une opération lourde : il faut **patcher partout** (Hermès `config.yaml`, autres clients MCP, scripts de monitoring) avant Stop+Start de l'add-on.

### Régénération du `secret_path`

Procédure en 6 étapes (transposable) :

1. Générer un nouveau token : `openssl rand -base64 24 | tr -d '/+=' | head -c 24` (24 chars, URL-safe).
2. HA UI → Apps → ha-mcp → Configuration → remplacer `secret_path` → Sauvegarder.
3. Stop puis Start (Restart **ne suffit pas**).
4. Tester avec `curl` : doit renvoyer 405/404 sur l'ancien path, 200 ou 405 sur le nouveau (selon la méthode).
5. `sed -i` ou édition manuelle dans **tous** les fichiers projet qui référencent l'ancien token.
6. Update `~/.hermes/config.yaml` (`mcp_servers.ha-mcp.url`).

Toujours vérifier qu'aucun fichier versionné public ne contient le `secret_path` (cf. ce repo, qui ne contient que des placeholders `<SECRET_PATH>`).

---

## 6. Tableau VRAM ↔ `num_ctx` ↔ KV cache

> Approximation pratique pour les modèles 27-32B Q4-Q5 sous Ollama (KV cache Q8 par défaut). RTX 3090 24 GB strict.

### Formule de calcul

```
VRAM_total = poids_modèle + KV_cache(num_ctx)
KV_cache ≈ num_ctx × 0.0001875 GB    (pour 27B Q5, KV cache Q8)
```

### Table

| `num_ctx` | KV cache 27B | Total 27B Q5 (~17 GB) | Total 32B Q5 (~20 GB) | Tient en 24 GB ? |
|---|---|---|---|---|
| 16 384 | ~3 GB | 20 GB | 23 GB | ✅ 100 % GPU |
| 32 768 | ~6 GB | 23 GB | 26 GB | ✅ 27B / ⚠️ 32B (offload léger) |
| 65 536 | ~11 GB | 28 GB | 31 GB | ❌ offload CPU 20-30 % |
| 131 072 | ~22 GB | 39 GB | 42 GB | ❌ gros offload CPU 50 %+ |

### Lecture

- Si tu cherches **latence minimale** : choisir le plus grand `num_ctx` qui tient en 100 % GPU (32 768 pour 27B Q5).
- Si tu acceptes l'offload CPU : `num_ctx 65536` reste viable post-update Hermès, ~5 min/tour pour une action HA simple.
- `num_ctx > 65k` : à éviter, latence dégrade rapidement (offload CPU > 50 %).

### Vérification ratio CPU/GPU réel

```bash
# Pendant un test Hermès
ollama ps
# Doit afficher quelque chose comme :
# NAME              ID    SIZE      PROCESSOR
# <agent>:latest    ...   28 GB     20%/80% CPU/GPU
```

ou via `nvidia-smi` côté Windows pour la VRAM réelle utilisée.

### Exception modèles ≤ 14B

Cette règle ne s'applique pas aux modèles ≤ 14B (rentrent en 24 GB avec n'importe quel `num_ctx` ≤ 256k). Pour ces modèles, le facteur limitant est plutôt la **qualité sémantique** sur tool calling, pas la VRAM.

---

## Fichiers de référence externes

- [Hermes Agent — Repository](https://github.com/NousResearch/hermes-agent)
- [Hermes Agent — Documentation](https://github.com/NousResearch/hermes-agent/blob/main/README.md)
- [Ollama — Documentation Modelfile](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)
- [add-on `homeassistant-ai/ha-mcp`](https://github.com/homeassistant-ai/ha-mcp)
- [WSL2 `.wslconfig` documentation](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)
