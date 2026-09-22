---
name: "formubees-create"
description: "Use this skill to create the full Formubees digital stack for a client — a downloadable .docx blueprint (Phase 1, user uploads to Drive themselves — never use Google Drive MCP), Tally form (Phase 2), and Make.com automation scenario (Phase 3). Trigger on /formubees-create, or whenever a user provides a PDF form and wants to digitize it into the Formubees workflow. Each phase is independent and can be run alone, in any order, if the corresponding artifact (docx/GDoc, Tally form) already exists. CRITICAL SAFETY RULE baked into this skill — Claude NEVER deletes, modifies, or overwrites anything — only creates new resources — and NEVER uploads to Google Drive via MCP."
---
 
# formubees-create — PDF → .docx + Tally + Make
 
Crée le stack digital Formubees complet pour un client à partir d'un PDF : document `.docx` téléchargeable (Phase 1), formulaire Tally (Phase 2), scénario Make.com (Phase 3).
 
Ce fichier est l'**index**. Le détail pas-à-pas de chaque phase est dans un fichier séparé, dans ce même dossier :
- **Phase 1 (document .docx)** → voir `PHASE1.md`
- **Phase 2 (formulaire Tally)** → voir `PHASE2.md`
- **Phase 3 (scénario Make.com)** → voir `PHASE3.md`
Toujours lire ce fichier `SKILL.md` en entier d'abord (règles absolues, constantes, table de correspondance), puis uniquement le fichier de la phase concernée.
 
## Outils requis
- Skill `docx` (génération du fichier Word local, voir `/mnt/skills/public/docx/SKILL.md`) — Phase 1
- Tally MCP : `https://api.tally.so/mcp`
- Make MCP : `https://mcp.make.com`
⛔ **Google Drive MCP interdit dans ce skill** — même si disponible dans la conversation, ne jamais l'utiliser pour créer/uploader un document. L'utilisateur gère lui-même l'upload sur Drive (voir `PHASE1.md`, Étape 5).
 
---
 
## 🔒 RÈGLE DE SÉCURITÉ ABSOLUE — NE JAMAIS VIOLER
 
```
❌ INTERDIT : supprimer, modifier, écraser, mettre à jour, archiver ou désactiver
              tout fichier, formulaire, scénario, webhook, bloc ou champ existant.
 
✅ AUTORISÉ : créer uniquement — nouveaux fichiers, nouveaux formulaires,
              nouveaux scénarios, nouveaux webhooks, nouveaux blocs.
```
 
Cette règle s'applique **partout** : Google Drive, Tally, Make.com.
Si une action semble nécessiter une modification → s'arrêter et alerter l'utilisateur.
 
## ✅ Vérification après création — règle absolue
 
Après toute création (formulaire, lot de blocs, scénario), **vérifier l'état réel via l'outil `get`/`list` correspondant** avant de continuer ou d'annoncer un succès — ne jamais se fier uniquement au retour de l'appel précédent. Exemple concret déjà rencontré : un lot de blocs Tally créé dans la mauvaise conversation/le mauvais formulaire à cause d'un état de session partagé — non détecté avant une vérification explicite via `list_forms`/`load_form`. En cas de doute sur un ID de connexion Make (compte, clé, webhook), vérifier dans un scénario existant du même type via `scenarios_get` plutôt que de supposer une valeur de ce skill.
 
---
 
## CONSTANTES — source unique de vérité
 
⚠️ Ces valeurs ne doivent être écrites **qu'ici**. Partout ailleurs dans ce skill (aperçus, tableaux récapitulatifs, phases), on y renvoie par leur nom plutôt que de les recopier, pour éviter les désynchronisations lors des futures corrections.
 
