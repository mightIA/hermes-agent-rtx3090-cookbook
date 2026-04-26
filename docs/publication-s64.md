# Publication GitHub — procédure init repo public (26/04/2026)

> Procédure pas-à-pas suivie pour publier ce cookbook comme repo public GitHub, avec garde-fous email no-reply et caviardage triple `grep`. Reproductible pour publier d'autres repos issus d'un dossier local privé.

## Contexte

Le contenu de ce cookbook avait été produit pendant ~1 semaine de sessions de travail privées (S57 à S63), dans un dossier local non-versionné. Une fois le verdict S63 stabilisé (cf. [`journey-s57-s63.md`](journey-s57-s63.md)), la décision a été prise de publier comme repo public GitHub pour partage communautaire.

Contraintes :

- Aucune fuite d'information personnelle (emails, IPs LAN, domaines internes, hostnames, paths nominatifs)
- Aucun risque que l'email Gmail personnel apparaisse dans un futur `git log` public
- Procédure reproductible — pas de manipulations « à la main » non documentées
- Coût marginal en temps et risque

Le résultat : commit racine `de7e268`, 7 fichiers, 1286 lignes, License MIT, durée totale ~1 h 45.

## Pré-requis

- Un compte GitHub. Pour ce cookbook, compte `mightIA` (compte distinct du compte Gmail principal — voir auto-memory `reference_compte_github_might.md`).
- Git installé côté Windows (Windows Git for Windows, ou Git fourni par GitHub Desktop).
- Git Credential Manager (GCM) — installé par défaut avec Git for Windows. Indispensable pour l'auth OAuth Browser.
- Brave (ou autre navigateur par défaut) ouvert sur le compte GitHub cible.
- Le dossier local à publier, déjà nettoyé du contenu privé.

## Étape 1 — Caviardage local (avant tout commit)

**Ne jamais committer en local avant le caviardage.** Un commit local est récupérable, mais une fois `git push` effectué, l'historique remote conserve la fuite **même après** un revert ou un force-push. La règle d'or : *ne jamais laisser le contenu sensible toucher l'index Git*.

### Triple grep multi-pattern

Lister les patterns sensibles à interdire, puis exécuter trois grep complémentaires :

```bash
# À adapter avec les patterns sensibles spécifiques au repo
PATTERNS=(
  "<domaine-interne>"          # ex. mon-domaine.tld
  "<sous-domaine-mcp>"         # ex. mcp.mon-domaine.tld
  "private_[A-Za-z0-9]{20,}"   # secret_path MCP
  "<prenom>"                   # prénom de l'auteur
  "<lieu-residence>"           # ville
  "<hostname-machine>"         # hostname Windows
  "/home/<user>/"              # path WSL2 nominatif
  "<user>@gmail\\.com"         # email perso 1
  "<user>@<autre>"             # email perso 2/3/4
  "192\\.168\\.[0-9]+\\.[0-9]+" # IPs LAN privées
  "10\\.[0-9]+\\.[0-9]+\\.[0-9]+" # IPs privées RFC1918
  "172\\.(1[6-9]|2[0-9]|3[0-1])\\." # IPs privées RFC1918
  "<session-id-prefix>"        # IDs internes Cowork/Hermès
)

# Grep 1 — sensibilité à la casse
for p in "${PATTERNS[@]}"; do
  echo "=== $p ==="
  grep -rn "$p" . --exclude-dir=.git --exclude-dir=node_modules
done

# Grep 2 — insensible à la casse
for p in "${PATTERNS[@]}"; do
  grep -rni "$p" . --exclude-dir=.git
done

# Grep 3 — recherche sur les fichiers binaires aussi (par sécurité)
for p in "${PATTERNS[@]}"; do
  grep -rna "$p" . --exclude-dir=.git
done
```

Toute occurrence remontée doit être corrigée avant l'étape suivante. **Zéro tolérance** : un faux positif accepté à la légère est une fuite garantie une fois publique.

### Exemples de remplacements génériques

