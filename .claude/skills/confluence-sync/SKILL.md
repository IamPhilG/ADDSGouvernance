# /confluence-sync

Met à jour les fichiers Markdown du repo local en fonction des changements sur Confluence.
N'écrase que les pages qui ont été modifiées sur Confluence depuis le dernier export.
Supporte un mode `--dry-run` pour prévisualiser sans modifier les fichiers.

## Usage

```
/confluence-sync --space <CLE> [--page <titre-ou-id>] [--output <dossier>] [--dry-run] [--new] [--force]
```

**Paramètres**
- `--space <CLE>` (requis) : clé de l'espace Confluence (ex: `DEV`, `RH`, `PROJ`)
- `--page <titre-ou-id>` (optionnel) : page racine à synchroniser avec ses enfants. Sans ce paramètre, synchronise tout l'espace.
- `--output` (optionnel, défaut: `./output`) : dossier local contenant les `.md`
- `--dry-run` (optionnel) : affiche ce qui serait mis à jour sans modifier aucun fichier
- `--new` (optionnel) : inclut aussi les pages nouvelles (non encore dans le repo). Par défaut, seules les pages existantes sont mises à jour.
- `--force` (optionnel) : force la mise à jour de toutes les pages, même si elles semblent à jour

## Instructions

> Prérequis : serveur MCP Atlassian configuré dans Claude Code.
> Outils attendus : `confluence_get_space`, `confluence_get_page`, `confluence_get_page_children`
> (adapte les noms exacts selon le serveur MCP installé)

### Étape 0 — Vérifications préalables

1. Vérifie que les outils MCP Atlassian sont disponibles.
   - Si absents : informe l'utilisateur de configurer le serveur MCP et arrête-toi.
2. Vérifie que le dossier `<output>/<SPACE>/` existe.
   - Si absent et `--new` absent : informe l'utilisateur de lancer `/confluence-export` d'abord et arrête-toi.
   - Si absent et `--new` présent : crée le dossier et procède comme un export initial.
3. Si `--dry-run` : affiche `[MODE SIMULATION] Aucun fichier ne sera modifié.`

### Étape 1 — Inventaire Confluence

1. Si `--page` fourni : récupère cette page comme racine.
   Sinon : récupère la page racine de l'espace `--space`.

2. **Parcours récursif** de la hiérarchie Confluence :
   - Pour chaque page, récupère ses métadonnées : titre, identifiant, date de dernière modification, auteur.
   - Construis un dictionnaire `confluence_pages` :
     ```
     { chemin_attendu: { id, titre, date_modif, auteur } }
     ```
   - Affiche la progression : `Confluence : N pages inventoriées…`

### Étape 2 — Calcul des mises à jour nécessaires

Pour chaque page dans `confluence_pages` :

1. Calcule le chemin local attendu : `<output>/<chemin_attendu>`
2. Si le fichier local n'existe pas :
   - Si `--new` : marquer comme **à créer**
   - Sinon : ignorer
3. Si le fichier local existe :
   - Extrais la date de modification du fichier local (depuis le frontmatter `**Dernière modification :**`)
   - Si `--force` OU date Confluence > date locale : marquer comme **à mettre à jour**
   - Sinon : marquer comme **à jour**, ignorer
4. Affiche le plan :
   ```
   Pages à mettre à jour : N
   Pages à créer : N (si --new)
   Pages déjà à jour : N
   ```
5. Si `--dry-run` : affiche la liste détaillée et arrête-toi ici.

### Étape 3 — Mise à jour des fichiers

Pour chaque page marquée **à mettre à jour** ou **à créer** :

1. Récupère le contenu complet via MCP (corps + métadonnées : titre, auteur, date de modification).

2. **Convertis en Markdown** (mêmes règles que `/confluence-export`) :
   - Titres (`h1`…`h6`) → `#`…`######`
   - Gras / italique / code inline → `**` / `*` / `` ` ``
   - Blocs de code → ` ```langage ``` `
   - Tableaux → tables Markdown `|`
   - Listes ordonnées / non ordonnées → `1.` / `-`
   - Liens internes Confluence → liens relatifs vers le `.md` correspondant
   - Macros non convertibles → `> [Macro Confluence : <nom>]`
   - Images embarquées → `[IMAGE: <nom-fichier>]`

3. Crée les dossiers intermédiaires si nécessaire.

4. **Écris ou remplace le `.md`** :

```markdown
# <Titre de la page>

> **Espace :** `<SPACE>`
> **Chemin Confluence :** `Espace › Parent › Page`
> **Dernière modification :** <date Confluence>
> **Auteur :** <auteur>
> **Synchronisé le :** <date de la synchronisation>

---

<contenu converti>

---

*Synchronisé depuis Confluence via Claude Code*
```

5. Affiche : `✓ Mis à jour : <titre> → <chemin>`

### Étape 4 — Mise à jour du README

Met à jour `<output>/<SPACE>/README.md` :
- Actualise la date de génération
- Actualise le nombre de pages
- Ajoute les nouvelles pages à l'arborescence (si `--new`)
- Ne retire pas les pages supprimées de Confluence (signale-les seulement)

Si des pages locales n'ont plus de correspondance sur Confluence, ajoute une section :

```markdown
## Pages archivées (supprimées de Confluence)

> Ces fichiers sont conservés localement mais n'existent plus sur Confluence.

- `<SPACE>/Parent/Page.md` (dernier export : <date>)
```

### Résumé final

- Nombre de pages mises à jour / total
- Nombre de pages créées (si `--new`)
- Nombre de pages à jour (non modifiées)
- Nombre de pages orphelines (locales sans correspondance Confluence)
- Liste des fichiers modifiés avec leur chemin
- Suggère `/confluence-diff` pour un rapport complet si des orphelines sont détectées
