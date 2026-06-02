# /confluence-images

Copie les images (pièces jointes) des pages Confluence dans le repo local.
L'arborescence des images reflète la hiérarchie Confluence.
Local par défaut. Option `--github` pour pousser sur un repo GitHub (privé par défaut).

## Usage

```
/confluence-images --space <CLE> [--page <titre-ou-id>] [--output <dossier>] [--github --repo <nom>] [--public]
```

**Paramètres**
- `--space <CLE>` (requis) : clé de l'espace Confluence (ex: `DEV`, `RH`, `PROJ`)
- `--page <titre-ou-id>` (optionnel) : page racine à parcourir avec tous ses enfants. Sans ce paramètre, parcourt tout l'espace.
- `--output` (optionnel, défaut: `./images`) : dossier racine de sortie
- `--github` (optionnel) : active le push GitHub (nécessite `--repo` et `GITHUB_TOKEN`)
- `--repo <nom>` (requis avec `--github`) : nom du repo GitHub à créer
- `--public` (optionnel) : rend le repo public (privé par défaut)

## Instructions

> Prérequis : serveur MCP Atlassian configuré dans Codex.
> Outils attendus : `confluence_get_space`, `confluence_get_page`, `confluence_get_page_children`, `confluence_get_page_attachments`
> (adapte les noms exacts selon le serveur MCP installé)

### Étape 0 — Vérifications préalables

1. Vérifie que les outils MCP Atlassian sont disponibles.
   - Si absents : informe l'utilisateur de configurer le serveur MCP et arrête-toi.
2. Crée le dossier `<output>/<SPACE>/` s'il n'existe pas.
3. Si `--github` : vérifie que `--repo` est présent et `$env:GITHUB_TOKEN` défini. Sinon arrête-toi.

### Étape 1 — Récupération de l'arborescence

1. Si `--page` fourni : récupère cette page comme racine du parcours.
   Sinon : récupère la page racine de l'espace `--space`.

2. **Parcours récursif** de la hiérarchie :
   - Pour chaque page, récupère ses enfants via MCP
   - Maintiens un arbre représentant la hiérarchie complète
   - Affiche la progression : `Découverte : N pages trouvées…`

### Étape 2 — Téléchargement des images de chaque page

Pour chaque page dans l'arborescence :

1. Récupère la liste des pièces jointes via MCP (`confluence_get_page_attachments` ou équivalent).
2. Filtre uniquement les fichiers image : extensions `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.svg`, `.bmp`.
3. Pour chaque image :
   - Télécharge le contenu binaire via MCP ou URL d'attachment.
   - **Détermine le chemin de sortie** en reflétant la hiérarchie de la page :
     ```
     <output>/<SPACE>/<Parent>/<Enfant>/<nom-image>.<ext>
     ```
     Sanitise chaque segment de dossier (caractères non alphanumériques → `_`). Conserve le nom de fichier original de l'image.
   - Crée les dossiers intermédiaires si nécessaire.
   - Enregistre le fichier image.
   - Si une image du même nom existe déjà : compare la taille ou le hash — si identique, saute ; sinon remplace et signale.

4. Affiche la progression : `Page "<titre>" : N image(s) copiée(s).`

### Étape 3 — Fichier d'index

Crée `<output>/<SPACE>/images-index.md` avec le récapitulatif :

```markdown
# Images Confluence — <SPACE>

Généré le <date> | <N> images copiées depuis <P> pages

## Index

| Image | Page Confluence | Chemin local |
|-------|----------------|--------------|
| nom.png | [Titre page](chemin/Page.md) | `<SPACE>/Parent/Page/nom.png` |
```

### Étape 4 — Push GitHub (si `--github`)

Si `--github` absent : passe au résumé final.

```powershell
$headers = @{ Authorization = "Bearer $env:GITHUB_TOKEN"; Accept = "application/vnd.github+json" }
$user = Invoke-RestMethod -Uri "https://api.github.com/user" -Headers $headers
$username = $user.login

$isPrivate = -not $public
$body = @{ name = "<repo>"; description = "Images Confluence <SPACE> — N fichiers"; private = $isPrivate; auto_init = $false } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "https://api.github.com/user/repos" -Headers $headers -Body $body -ContentType "application/json"

cd <output>
git init
git config user.email "pipeline@local"
git config user.name "Pipeline Codex"
git add .
git commit -m "feat: images Confluence <SPACE> — N fichiers"
git branch -M main
git remote add origin "https://github.com/$username/<repo>.git"
$env:GIT_TERMINAL_PROMPT = "0"
git -c "http.extraHeader=Authorization: Bearer $env:GITHUB_TOKEN" push -u origin main
cd ..
```

Affiche l'URL du repo et le SHA du commit.

### Résumé final

- Nombre d'images copiées / total détecté
- Nombre de pages parcourues
- Nombre d'images ignorées (déjà à jour)
- Arborescence des images dans `<output>/<SPACE>/`
- URL du repo GitHub (si `--github`)
