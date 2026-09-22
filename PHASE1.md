# PHASE 1 — Document .docx (téléchargeable, upload manuel par l'utilisateur — pas de Drive MCP)
 
Voir `SKILL.md` pour les règles absolues, les CONSTANTES et le format de la TABLE DE CORRESPONDANCE avant de lire cette phase.
 
### Étape 1 — Lire et analyser le PDF
 
Identifier :
- Structure : titres, sections, champs, tableaux
- Images / logos : présence ou absence
- Liste complète des champs avec leurs labels exacts
- Présence de signatures ou file uploads
### Étape 2 — Gérer les images
 
- Logo fourni séparément → noter, à insérer dans le Doc
- Image dans le PDF mais non fournie → signaler dans l'aperçu, ne pas bloquer
- Pas d'image → continuer normalement
### Étape 3 — Collecter les informations manquantes
 
Vérifier que l'utilisateur a fourni :
1. **Nom du fichier** selon nomenclature `⚙️ TYPE / NOM CLIENT` (voir `SKILL.md`)
Le **dossier Drive de destination n'est PAS requis en Phase 1**. On ne fait aucun upload Drive à ce stade (voir Étape 5) ; ce dossier ne sera nécessaire qu'en Phase 3 (voir `PHASE3.md`, Étape 9), où il sert de paramètre `folderId` au module Google Docs de Make — c'est-à-dire l'endroit où Make déposera les documents *générés automatiquement* à chaque soumission Tally, pas le document blueprint lui-même.
 
Si le nom manque → demander avant de continuer.
 
### Étape 4 — Aperçu et confirmation
 
```
📄 Aperçu du document à créer
 
Nom      : ⚙️ [TYPE] / [NOM CLIENT]
Format   : .docx téléchargeable (l'utilisateur l'uploadera lui-même sur Drive)
 
Structure détectée :
- [X] section(s) / titre(s)
- [X] champ(s) de formulaire
- [X] tableau(x)
- Signatures / uploads : [oui / non]
- Images : [reproduites / ignorées / en attente]
 
✅ Souhaitez-vous créer ce document ?
```
 
❌ Ne jamais créer sans confirmation explicite.
 
### Étape 5 — Créer le document
 
⛔ **RÈGLE ABSOLUE — INTERDICTION D'UPLOAD GOOGLE DRIVE VIA MCP**
Ce skill **NE DOIT JAMAIS** utiliser le Google Drive MCP pour créer, uploader, ou déposer un document dans Drive — même si le connecteur Google Drive MCP est disponible dans la conversation. L'utilisateur gère lui-même l'upload sur Drive.
 
Procédure correcte :
1. **Générer un fichier `.docx`** (skill `docx` — voir `/mnt/skills/public/docx/SKILL.md`) reproduisant fidèlement la structure du PDF :
   - Titres H1/H2/H3, gras/italique, listes, tableaux
   - **Placeholders Make** pour chaque champ, au format `Libellé : {{placeholder-court}}` (noms courts sans accents — voir règles de nommage dans `SKILL.md`, section TABLE DE CORRESPONDANCE)
   - Pour les champs hidden fields aussi : `Chantier : {{chantier}}`, `Adresse : {{adresse}}`
   - Pour les signatures : insérer une **image blanche placeholder** à l'emplacement de la signature (l'alt-text/nom sera déterminé plus tard, une fois le doc converti en Google Doc par l'utilisateur — voir limitation connue en `PHASE3.md`)
   - Pour tout logo fourni séparément par l'utilisateur : l'insérer directement à l'emplacement prévu (remplace le placeholder visuel)
2. Sauvegarder dans `/mnt/user-data/outputs/` et **présenter le fichier via `present_files`** — jamais de lien Drive à ce stade
3. **Produire la TABLE DE CORRESPONDANCE complète** (voir format dans `SKILL.md`) — elle sera réutilisée telle quelle en Phase 3
4. Annoncer : *"Phase 1 terminée ✅ — Conserve cette table, elle servira en Phase 3. Une fois que tu as converti/uploadé ce .docx en Google Doc sur ton Drive, donne-moi son lien/ID (et le dossier Drive de destination) pour la Phase 3. Dis-moi quand tu es prêt pour la Phase 2 (Tally), ou fournis-moi la SPEC client si tu en as une."*
⚠️ Le lien du Google Doc et le dossier Drive de destination sont donc collectés **uniquement en Phase 3** (voir `PHASE3.md`, Étape 9), une fois que l'utilisateur a fait la conversion/l'upload lui-même — jamais en Phase 1.