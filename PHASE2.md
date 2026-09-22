# PHASE 2 — Formulaire Tally
 
Voir `SKILL.md` pour les règles absolues, les CONSTANTES et la TABLE DE CORRESPONDANCE avant de lire cette phase. Si cette phase démarre sans que la Phase 1 ait été faite via ce skill (document déjà créé par l'utilisateur lui-même), voir `SKILL.md` section "Démarrer depuis n'importe quelle phase".
 
### Déclenchement
 
Démarre quand l'utilisateur confirme être prêt, avec ou sans SPEC.
**SPEC** = texte brut, notes du RDV client avec d'éventuelles précisions sur le formulaire.
Si pas de SPEC → créer le Tally basé sur le PDF seul, le signaler.
 
### Étape 6 — Analyser PDF + SPEC → déduire les champs
 
**Règles de déduction des types :**
 
| Champ détecté dans le PDF | Type Tally |
|---|---|
| Texte court (nom, référence, lieu…) | `INPUT_TEXT` |
| Texte long (description, commentaire…) | `TEXTAREA` |
| Date | `INPUT_DATE` |
| Heure | `INPUT_TIME` |
| Nombre / quantité | `INPUT_NUMBER` |
| Signature | `SIGNATURE` |
| Case à cocher unique (oui/non) | `CHECKBOXES` |
| Choix multiples exclusifs | `MULTIPLE_CHOICE` |
| Upload fichier / photo | `FILE_UPLOAD` |
| Email | `INPUT_EMAIL` |
| Téléphone | `INPUT_TEL` |
 
**Règle obligatoire — champs non obligatoires :**
- **Tous les champs sont `isRequired: false` sans exception.**
- Les champs avec `defaultAnswer` ou `defaultToday` sont préremplis automatiquement, donc l'obligation n'a pas de sens pour eux non plus.
### Étape 7 — Aperçu Tally et confirmation
 
```
📋 Aperçu du formulaire Tally à créer
 
Nom              : ⚙️ [TYPE] / [NOM CLIENT]
Dossier Tally    : FORMUBEES (clients)
 
Champs à créer :
1. [Label Tally] — [Type] — non obligatoire
…
 
Précisions SPEC : [résumé ou "Aucune"]
 
✅ Souhaitez-vous créer ce formulaire ?
```
 
❌ Ne jamais créer sans confirmation.
 
### Étape 8 — Créer le formulaire Tally
 
Toutes les opérations ci-dessous sont des **créations uniquement** — ✅ jamais de suppression ni modification.
 
**8.1** `Tally:create_new_form` dans le workspace `FORMUBEES (clients)` (voir CONSTANTES dans `SKILL.md` pour l'ID), bouton `Envoyer`
 
Deux titres distincts à gérer :
 
- **Titre interne** (nom du formulaire dans Tally, non visible par le client) : `⚙️ TYPE / NOM CLIENT` — suit la nomenclature standard
- **Titre visible** (FORM_TITLE affiché en tête du formulaire, vu par le client) : utiliser le **nom complet du document** selon la table suivante :
| Abréviation | Titre visible |
|---|---|
| RDI | Rapport d'intervention |
| BDI | Bon d'intervention |
| BDC | Bon de commande |
| PV | PV de réception |
 
Si l'abréviation ne figure pas dans cette table → demander à l'utilisateur le nom complet avant de créer.
 
Le `title` passé à `Tally:create_new_form` est le titre interne (`⚙️ TYPE / NOM CLIENT`). Le titre visible est mis à jour juste après via `Tally:set_form_title` avec le nom complet.
 
**8.2** Ajouter les **champs** déduits à partir du PDF/SPEC, dans l'ordre logique :
- ⚠️ Maximum 10 groupes par appel `create_blocks` — scinder en plusieurs appels si nécessaire
- **Tous les champs `isRequired: false`** — sans exception
- Labels Tally = exactement la colonne "Label Tally" de la table de correspondance
- ⚠️ **Lignes répétées (tableaux multi-lignes) — labels uniques obligatoires** : si le PDF contient un tableau où une même ligne de champs se répète (ex. 15 lignes d'articles avec FABR./RÉF/DÉSIGNATION/QT…), **chaque titre de champ doit être unique sur tout le formulaire** — jamais le même libellé répété à l'identique ligne après ligne. Le mapping Make (`LAMBDA_ID.data.\`Label\``) fonctionne par nom de libellé exact ; des libellés dupliqués (ex. "DÉSIGNATION" × 15) rendent les lignes indiscernables pour le Lambda qui transforme les réponses Tally en objet `data` — une seule valeur survit, les autres sont perdues silencieusement à la génération du document. Toujours suffixer chaque titre avec son numéro de ligne, ex. `FABR. 1`, `FABR. 2` … `FABR. 15` (même convention pour RÉF, DÉSIGNATION, QT, etc.). Mettre à jour la table de correspondance en conséquence (colonne "Label Tally" = libellé suffixé, pas le libellé générique du PDF).
- ✅ Après chaque lot de blocs, vérifier l'état réel du formulaire (`save_form` en `DRAFT` + `list_forms`/`load_form`) avant de continuer, conformément à la règle de vérification post-création de `SKILL.md` — un état de session partagé entre plusieurs conversations Make/Tally peut faire atterrir un lot de blocs dans le mauvais formulaire sans qu'aucune erreur ne soit renvoyée.
**8.3** Créer les **règles de logique conditionnelle** via `Tally:apply_logic` si spécifiées dans la SPEC :
- Exemple : SI champ A = "Oui" THEN SHOW champs B, C, D
- Syntaxe DSL : `WHEN <condition> THEN <action>`
**8.4** ⚠️ **Passer les champs conditionnels en non obligatoires — règle absolue**
 
Dès qu'un formulaire contient des champs affichés via logique conditionnelle (règles SHOW), ces champs apparaissent en `isRequired: true` par défaut après leur création, ce qui forcerait le répondant à les remplir tous. Il faut impérativement les passer en `isRequired: false` **après la création des champs visibles et des règles de logique**.
 
Procédure : **un seul appel `Tally:configure_blocks` groupé** contenant tous les questionUuids des champs concernés (lignes conditionnelles entières).
 
```
// Exemple — 1 seul appel pour toutes les lignes conditionnelles
Tally:configure_blocks([
  { questionUuid: "uuid-ref-2",  isRequired: false },
  { questionUuid: "uuid-desc-2", isRequired: false },
  { questionUuid: "uuid-qte-2",  isRequired: false },
  { questionUuid: "uuid-delai-2",isRequired: false },
  // ... idem pour lignes 3 à N
])
```
 
Cette étape est obligatoire même si `isRequired: false` a été spécifié à la création — Tally peut l'ignorer pour les champs ciblés par des règles SHOW.
 
**8.5** Ajouter la **page de remerciement** :
- `PAGE_BREAK` (`isThankYouPage: true`)
- `HEADING_3` : "✅ Votre rapport a bien été enregistré ✅"
- `TEXT` : "Dans quelques instants vous retrouverez votre rapport dans les documents de votre chantier sur Alobees 📲"
- `TEXT` : "Bonne journée ☀️"
- `IMAGE` : voir CONSTANTES dans `SKILL.md` ("Webhook page de remerciement Tally — image") avec lien `https://app.alobees.com`
**8.6** `Tally:save_form` status `PUBLISHED`, puis **vérifier** via `list_forms`/`load_form` que le formulaire publié correspond bien à ce qui a été construit, et confirmer avec le lien.
 
**8.7** Annoncer : *"Phase 2 terminée ✅ — ⚠️ Action manuelle requise : connecter le webhook dans **Tally > Intégrations > Webhooks** avec l'endpoint `https://app.alobees.com/api/form/webhook` (l'API Tally ne permet pas de le configurer automatiquement). Dis-moi quand tu es prêt pour la Phase 3 (Make). J'aurai besoin du lien du GDoc blueprint converti et du dossier Drive de destination si tu ne me les as pas encore donnés."*
 
⚠️ **Cette URL est fixe et unique pour tous les clients** (`https://app.alobees.com/api/form/webhook`) — c'est un relais Alobees, pas l'URL du webhook Make créé en Phase 3 (Étape 13). Ne jamais donner à l'utilisateur l'URL `hook.eu1.make.com/...` du scénario Make à la place : erreur déjà commise une fois, elle casserait la connexion Tally ↔ Alobees.