# PHASE 3 — Scénario Make.com
 
Voir `SKILL.md` pour les règles absolues, les CONSTANTES et la TABLE DE CORRESPONDANCE avant de lire cette phase. Si cette phase démarre sans que les Phases 1/2 aient été faites via ce skill, voir `SKILL.md` section "Démarrer depuis n'importe quelle phase".
 
### Déclenchement
 
Démarre quand l'utilisateur confirme être prêt après la Phase 2.
 
### Étape 9 — Collecter les informations manquantes
 
Vérifier que l'utilisateur a fourni :
1. **Lien du GDoc blueprint** (document Phase 1 après conversion manuelle) → sinon demander
2. **Dossier Drive de destination** pour les documents générés à chaque soumission (paramètre `folderId` du module Google Docs) → sinon demander. C'est différent du document blueprint lui-même : ce dossier reçoit les *copies générées automatiquement*, pas le blueprint.
### Étape 10 — Identifier le numéro du module Lambda
 
Le numéro du module HTTP Lambda n'est pas fixe — il change selon le scénario. Il ne faut **jamais supposer** une valeur fixe.
 
Procédure obligatoire :
1. Récupérer le blueprint du scénario nouvellement créé via `Make:scenarios_get`
2. Parcourir le `flow` et trouver le module `http:MakeRequest` dont le `mapper.url` contient `lambda-url.eu-west-1.on.aws`
3. Extraire son `id` → appeler ce numéro `LAMBDA_ID`
4. Utiliser `{{LAMBDA_ID.data.xxx}}` dans **tout** le mapping `requests` et `image`
### Étape 11 — Construire le mapping depuis la table de correspondance
 
Reprendre la table produite en Phase 1 (ou reconstruite) et construire avec `LAMBDA_ID` :
 
**`requests`** (champs texte/date/choix) — clé = placeholder GDoc, valeur selon type :
- Texte / Nombre / Choix : `{{LAMBDA_ID.data.\`Label Tally\`}}`
- Date : `{{if(LAMBDA_ID.data.\`Label Tally\`; formatDate(LAMBDA_ID.data.\`Label Tally\`; "DD/MM/YYYY"))}}`
- **Incrémentation automatique** : voir section dédiée ci-dessous
- **Dropdown / cases à cocher** : voir pattern ci-dessous
**Dropdown Tally → case à cocher dans le GDoc (pattern Formubees) :**
 
⚠️ **Rappel — Make est indexé en base 1** (voir CONSTANTES dans `SKILL.md`) : `[1]` = 1er élément, jamais `[0]`.
 
Les champs dropdown Tally renvoient un array. Pour afficher ☑ ou ☐ dans le GDoc, on compare le **premier élément de l'array** (`[1]`) à la valeur attendue (ex : `"Oui"`).
 
Syntaxe Make :
```
{{if(LAMBDA_ID.data.`Label Tally`[1] = "Valeur attendue"; "☑"; "☐")}}
```
 
Exemple réel pour "Facturable ?" dont la première option est "Oui" :
```
{{if(LAMBDA_ID.data.`Facturable ?`[1] = "Oui"; "☑"; "☐")}}
```
 
Règles :
- `[1]` = premier élément de l'array (la valeur sélectionnée)
- La valeur attendue est la **première option du dropdown** (ex : `"Oui"`, `"Terminé"`, etc.) — c'est la valeur qui affichera ☑
- Si le champ n'est pas rempli ou si une autre option est sélectionnée → ☐
- S'applique à **tous les dropdowns Tally** quel que soit le label (Oui/Non, Terminé/Partiel, etc.)
- Dans la table de correspondance, noter le type `DROPDOWN` et la valeur-cible pour chaque champ
**`image`** (signatures et file uploads) — clé = nom alt de l'image placeholder dans le GDoc :
 