| Constante | Valeur |
|---|---|
| Team ID Make | `169479` |
| Dossier Make (FORMUBEES CLIENTS) | `240050` |
| Workspace Tally (FORMUBEES clients) | `3X4jdz` |
| Lambda URL | `https://avp37fi4qbfynfzbenigwsvmla0yaxxe.lambda-url.eu-west-1.on.aws/` |
| Connexion Alobees (module `app#alobees-3d4e-htqbwt:formubees`, param `__IMTCONN__`) | Account ID `9730558`, type `account:app#alobees-3d4e-htqbwt`, label "GAMMA-PROD (AzqdPTV36a6Fd2WwX)" — ⚠️ ce n'est PAS une clé API (`63789` est une Key et provoque `Account not found` sur ce module) |
| Destinataire email (module `google-email:sendAnEmail`, champ `to`) | `christian@alobees.com` — fixe, toujours, ne jamais demander ni utiliser une autre adresse |
| Gmail Connection | `Automation's Gmail Connection` |
| Google Docs Connection | `Automation's Google Docs Connection` |
| Datastore incrémentation (Formubees : No de Fiche) | ID `68679` |
| Image placeholder URL (signatures vides) | `https://drive.usercontent.google.com/download?id=1ItAK655cxahbBs18Bz9r0d0dmClIanV3&export=view&authuser=1` |
| Webhook page de remerciement Tally — image | `https://storage.tally.so/63d737f1-413b-47ba-bcda-6f547b3366f8/Capture-d-ecran-2025-05-05-a-10.07.53.png` (lien : `https://app.alobees.com`) |
| Scénario Make de référence | `6718018` (🟢 Formubees / BdC n°1 / MENUISERIE ENTSIA) |
 
**Indexation Make — rappel général** : Make est indexé en **base 1** (pas en base 0 comme en programmation classique) : `[1]` = 1er élément d'une collection/array, `[2]` = 2e, etc. Ne jamais utiliser `[0]`. S'applique à tout accès par index dans Make, pas seulement aux dropdowns (voir `PHASE3.md`).
 
---
 
## Nomenclature — règle absolue
 
Document `.docx`/GDoc et Tally portent **toujours ce nom** :
 
```
⚙️ [TYPE] / [NOM CLIENT]
```
 
Le scénario Make (et son webhook) portent **toujours ce nom** (avec `Formubees /` en plus) :
 
```
⚙️ Formubees / [TYPE] / [NOM CLIENT]
```
 
- `⚙️` = engrenage = "en cours"
- `[TYPE]` = type de document abrégé : BdI, CR, PV, Devis…
- `[NOM CLIENT]` = nom de l'entreprise en MAJUSCULES
Si le nom n'est pas fourni → demander avant de continuer.
 
---
 
## Nommage de la conversation
 
**Renommer la conversation** après extraction des informations du document et avant la Phase 1.
 
**Format :**
```
[NOM_ENTREPRISE] / [TYPE]
```
 
**Exemples :**
- `DEVILLERS NORBERT / Bon de commande`
- `SOPITHERME / Rapport d'intervention`
- `ACB / Devis`
- `CLIENT_NAME / Formulaire sur mesure`
⚠️ **Contrairement à formubees-catalogue, ne jamais ajouter "Catalogue" ni de numéro** — ce skill est réservé aux formulaires sur mesure.
 
Ça aide à suivre l'avancement et à retrouver facilement ce qui est en cours de création.
 
---
 
## Démarrer depuis n'importe quelle phase
 
Ce skill n'impose pas de commencer par la Phase 1. Si l'utilisateur a déjà créé lui-même le document (docx/GDoc) et/ou le formulaire Tally, on peut démarrer directement à la phase suivante :
 
