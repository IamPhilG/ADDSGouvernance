# /ocr-pipeline

Transcrit des images en fichiers Markdown via vision native Claude Code.
Local par défaut. Option `--github` pour pousser sur un repo GitHub (privé par défaut).

## Usage

```
/ocr-pipeline [--images <dossier>] [--output <dossier>] [--github --repo <nom>] [--public]
```

**Paramètres**
- `--images` (optionnel, défaut: `./images`) : dossier contenant les images
- `--output` (optionnel, défaut: `./output`) : dossier de sortie pour les `.md`
- `--github` (optionnel) : active le push GitHub (nécessite `--repo` et `GITHUB_TOKEN`)
- `--repo <nom>` (requis avec `--github`) : nom du repo GitHub à créer
- `--public` (optionnel) : rend le repo public (privé par défaut)

## Instructions

### Étape 0 — Vérifications préalables

1. Liste les images dans `--images` (extensions : `.png`, `.jpg`, `.jpeg`, `.webp`, `.gif`).
   - Si aucune image : informe l'utilisateur et arrête-toi.
2. Crée `--output` s'il n'existe pas.
3. Si `--github` : vérifie que `--repo` est présent et `$env:GITHUB_TOKEN` défini. Sinon arrête-toi.

### Étape 1 — OCR de chaque image

Pour chaque image :

1. **Lis l'image** avec ta vision. Extrais TOUT le contenu textuel avec fidélité maximale :
   - Texte EXACT (casse, ponctuation, espaces, retours à la ligne)
   - Tableaux → structure `|`
   - Formulaires → `Label: Valeur`
   - Schémas → texte des bulles + relations (`→`)
   - Visuels non textuels → `[IMAGE: description brève]`

2. **Auto-valide** : relis l'image, note la conformité /100.
   - ≥ 99 → continue
   - < 99 → ré-extrais avec feedback (max 3 tentatives)
   - Après 3 échecs → demande : accepter / donner du contexte / ignorer

3. **Écris le `.md`** dans `--output/` :

```markdown
# <nom_sans_extension>

> **Source :** `<nom_image>`
> **Score de conformité :** <score>%
> **Généré le :** <date>

---

<texte extrait>

---

*Pipeline OCR Claude Code*
```

   Nom de fichier : `<stem_sanitisé>.md` (caractères non alphanumériques → `_`)

### Étape 2 — README de synthèse

Crée `<output>/README.md` :

```markdown
# Pipeline OCR — Résultats

Généré le <date>

| Fichier | Image source | Score |
|---------|-------------|-------|
| [nom.md](./nom.md) | image.png | 99.2% |
```

### Étape 3 — Push GitHub (si `--github`)

Si `--github` absent : passe au résumé final.

```powershell
$headers = @{ Authorization = "Bearer $env:GITHUB_TOKEN"; Accept = "application/vnd.github+json" }
$user = Invoke-RestMethod -Uri "https://api.github.com/user" -Headers $headers
$username = $user.login

$isPrivate = -not $public
$body = @{ name = "<repo>"; description = "Pipeline OCR — N document(s)"; private = $isPrivate; auto_init = $false } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "https://api.github.com/user/repos" -Headers $headers -Body $body -ContentType "application/json"

cd <output>
git init
git config user.email "pipeline@local"
git config user.name "Pipeline Claude Code"
git add .
git commit -m "feat: OCR — N transcription(s)"
git branch -M main
git remote add origin "https://github.com/$username/<repo>.git"
$env:GIT_TERMINAL_PROMPT = "0"
git -c "http.extraHeader=Authorization: Bearer $env:GITHUB_TOKEN" push -u origin main
cd ..
```

Affiche l'URL du repo et le SHA du commit.

### Résumé final

- Nombre d'images traitées / total
- Score moyen de conformité
- Liste des `.md` créés dans `<output>/`
- URL du repo GitHub (si `--github`)
