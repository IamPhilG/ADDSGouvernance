# /confluence-diff

Compare les pages Confluence avec les fichiers Markdown du repo local.
Identifie les pages nouvelles, modifiées ou supprimées côté Confluence par rapport au repo.
Produit un rapport de différences lisible.

## Usage

```
/confluence-diff --space <CLE> [--page <titre-ou-id>] [--output <dossier>] [--report <fichier>]
```

**Paramètres**
- `--space <CLE>` (requis) : clé de l'espace Confluence (ex: `DEV`, `RH`, `PROJ`)
- `--page <titre-ou-id>` (optionnel) : page racine à comparer avec ses enfants. Sans ce paramètre, compare tout l'espace.
- `--output` (optionnel, défaut: `./output`) : dossier local contenant les `.md` exportés depuis Confluence
- `--report` (optionnel, défaut: `./output/<SPACE>/diff-report.md`) : chemin du rapport de différences

## Instructions

> Prérequis : serveur MCP Atlassian configuré dans Codex.
> Outils attendus : `confluence_get_space`, `confluence_get_page`, `confluence_get_page_children`
> (adapte les noms exacts selon le serveur MCP installé)

### Étape 0 — Vérifications préalables

1. Vérifie que les outils MCP Atlassian sont disponibles.
   - Si absents : informe l'utilisateur de configurer le serveur MCP et arrête-toi.
2. Vérifie que le dossier `<output>/<SPACE>/` existe.
   - Si absent : informe l'utilisateur de lancer `/confluence-export` d'abord et arrête-toi.

### Étape 1 — Inventaire Confluence

1. Si `--page` fourni : récupère cette page comme racine.
   Sinon : récupère la page racine de l'espace `--space`.

2. **Parcours récursif** de la hiérarchie Confluence :
   - Pour chaque page, récupère ses métadonnées via MCP : titre, identifiant, date de dernière modification, auteur.
   - Construis un dictionnaire `confluence_pages` :
     ```
     { chemin_attendu: { id, titre, date_modif, auteur } }
     ```
     Le `chemin_attendu` suit la convention de `/confluence-export` :
     `<SPACE>/<Parent>/<Enfant>/<Page>.md` (segments sanitisés).
   - Affiche la progression : `Confluence : N pages inventoriées…`

### Étape 2 — Inventaire du repo local

1. Liste récursivement tous les fichiers `.md` dans `<output>/<SPACE>/` (sauf `README.md` et `diff-report.md`).
2. Pour chaque fichier, extrais depuis le frontmatter Markdown :
   - La date de dernière modification (`**Dernière modification :**`)
   - L'auteur (`**Auteur :**`)
3. Construis un dictionnaire `local_pages` :
   ```
   { chemin_relatif: { date_modif, auteur } }
   ```
4. Affiche : `Repo local : N fichiers inventoriés.`

### Étape 3 — Calcul des différences

Compare `confluence_pages` et `local_pages` :

1. **Pages nouvelles sur Confluence** (présentes dans `confluence_pages`, absentes dans `local_pages`) :
   - Classer par chemin
   - Signaler : titre, chemin attendu, date de modification Confluence

2. **Pages supprimées de Confluence** (absentes de `confluence_pages`, présentes dans `local_pages`) :
   - Signaler : chemin local, date de modification locale

3. **Pages modifiées** (présentes dans les deux, date Confluence > date locale) :
   - Signaler : titre, chemin, date locale, date Confluence, auteur de la modification

4. **Pages à jour** (présentes dans les deux, dates identiques ou date locale ≥ date Confluence) :
   - Compter seulement, ne pas lister en détail.

### Étape 4 — Rapport de différences

Écris le rapport dans `--report` :

```markdown
# Rapport de différences — <SPACE>

Généré le <date>
Comparaison : Confluence vs `<output>/<SPACE>/`

## Résumé

| Statut | Nombre |
|--------|--------|
| Nouvelles pages (Confluence → repo) | N |
| Pages supprimées (plus sur Confluence) | N |
| Pages modifiées (Confluence plus récent) | N |
| Pages à jour | N |
| **Total Confluence** | N |
| **Total repo local** | N |

---

## Nouvelles pages sur Confluence

> Ces pages existent sur Confluence mais n'ont pas encore été exportées dans le repo.
> Lancez `/confluence-sync` ou `/confluence-export` pour les récupérer.

| Titre | Chemin attendu | Modifié le | Auteur |
|-------|---------------|------------|--------|
| Titre page | `SPACE/Parent/Page.md` | 2024-01-15 | Jean Dupont |

---

## Pages supprimées de Confluence

> Ces fichiers locaux n'ont plus de correspondance sur Confluence.
> Vérifiez s'ils peuvent être archivés ou supprimés.

| Fichier local | Dernière modif locale |
|---------------|-----------------------|
| `SPACE/Parent/Page.md` | 2023-12-01 |

---

## Pages modifiées sur Confluence

> Confluence est plus récent que le repo local.
> Lancez `/confluence-sync` pour mettre à jour ces fichiers.

| Titre | Fichier local | Modifié (local) | Modifié (Confluence) | Auteur |
|-------|--------------|-----------------|----------------------|--------|
| Titre page | `SPACE/Parent/Page.md` | 2023-11-01 | 2024-01-20 | Jean Dupont |

---

## Pages à jour

N pages sont synchronisées entre Confluence et le repo local.
```

### Résumé final

- Affiche un tableau récapitulatif en console (nouvelles / supprimées / modifiées / à jour)
- Indique le chemin du rapport généré
- Suggère `/confluence-sync` si des pages sont nouvelles ou modifiées