- **Démarrer à la Phase 2 directement** : demander le lien/ID du GDoc (utile pour la table de correspondance) — si l'utilisateur n'a pas de table de correspondance, la reconstruire à partir du GDoc fourni avant de créer le Tally.
- **Démarrer à la Phase 3 directement** : demander le lien du GDoc, le dossier Drive de destination, et la table de correspondance (Phase 1) — si elle n'existe pas, la reconstruire à partir du GDoc et du formulaire Tally déjà créés avant de construire le mapping Make.
- Dans tous les cas, reconstituer/produire la table de correspondance manquante **avant** de passer à la création Make — c'est la pièce centrale qui garantit la cohérence GDoc ↔ Tally ↔ Make (voir section suivante).
---
 
## Transitions entre phases — règle absolue
 
```
Phase 1 terminée → produire la TABLE DE CORRESPONDANCE + ATTENDRE confirmation utilisateur
Phase 2 terminée → annoncer + ATTENDRE confirmation utilisateur
Phase 3 → créer webhook + scénario en une seule fois (pas de pause interne)
```
 
❌ Ne JAMAIS enchaîner automatiquement les phases.
L'utilisateur a des actions manuelles entre chaque phase (conversion GDoc, vérifications Tally…).
 
---
 
## TABLE DE CORRESPONDANCE — pièce centrale du workflow
 
La table de correspondance est générée à la fin de la Phase 1 (ou reconstruite si on démarre à une autre phase) et réutilisée telle quelle en Phase 3. Elle est la **source de vérité unique** qui garantit la cohérence entre GDoc, Tally et Make.
 
Format :
 
```
| Label PDF               | Placeholder GDoc  | Label Tally             | Donnée Make                                      |
|-------------------------|-------------------|-------------------------|--------------------------------------------------|
| Date                    | {{date}}          | Date                    | {{if(LAMBDA_ID.data.`Date`; formatDate(...))}}   |
| Heure d'arrivée         | {{heure-arrivee}} | Heure d'arrivée         | {{LAMBDA_ID.data.`Heure d'arrivée`}}             |
| Signature du client     | [image GDoc]      | Signature du client     | {{if(LAMBDA_ID.data.`Signature du client`[]; ...)}} |
| Nom du chantier         | {{chantier}}      | (hidden field)          | {{LAMBDA_ID.data.siteName}}                      |
```
 
**Règles de nommage des placeholders (Option C — noms courts) :**
- Minuscules, sans accents, sans apostrophes
- Mots séparés par un tiret `-`
- **Court et non ambigu** — pas besoin de reproduire le label complet
- Exemples : `{{date}}` · `{{heure-arrivee}}` · `{{heure-depart}}` · `{{client}}` · `{{descriptif}}` · `{{materiels}}` · `{{statut}}` · `{{technicien}}` · `{{chantier}}`
- ⚠️ Le placeholder dans le GDoc doit être **identique à la clé** dans Make `requests` — c'est la même chaîne, copiée telle quelle
**Champs signature et file upload — traitement spécial :**
- Dans le GDoc : insérer une **image blanche placeholder** (obligatoire pour éviter les erreurs Google Docs à la génération)
- Dans Make `image` (pas `requests`) : utiliser le pattern avec fallback vers l'image blanche (voir `PHASE3.md`).
⚠️ **Syntaxe exacte et obligatoire** — le `[]` se place **après le backtick fermant** du label, jamais à l'intérieur :
 
```
{{if(LAMBDA_ID.data.`Signature du client`[]; LAMBDA_ID.data.`Signature du client`[]; "<Image placeholder URL — voir CONSTANTES>")}}
```
 
Règle : backticks autour du label (si le label contient des espaces/accents/caractères spéciaux), puis `[]` immédiatement après le backtick fermant `` ` ``, avant le `;`.
 
- L'URL de l'image blanche placeholder → voir CONSTANTES ("Image placeholder URL")
- Si le répondant ne signe pas → l'image blanche est insérée automatiquement → Google Docs ne renvoie pas d'erreur
---
 
## Récapitulatif des règles absolues
 