⚠️ Syntaxe exacte : `[]` **après** le backtick fermant du label, jamais à l'intérieur. Forme générique (remplacer `<Label Tally>` par le label exact, et l'URL par la valeur en CONSTANTES) :
```
{{if(LAMBDA_ID.data.`<Label Tally>`[]; LAMBDA_ID.data.`<Label Tally>`[]; "<Image placeholder URL — voir CONSTANTES dans SKILL.md>")}}
```
 
⚠️ **Limitation connue** : Google Docs assigne ses propres identifiants internes (`kix.xxxx-t.0`) à chaque image une fois le doc réellement généré — impossibles à prédire via l'API avant la première génération. Le module Google Docs de Make apparaîtra donc probablement **vide** sur le champ `image` juste après la création du scénario : c'est normal, il faudra rouvrir le module dans l'interface Make et resélectionner les images manuellement (voir Cas particuliers dans `SKILL.md`).
 
---
 
## Incrémentation automatique (si demandée)
 
Ajouter 2 modules `datastore` entre le Lambda et le module Google Docs :
 
**Module A — Rechercher le dernier numéro (`datastore:SearchRecord`)**
- Datastore : voir CONSTANTES dans `SKILL.md` (ID `68679`, "Formubees : No de Fiche")
- Filtre (ET) :
  - `Type de rapport` = `"[TYPE]"` (ex : `"BDC"`, `"RDI"`…) — opérateur `text:equal`
  - `company_id` = `{{2.user.company_id}}` — opérateur `text:equal` (⚠️ vient du webhook, champ `user.company_id` du module 2, **pas** du Lambda)
- `continueWhenNoRes` : `false`
**Module B — Incrémenter et sauvegarder (`datastore:UpdateRecord`)**
- Datastore : voir CONSTANTES (ID `68679`)
- Key : `{{ModuleA.data.company_id}}-[TYPE]` (ex : `85383aad-9a39-4fa0-9130-064e36cd0382-BDC`)
- Data :
  - `Number` : `{{ModuleA.data.Number + 1}}`
  - `company_id` : laissé vide (déjà présent dans la clé)
  - `Type de rapport` : laissé vide (déjà présent dans la clé)
- `upsert` : `true`
**Dans le GDoc :**
- Ajouter un placeholder (ex : `{{numero}}`) mappé dans `requests` à `{{ModuleA.data.Number}}`
⚠️ **Piège classique** : on utilise le numéro renvoyé par le **Module A** (celui trouvé *avant* incrémentation) pour le document en cours — le **Module B** calcule et sauvegarde le numéro pour la *prochaine* soumission. Ne jamais mapper le GDoc sur `{{ModuleB.data.Number}}`.
 
Le `[TYPE]` dans le filtre et dans la clé doit correspondre exactement à l'abréviation du document (`BDC`, `RDI`, `PV`, etc.) — à adapter par client/document.
 
*(Reconstruit le 2026-09-10 à partir du scénario de référence réel `6718018` (voir CONSTANTES), modules `43`/`44` — la version précédente de cette section était corrompue, voir Changelog dans `SKILL.md`.)*
 
---
 
### Étape 12 — Aperçu Make et confirmation
 
```
⚙️ Aperçu du scénario Make à créer
 
Nom              : ⚙️ Formubees / [TYPE] / [NOM CLIENT]
Team ID          : (voir CONSTANTES)
Dossier Make     : FORMUBEES CLIENTS (voir CONSTANTES)
Webhook          : Formubees / [TYPE] / [NOM CLIENT]
Connexion Alobees, Destinataire email, Gmail/Google Docs Connection : (voir CONSTANTES dans SKILL.md)
GDoc blueprint   : [ID extrait du lien]
Dossier Drive    : [ID extrait du lien]
Module Lambda    : ID détecté automatiquement après création
 
requests (depuis table Phase 1) :
  [placeholder] → {{LAMBDA_ID.data.`Label Tally`}}
  …
 
image (signatures) :
  [nom-alt-image] → {{if(LAMBDA_ID.data.`Label`[]; ...)}}
 
Branche photos : [oui si FILE_UPLOAD / non]
Incrémentation : [oui / non]
 
✅ Souhaitez-vous créer ce scénario ?
```
 