| Sensible | Remplacement public |
|---|---|
| `mcp.mon-domaine.tld/private_ABC...` | `mcp.example.com/<secret_path>` |
| `192.168.1.11` | `192.168.1.X` ou `<HA-LAN-IP>` |
| `/home/john/projets/...` | `/home/<user>/projects/...` |
| `john.doe@gmail.com` | (à supprimer) ou `<user>@example.com` |
| `MIGHT-1000D` (hostname) | (à supprimer) |
| `MAJ S63` (référence interne session) | (à supprimer ou reformuler en *« run #N »*) |

## Étape 2 — Configuration Git locale (no-reply email)

```powershell
# PowerShell, dans le dossier local à publier
git config --global user.name "<github-username>"
git config --global user.email "<github-id>+<github-username>@users.noreply.github.com"
```

L'ID numérique GitHub (`<github-id>`) se trouve via `https://api.github.com/users/<username>` (champ `id`). L'email résultant est de la forme `12345678+username@users.noreply.github.com`.

**Vérification obligatoire** :

```powershell
git config --global --get user.email
# Doit afficher l'email no-reply, PAS l'email Gmail réel
```

C'est le **piège n°1** : `git config --global` renvoie souvent vide par défaut sous Windows malgré une déclaration *« déjà configuré »* faite à un autre moment. Vérification systématique avant chaque premier commit sur un nouveau repo.

## Étape 3 — Garde-fous côté GitHub Web

Avant d'initialiser le repo, activer les deux toggles côté `https://github.com/settings/emails` :

| Toggle | État cible | Effet |
|---|---|---|
| *Keep my email addresses private* | ON | GitHub masque l'email perso dans toutes les API publiques et propose l'email no-reply |
| *Block command line pushes that expose my email* | ON | GitHub **refuse** côté serveur tout push qui contiendrait l'email perso dans un commit. Filet final infranchissable |

Le second toggle est ceinture + bretelles : si malgré tout l'email perso se glisse dans un commit, GitHub refusera le push avec un message d'erreur explicite.

## Étape 4 — Création du repo distant (Web UI VIDE)

Aller sur `https://github.com/new`. **Ne PAS cocher** :

- ❌ *Add a README file*
- ❌ *Add .gitignore*
- ❌ *Choose a license*

C'est le **piège n°2** : si on coche l'une de ces options, GitHub crée un commit auto-généré sur `main` côté remote. Au premier `git push -u origin main` depuis local, l'historique diverge (le commit auto-généré n'est pas l'ancêtre du commit local). Il faut alors faire un `git pull --rebase --allow-unrelated-histories` ou un force-push, ce qui complique la procédure et masque l'historique propre.

Le repo doit être **strictement vide** côté remote au moment du premier push.

Configurer également :

- *Repository name* : `<nom-repo>`
- *Description* : courte, sans contenu sensible
- *Public*
- *Default branch name* : `main`

## Étape 5 — Init local et push initial

```powershell
# PowerShell, dans le dossier local
cd "D:\<chemin>\<dossier>"

# Init Git
git init -b main

# Vérifier le .gitignore avant tout add
cat .gitignore
# Si absent, le créer (cf. étape 5b)

# Stage
git add .

# Sanity check : voir ce qui sera committé
git status
git diff --cached --stat

# Commit racine
git commit -m "docs: <description courte sans contenu sensible>"

# Lien vers le remote
git remote add origin https://github.com/<owner>/<repo>.git

# Push initial
git push -u origin main
```

### Étape 5b — `.gitignore` minimal

Avant `git add .`, vérifier que `.gitignore` exclut au minimum :

```gitignore
# Secrets génériques (PAS de pattern qui inclut le domaine sensible !)
**/secret_*
**/.env
**/.env.*
**/*.key
**/*.pem
**/credentials.*

# OS
.DS_Store
Thumbs.db
desktop.ini

# IDE
.vscode/
.idea/
*.swp
*.swo

# Build
node_modules/
__pycache__/
*.pyc
dist/
build/
```

C'est le **piège n°3** : un pattern comme `**/<domaine>*` dans `.gitignore` publie paradoxalement le domaine sensible dans le `.gitignore` lui-même, qui sera committé et public. Toujours utiliser des **patterns génériques** (`**/secret_*`, `**/credentials.*`) plutôt que des patterns nominatifs.

## Étape 6 — Authentification OAuth Browser

Au premier `git push`, Git Credential Manager (GCM) ouvre une popup native Windows :

```
Git Credential Manager
┌────────────────────────────────────┐
│  Sign in to GitHub                 │
│                                    │
│  ○ Browser  (recommandé)           │
│  ○ Device code                     │
│  ○ Personal access token           │
└────────────────────────────────────┘
```

Choisir **Browser**. Brave (ou navigateur par défaut) s'ouvre sur la page d'autorisation GitHub :

```
Authorize git-ecosystem
This application will be able to:
- Read all user profile data
- Read user email addresses (read-only)
- Full access to private repositories
[Authorize git-ecosystem]
```

Cliquer *Authorize git-ecosystem*. Une page de succès apparaît, le navigateur peut être fermé.

Côté terminal, le `git push` reprend silencieusement. C'est le **piège n°4** : `git push -u origin main` peut afficher `Everything up-to-date` si on a tapé une seconde fois la commande pendant la popup OAuth, alors que le premier push a en réalité réussi pendant la fenêtre. Vérifier sur la page du repo GitHub que les fichiers sont bien apparus, plutôt que de se fier au seul retour terminal.

Le token OAuth obtenu est stocké dans **Windows Credential Manager** (Panneau de configuration → Comptes d'utilisateurs → Gestionnaire d'identification). Les pushes suivants seront silencieux jusqu'à expiration ou révocation côté GitHub Settings → Applications.

## Étape 7 — Vérification post-publication

| Vérif | Action | Critère OK |
|---|---|---|
| Fichiers visibles | Ouvrir `https://github.com/<owner>/<repo>` | Tous les fichiers attendus apparaissent |
| Email caviardé dans `git log` | `git log -1 --format='%ae'` côté local + clic sur le commit côté GitHub Web | Email affiché = `<id>+<user>@users.noreply.github.com` |
| Aucune fuite résiduelle | Refaire le triple `grep` sur le commit poussé : `git show HEAD | grep -E "<patterns sensibles>"` | Zéro match |
| License visible | GitHub doit reconnaître la License (badge sur la page repo) | License MIT (ou autre) affichée correctement |
| Branche `main` par défaut | Settings → General → Default branch | `main` |
| `.gitignore` propre | Ouvrir `.gitignore` côté GitHub Web | Pas de pattern nominatif sensible visible |

## Limitations connues de cette procédure

- **Rotation de token GCM** : si le token OAuth expire (typiquement 6-12 mois), Brave rouvre une popup au prochain push. Renouvellement transparent mais nécessite que le navigateur par défaut soit accessible au moment du push.
- **Repos privés convertis en publics** : si on convertit un repo privé existant en public **après** y avoir poussé du contenu sensible, l'historique privé devient public. Pour ce cas-là : repo neuf + cherry-pick des bons fichiers + abandon de l'ancien repo.
- **Force-push après publication** : un `git push --force` sur `main` après publication peut briser les forks et les clones existants. À éviter sauf urgence sécu (dans ce cas : faire un `force-push` immédiat + créer une issue publique de notification + révoquer les éventuels secrets fuités).

## Ce que l'on a évité grâce à ces garde-fous

- L'email Gmail personnel n'apparaît dans aucun commit public, présent ou futur (toggle GitHub *« Block command line pushes that expose my email »* refuserait le push).
- Le domaine interne, le hostname machine, les paths nominatifs WSL2, les IPs LAN, et les session IDs internes ont été substitués avant le premier `git add` (caviardage triple grep).
- L'historique du remote est propre dès le commit racine (Web UI VIDE → pas de divergence à réconcilier au premier push).
- Le `.gitignore` ne révèle aucun pattern nominatif sensible (patterns génériques uniquement).

## Crédits

- Procédure inspirée par : [GitHub Docs — Setting your commit email address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address) + [Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager).
- Pièges 1-4 documentés ci-dessus identifiés à chaud lors de la publication initiale.

## Voir aussi

- [README](../README.md) — vue d'ensemble du cookbook
- [`journey-s57-s63.md`](journey-s57-s63.md) — récit chronologique du parcours technique amont
- [`audit-methodologique.md`](audit-methodologique.md) — pattern d'audit niveau 2 réutilisable

---

*Fin de publication-s64.md — 1.0 — 26 avril 2026*