| Règle | Détail |
|---|---|
| 🔒 Création uniquement | Jamais de suppression, modification, écrasement nulle part |
| ⛔ **Pas de Google Drive MCP** | Phase 1 = génération d'un `.docx` téléchargeable via `present_files`, jamais d'upload/création Drive via MCP — même si le connecteur est disponible |
| ⛔ Confirmation obligatoire | Avant chaque phase, avant chaque création |
| ⛔ Pas d'enchaînement auto | Attendre confirmation entre Phase 1→2 et Phase 2→3 |
| ✅ Vérification post-création | Toujours vérifier via `get`/`list` après une création, ne pas se fier au seul retour de l'appel |
| ⚠️ **Scénario de référence = source vivante** | Toujours interroger via `scenarios_get` avant de construire le blueprint Phase 3 — ne jamais se fier aux valeurs de ce skill sans les reconfirmer contre le scénario réel (voir `PHASE3.md`, Étape 14) |
| ✅ LAMBDA_ID | Toujours détecter le vrai ID après création — ne jamais supposer une valeur fixe |
| ✅ Table de correspondance | Générée en Phase 1 (ou reconstruite), réutilisée telle quelle en Phase 3 |
| ✅ Placeholders courts | Sans accents, sans apostrophes, kebab-case court — identiques dans GDoc ET Make |
| ✅ Signatures | Image blanche placeholder dans GDoc + pattern `if([]; []; URL-blanche)` dans Make |
| ✅ Dropdowns (cases à cocher) | `{{if(LAMBDA_ID.data.\`Label\`[1] = "ValeurCible"; "☑"; "☐")}}` — `[1]` = première option = valeur qui affiche ☑ |
| ⚠️ **Indexation Make** | **Base 1, jamais base 0** — voir CONSTANTES |
| ✅ Nomenclature | GDoc/Tally : `⚙️ TYPE / NOM CLIENT` — Make : `⚙️ Formubees / TYPE / NOM CLIENT` |
| ✅ Champs Tally | Tous `isRequired: false` — aucune exception |
| ⚠️ **Lignes répétées → labels uniques** | Tableau multi-lignes (ex. 15 lignes d'articles) : jamais le même titre de champ répété à l'identique — toujours suffixer `— Ligne N` (ex. `DÉSIGNATION — Ligne 1`). Sinon le mapping Make par libellé (`LAMBDA_ID.data.\`Label\``) ne peut pas distinguer les lignes et perd des données silencieusement. Voir `PHASE2.md`, Étape 8.2 |
| ✅ Toutes les valeurs fixes | Voir section CONSTANTES en tête de ce fichier — ne pas les recopier ailleurs |
| ✅ FILE_UPLOAD absent | Supprimer la branche photos du router Make |
| ✅ Incrémentation absente | Ne pas inclure les modules datastore — uniquement si explicitement demandé |
| ✅ Incrémentation présente | Voir `PHASE3.md`, section incrémentation (datastore `68679`) |
| ✅ create_blocks | Max 10 groupes par appel — scinder si nécessaire |
 
---
 
## Cas particuliers
 
| Situation | Action |
|---|---|
| PDF scanné | Signaler la limitation, extraire au mieux |
| Tableau complexe | Reproduire au mieux, signaler |
| Tableau multi-lignes (lignes répétées) | Suffixer chaque titre de champ `— Ligne N` pour garantir l'unicité — voir règle dédiée ci-dessus et `PHASE2.md` Étape 8.2 |
| SPEC absente | Tally sur PDF seul, le signaler |
| Champ PDF ambigu | Choisir le type le plus logique, mentionner dans l'aperçu |
| Phase demandée sans les phases précédentes | Possible — voir section "Démarrer depuis n'importe quelle phase" : reconstruire la table de correspondance à partir des artefacts déjà fournis avant de continuer |
| Action qui ressemble à une modification | S'arrêter, alerter l'utilisateur, ne pas procéder |
| Module Google Docs vide après création Make | Normal — nécessite validation manuelle dans l'interface Make (limitation API) |
| `MakeApiError: Account 'XXXXX' not found` sur le module `app#alobees-3d4e-htqbwt:formubees` | La valeur fournie est une Key (API Key), pas un Account. Ce module attend `__IMTCONN__` = Account ID — voir CONSTANTES. En cas de doute sur un ID de connexion, vérifier dans un scénario existant du même type via `scenarios_get` plutôt que de supposer une valeur du skill. |
 
---
 
## Changelog
 
| Date | Modification |
|---|---|
| 2026-09-10 | Interdiction du Google Drive MCP — Phase 1 génère désormais un `.docx` téléchargeable (`present_files`) au lieu d'un GDoc créé via MCP. Dossier Drive de destination déplacé de l'Étape 3 (Phase 1, supprimé) vers l'Étape 9 (Phase 3, où il sert réellement — mapping `folderId` du module Google Docs). |
| 2026-09-10 | Correction connexion Alobees : Account ID `9730558` (GAMMA-PROD) remplace l'ancien `63789` (une Key, pas un Account — provoquait `Account not found`). |
| 2026-09-10 | Destinataire email fixé à `christian@alobees.com` (auparavant demandé à l'utilisateur à chaque fois). |
| 2026-09-10 | Ajout de la règle générale d'indexation Make en base 1 (`[1]` = 1er élément), auparavant seulement implicite dans le pattern dropdown. |
| 2026-09-10 | Restructuration en 4 fichiers (`SKILL.md` + `PHASE1.md`/`PHASE2.md`/`PHASE3.md`) pour la lisibilité et pour limiter le risque qu'une section entière (ex: incrémentation) devienne illisible sans être détectée. |
| 2026-09-10 | Ajout d'une section CONSTANTES unique en tête de fichier — les autres sections y renvoient au lieu de recopier les valeurs, pour éviter les désynchronisations. |
| 2026-09-10 | Reconstruction complète de la section "Incrémentation automatique" (Phase 3), corrompue dans une édition précédente (variables `{{...}}` vidées). Reconstruite à partir du scénario réel `6718018`, modules `43`/`44`. |
| 2026-09-10 | Traduction en français de la section "Nommage de la conversation" (auparavant en anglais, seul passage non francophone du skill). |
| 2026-09-10 | Suppression de la ligne "Webhook Make indisponible" (Cas particuliers) — ne renvoyait à aucune liste concrète, jugée obsolète/inutile. |
| 2026-09-21 | Ajout de la règle "Lignes répétées → labels uniques" (Étape 8.2, `PHASE2.md`) suite à une erreur réelle : un tableau de 15 lignes d'articles créé avec des titres identiques ligne après ligne (`FABR.`, `RÉF`, `DÉSIGNATION`…) rendait le mapping Make par libellé impossible à construire correctement — 14 lignes sur 15 auraient perdu leurs données silencieusement. Correction manuelle faite par l'utilisateur sur le formulaire concerné ; règle ajoutée pour l'empêcher à l'avenir. |
| 2026-09-10 | Clarification "Phase demandée sans la précédente" → nouvelle section "Démarrer depuis n'importe quelle phase" expliquant qu'on peut commencer à n'importe quelle étape si les artefacts existent déjà, avec reconstruction de la table de correspondance dans ce cas. |
| 2026-09-10 | Ajout d'une règle explicite : le scénario de référence Make doit être interrogé via `scenarios_get` avant toute construction de blueprint Phase 3 (source de vérité vivante), et non traité comme une simple référence documentaire — aurait détecté immédiatement les erreurs `63789` et incrémentation corrompue. |
| 2026-09-10 | Renforcement de la consigne sur l'URL webhook Tally→Alobees (`https://app.alobees.com/api/form/webhook`, fixe pour tous les clients) — suite à une erreur réelle où l'URL du webhook Make (`hook.eu1.make.com/...`) avait été donnée par erreur à la place. |