❌ Ne jamais créer sans confirmation.
 
### Étape 13 — Créer le webhook Make
 
`Make:hooks_create` — ✅ création uniquement.
 
- Nom : `Formubees / [TYPE] / [NOM CLIENT]`
- Team ID : voir CONSTANTES
- data : `{"headers": [], "method": "post", "stringify": false}`
### Étape 14 — Créer le scénario Make
 
⚠️ **Le scénario de référence est une source de vérité vivante, pas une simple étiquette documentaire.**
 
Avant de construire le blueprint, **toujours** appeler `Make:scenarios_get` sur le scénario de référence (voir CONSTANTES dans `SKILL.md`) et l'inspecter réellement :
- Comparer chaque type de module utilisé (`gateway:CustomWebHook`, `http:MakeRequest`, `google-docs:createADocumentFromTemplate`, `app#alobees-3d4e-htqbwt:formubees`, `google-email:sendAnEmail`, `datastore:SearchRecord`/`UpdateRecord`, etc.) à ce qui est décrit dans ce skill.
- Vérifier que les IDs de connexion (`__IMTCONN__`), le Team ID, le Datastore ID, et la structure des modules Alobees correspondent encore à ce qui est écrit dans CONSTANTES.
- Toute valeur écrite en dur dans ce skill (IDs, connexions, structure de modules) est **provisoire** tant qu'elle n'a pas été confirmée contre le scénario de référence live — le skill peut devenir obsolète, le scénario réel non. En cas d'écart, faire confiance au scénario réel et signaler l'écart à l'utilisateur (voir Changelog dans `SKILL.md`).
- Ce n'est pas une étape facultative même si le skill semble déjà donner toutes les valeurs nécessaires : deux erreurs réelles rencontrées (connexion Alobees `63789` invalide, section incrémentation corrompue) auraient été détectées immédiatement par cette simple vérification.
Une fois vérifié, `Make:scenarios_create` basé sur le blueprint du scénario de référence — ✅ création uniquement.
 
⚠️ Ce scénario de référence a remplacé l'ancien template `6404196` le 29/07/2026. Le module mail (`google-email:sendAnEmail`) y est simplifié — utiliser impérativement ce nouveau format HTML (voir "Format mail — Étape 14bis" ci-dessous) et non l'ancien format. *(Cette information elle-même doit être reconfirmée contre le scénario réel si elle semble dater.)*
 
#### Format mail — Étape 14bis (obligatoire, remplace tout ancien format)
 
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
</head>
<body style="font-family: Arial, sans-serif; background: #f0e8e8; margin: 0; padding: 20px;">
  <div style="max-width: 600px; margin: 0 auto; background: white; padding: 30px; border-radius: 8px;">
    <img src="https://img.mailinblue.com/4317030/images/content_library/original/6800c9028f20a4205acfe48a.jpg" width="200" alt="Logo" style="display: block; margin: 0 auto 20px;">
    
    <h1 style="color: #1f2d3d; font-size: 28px; margin: 0 0 20px 0; text-align: center;">Votre [TYPE DOCUMENT] est prêt</h1>
    
    <div style="color: #3b3f44; margin: 15px 0; text-align: center;">
      <p><strong>Chantier :</strong> {{LAMBDA_ID.data.siteName}}</p>
      <p><strong>Date :</strong> {{formatDate(now; "DD/MM/YYYY")}}</p>
    </div>
    
    <div style="text-align: center; margin-top: 30px;">
      <a href="https://docs.google.com/document/u/0/export?format=pdf&id={{DOC_MODULE_ID.id}}&includes_info_params=true&usp=drive_web&cros_files=false&tab=t.0&inspectorResult=%7B%22pc%22%3A2%2C%22lplc%22%3A4%7D" style="padding: 12px 30px; border-radius: 6px; text-decoration: none; font-weight: bold; display: inline-block; background: #284381; color: white; border: 3px solid #274382; margin: 5px;">Télécharger le rapport</a>
      <a href="{{DOC_MODULE_ID.webViewLink}}" style="padding: 12px 30px; border-radius: 6px; text-decoration: none; font-weight: bold; display: inline-block; background: #fbfcff; color: #274382; border: 3px solid #284381; margin: 5px;">Modifier le rapport</a>
    </div>
    
    <img src="https://img.mailinblue.com/4317030/images/content_library/original/680b47245b5ed8e012ea22d6.png" width="100%" alt="" style="margin-top: 30px;">
  </div>
