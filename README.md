# Pipeline → Markdown

Deux skills Claude Code, pilotés par ton abonnement Pro (aucune clé API séparée).

## Skills disponibles

### `/ocr-pipeline` — Images → Markdown

Transcrit des images en fichiers Markdown via vision native Claude Code.

```text
/ocr-pipeline
/ocr-pipeline --images ./mes_scans
/ocr-pipeline --output ./resultats
/ocr-pipeline --github --repo mon-repo
/ocr-pipeline --github --repo mon-repo --public
```

**Formats :** `.png` `.jpg` `.jpeg` `.webp` `.gif`

**Contenu supporté :** texte imprimé, tableaux, formulaires, schémas, documents dégradés

**Seuil de conformité :** 99% — 3 tentatives max par image

---

### `/confluence-export` — Confluence → Markdown

Exporte des pages Confluence en Markdown via MCP Atlassian.
L'arborescence des `.md` reflète la hiérarchie Confluence.

Prérequis : serveur MCP Atlassian configuré dans Claude Code.

```text
/confluence-export --space DEV
/confluence-export --space DEV --page "Architecture technique"
/confluence-export --space RH --output ./docs-rh
/confluence-export --space DEV --github --repo confluence-export
```

Exemple d'arborescence générée :

```text
output/
  DEV/
    README.md
    Architecture/
      Architecture.md
      Backend/
        Backend.md
      Frontend/
        Frontend.md
```

---

## Push GitHub (optionnel — commun aux deux skills)

Les deux skills acceptent `--github --repo <nom>`. Le repo est **privé par défaut** (`--public` pour changer).

```powershell
$env:GITHUB_TOKEN = "ghp_..."   # scope: repo
```

### Créer un GitHub Token

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Coche : `repo`
3. Génère et copie le token
