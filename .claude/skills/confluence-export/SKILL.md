# /confluence-export

Exporte des pages Confluence en fichiers Markdown via MCP Atlassian.
L'arborescence des `.md` reflète la hiérarchie Confluence.
Local par défaut. Option `--github` pour pousser sur un repo GitHub (privé par défaut).

## Usage

```
/confluence-export --space <CLE> [--page <titre-ou-id>] [--output <dossier>] [--github --repo <nom>] [--public]
```

**Paramètres**
- `--space <CLE>` (requis) : clé de l'espace Confluence (ex: `DEV`, `RH`, `PROJ`)
- `--page <titre-ou-id>` (optionnel) : page racine à exporter avec tous ses enfants. Sans ce paramètre, exporte tout l'espace.
- `--output` (optionnel, défaut: `./output`) : dossier racine de sortie
- `--github` (optionnel) : active le push GitHub (nécessite `--repo` et `GITHUB_TOKEN`)
- `--repo <nom>` (requis avec `--github`) : nom du repo GitHub à créer
- `--public` (optionnel) : rend le repo public (privé par défaut)

## Instructions

> Prérequis : serveur MCP Atlassian configuré dans Claude Code.
> Outils attendus : `confluence_get_space`, `confluence_get_page`, `confluence_get_page_children`
> (adapte les noms exacts selon le serveur MCP installé)

### Étape 0 — Vérifications préalables

1. Vérifie que les outils MCP Atlassian sont disponibles.
   - Si absents : informe l'utilisateur de configurer le serveur MCP et arrête-toi.
2. Crée le dossier `<output>/<SPACE>/` s'il n'existe pas.
3. Si `--github` : vérifie que `--repo` est présent et `$env:GITHUB_TOKEN` défini. Sinon arrête-toi.

### Étape 1 — Récupération de l'arborescence

1. Si `--page` fourni : récupère cette page comme racine de l'export.
   Sinon : récupère la page racine de l'espace `--space`.

2. **Parcours récursif** de la hiérarchie :
   - Pour chaque page, récupère ses enfants via MCP
   - Maintiens un arbre représentant la hiérarchie complète
   - Affiche la progression : `Découverte : N pages trouvées…`

### Étape 2 — Conversion de chaque page

Pour chaque page dans l'arborescence :

1. Récupère le contenu complet via MCP (corps + métadonnées : titre, auteur, date de modification).

2. **Convertis en Markdown** :
   - Titres (`h1`…`h6`) → `#`…`######`
   - Gras / italique / code inline → `**` / `*` / `` ` ``
   - Blocs de code → ` ```langage ``` `
   - Tableaux → tables Markdown `|`
   - Listes ordonnées / non ordonnées → `1.` / `-`
   - Liens internes Confluence → liens relatifs vers le `.md` correspondant
   - Macros non convertibles → `> [Macro Confluence : <nom>]`
   - Images embarquées → `[IMAGE: <nom-fichier>]`

3. **Détermine le chemin de sortie** en reflétant la hiérarchie :
   ```
   <output>/<SPACE>/<Parent>/<Enfant>/<Page>.md
   ```
   Sanitise chaque segment (caractères non alphanumériques → `_`).

4. Crée les dossiers intermédiaires si nécessaire et **écris le `.md`** :

```markdown
# <Titre de la page>

> **Espace :** `<SPACE>`
> **Chemin Confluence :** `Espace › Parent › Page`
> **Dernière modification :** <date>
> **Auteur :** <auteur>

---

<contenu converti>

---

*Exporté depuis Confluence via Claude Code*
```

### Étape 3 — README de synthèse

Crée `<output>/<SPACE>/README.md` avec l'arborescence complète :

```markdown
# Export Confluence — <SPACE>

Généré le <date> | <N> pages exportées

## Arborescence

- [Page racine](./Page_racine.md)
  - [Enfant 1](./Page_racine/Enfant_1.md)
    - [Petit-enfant](./Page_racine/Enfant_1/Petit_enfant.md)
  - [Enfant 2](./Page_racine/Enfant_2.md)
```

### Étape 4 — Push GitHub (si `--github`)

Si `--github` absent : passe au résumé final.

```powershell
$headers = @{ Authorization = "Bearer $env:GITHUB_TOKEN"; Accept = "application/vnd.github+json" }
$user = Invoke-RestMethod -Uri "https://api.github.com/user" -Headers $headers
$username = $user.login

$isPrivate = -not $public
$body = @{ name = "<repo>"; description = "Export Confluence <SPACE> — N pages"; private = $isPrivate; auto_init = $false } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "https://api.github.com/user/repos" -Headers $headers -Body $body -ContentType "application/json"

cd <output>
git init
git config user.email "pipeline@local"
git config user.name "Pipeline Claude Code"
git add .
git commit -m "feat: export Confluence <SPACE> — N pages"
git branch -M main
git remote add origin "https://github.com/$username/<repo>.git"
$env:GIT_TERMINAL_PROMPT = "0"
git -c "http.extraHeader=Authorization: Bearer $env:GITHUB_TOKEN" push -u origin main
cd ..
```

Affiche l'URL du repo et le SHA du commit.

### Résumé final

- Nombre de pages exportées / total découvert
- Arborescence des `.md` créés dans `<output>/<SPACE>/`
- URL du repo GitHub (si `--github`)