</body>
</html>
```
 
Règles :
- `[TYPE DOCUMENT]` = nom complet du document (ex : "Bon de commande", "Rapport d'intervention")
- `{{LAMBDA_ID.data.siteName}}` : remplacer `LAMBDA_ID` par le vrai module Lambda détecté à l'Étape 10
- `{{DOC_MODULE_ID.id}}` et `{{DOC_MODULE_ID.webViewLink}}` : remplacer `DOC_MODULE_ID` par l'ID du module `google-docs:createADocumentFromTemplate` (à vérifier dans le scénario généré, pas supposé fixe)
- Objet du mail : `Votre document {{DOC_MODULE_ID.name}} est prêt`
- Destinataire (`to`) : voir CONSTANTES dans `SKILL.md` — fixe, ne jamais demander à l'utilisateur
- Ce module mail fait partie de la branche "sans photo" du router (branche 2) ; il suit toujours les modules HTTP Alobees (POST document + POST create post)
**Valeurs à adapter par client :**
 
| Paramètre | Source |
|---|---|
| Nom du scénario | `⚙️ Formubees / TYPE / NOM CLIENT` |
| Hook ID | ID créé à l'Étape 13 |
| Webhook label | `Formubees / TYPE / NOM CLIENT` |
| `document` (GDoc ID) | Extrait du lien GDoc fourni |
| `folderId` (Drive destination) | Extrait du lien Drive fourni |
| `requests` | Table de correspondance Phase 1 |
| `image` | Table de correspondance Phase 1 (signatures) |
| Titre doc généré | `{{LAMBDA_ID.data.siteName}} - [Type] - {{formatDate(now; "DD/MM/YYYY")}}` |
| Branche photos router | Inclure UNIQUEMENT si `FILE_UPLOAD` présent dans le Tally |
| Modules datastore | Inclure UNIQUEMENT si incrémentation automatique demandée — placer entre Lambda et Google Docs |
 
### Étape 15 — Détecter le LAMBDA_ID et mettre à jour le mapping
 
1. `Make:scenarios_get` avec l'ID du scénario créé
2. Parcourir le `flow`, trouver le `http:MakeRequest` dont `mapper.url` contient `lambda-url.eu-west-1.on.aws`
3. Extraire son `id` → c'est le `LAMBDA_ID`
4. Si le mapping utilisé lors de la construction ne correspond pas au `LAMBDA_ID` réel : signaler à l'utilisateur avec les valeurs corrigées à copier-coller manuellement dans Make
### Étape 16 — Confirmer
 
Fournir :
- Lien vers le scénario Make créé (dossier : `https://eu1.make.com/{Team ID}/scenarios?folder={Dossier Make}` — voir CONSTANTES)
- Le `LAMBDA_ID` détecté et confirmation que le mapping est correct
- Rappel des **actions manuelles restantes** :
  1. Vérifier dans Make que le module Google Docs a bien récupéré les champs du GDoc
  2. Connecter le webhook Make au formulaire Tally (Tally > Intégrations)
  3. Vérifier la connexion Google dans le module Google Docs
  4. Tester avec une soumission réelle