# Module 04 — Contacts (personnes physiques)

> **Version** : 3.0 (refonte vérifiée contre le code source actuel, juillet 2026)
> **Code de référence** :
> - `ERP_REACT/backend/routers/companies.py` — CRUD des contacts, servi par le **même** routeur que les Entreprises ; 5 endpoints réels sous `/contacts` : liste (`:621`), statistiques (`:707`), création (`:753`), modification (`:829`), suppression (`:934`). Il n'existe **aucun** fichier `routers/contacts.py`.
> - `ERP_REACT/backend/routers/contacts_ai.py` — assistant IA dédié aux Contacts ; 2 endpoints (`/contacts/ai/chat` `:284`, `/contacts/ai/confirm-action` `:423`) ; patron « proposer puis confirmer ».
> - `ERP_REACT/frontend/src/pages/ContactsPage.tsx` (523 lignes) — page `/contacts` : liste, statistiques, recherche, modales de création et de modification.
> - `ERP_REACT/frontend/src/components/contacts/ContactsAssistantTab.tsx` — modale de l'assistant IA (clavardage + cartes de proposition).
> - `ERP_REACT/frontend/src/api/companies.ts` — client d'API des contacts (`listContacts` `:128`, `getContactStats` `:143`, `createContact` `:148`, `updateContact` `:153`, `deleteContact` `:157`) ; `ERP_REACT/frontend/src/api/contactsAi.ts` — client de l'assistant IA. Il n'existe **aucun** fichier `api/contacts.ts`.
> - Libellés : `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json`, clés `contacts.*` (lignes 1039-1118).
> **Préfixe des API** : `/api/erp/v1` (`erp_config.py:9`). Exemple réel : `GET /api/erp/v1/contacts`.
> **Table PostgreSQL** (par schéma de tenant) : `contacts` (entité principale) ; les colonnes d'adresse et quelques autres sont ajoutées à la volée par `_ensure_contact_address_cols` au premier accès ; les crédits de l'assistant IA vivent sur `public.ai_prepaid_credits`.
> **Cadrage** : ce module gère le **carnet des personnes physiques** (les individus), qu'elles soient rattachées ou non à une entreprise. Il est **distinct** du module Entreprises (les personnes morales, manuel 04) et du module CRM et opportunités (le pipeline commercial, manuel 06). Un contact n'est **pas** un compte utilisateur de l'ERP.

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Processus pas à pas](#3-processus-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission du module

Le module **Contacts** est votre **carnet d'adresses des personnes**. Il centralise chaque individu avec qui votre entreprise fait affaire : le chargé de projet chez un client, le représentant d'un fournisseur, l'estimateur d'un sous-traitant, l'architecte, l'ingénieur, l'arpenteur-géomètre, le conseiller à la banque, le courtier en assurance, ou toute autre personne à joindre.

Un contact est **toujours une personne** (un prénom et un nom de famille). Il peut être **rattaché à une entreprise** — par exemple « Marie Gagnon, ingénieure chez Construction ABC » — ou **exister seul**, sans employeur, lorsque vous ne connaissez que la personne. Chaque fiche transporte les coordonnées (courriel, téléphone fixe, mobile), le rôle ou poste, la fonction, le département, une adresse postale et des notes.

Les contacts servent de **point de rattachement** dans le reste de l'ERP : ils sont sélectionnables comme interlocuteurs dans les opportunités du CRM, les soumissions, les projets, les contrats et les courriels. Ils alimentent aussi le **« contact principal »** d'une entreprise (voir le manuel 04).

> **Contacts et Entreprises sont deux modules jumeaux.** Ils partagent le même code côté serveur (`companies.py`) et sont fortement liés : la fiche d'une entreprise affiche ses contacts, et un contact pointe vers son entreprise. Mais ce sont **deux pages distinctes** dans le menu (`/contacts` et `/entreprises`). Le module Contacts gère les **personnes** ; le module Entreprises gère les **organisations**.

### 1.2 Accès par le menu latéral

- **Menu latéral** → groupe **« GESTION »** → **« Contacts »** (icône `Users`), situé entre « Entreprises » et « Ventes » (`Sidebar.tsx:49`, clé `nav.contacts` de `nav.json:8`).
- **Adresse** : `/contacts`.
- **Composant** : `ContactsPage` (chargé à la demande, `App.tsx:100` et `App.tsx:224`), protégé par l'authentification (`ProtectedRoute`).
- **Titre de la page** : « Contacts » (`ContactsPage.tsx:216`).
- **Une seule vue** : il n'y a **pas d'onglets** ni de sous-pages. Toute la navigation se fait par la recherche, le tri des colonnes, la pagination et les deux modales (créer, modifier). Il n'existe **pas** de page de détail dédiée du type `/contacts/{id}`.

### 1.3 Permissions et rôles

| Action | Garde côté serveur | Qui peut le faire |
|---|---|---|
| Consulter, rechercher (lecture) | `get_current_user` | Tout utilisateur authentifié du tenant |
| Créer un contact | `get_current_user` | Tout utilisateur authentifié du tenant |
| Modifier un contact | `get_current_user` | Tout utilisateur authentifié du tenant |
| **Supprimer un contact** | **`require_tenant_admin_or_role()`** (`companies.py:935`) | **Administrateur du tenant** (ou super-administrateur) |
| Assistant IA (clavardage + confirmation) | `get_current_user` + garde IA + crédits | Tout utilisateur authentifié, si le solde de crédits IA le permet |

- **La suppression est réservée aux administrateurs.** La garde `require_tenant_admin_or_role()` (`erp_auth.py:720-747`) autorise l'utilisateur dont le drapeau `is_admin` est vrai — relu côté serveur, donc infalsifiable —, ou dont le rôle est `admin`, ou un super-administrateur de plateforme. Un employé ordinaire peut **créer et modifier** une fiche, mais **pas la supprimer**.
- **Détail important sur le bouton Supprimer** : l'icône de corbeille est **affichée à tout le monde** dans le tableau et les cartes (`ContactsPage.tsx:313-319` et `:372-378`), sans masquage selon le rôle. Un utilisateur non administrateur peut donc cliquer sur « Supprimer », mais l'ERP lui répondra alors **403 (accès refusé)** et le contact ne sera pas supprimé. Ce n'est pas un bogue : la sécurité est appliquée côté serveur, pas par le masquage du bouton.
- **Isolation par tenant** : chaque requête applique `db.set_tenant(conn, user.schema)`. Sans contexte de tenant, l'API renvoie **HTTP 400 « Contexte tenant manquant »**. Un contact d'un tenant n'est **jamais** visible d'un autre tenant ; aucun identifiant de tenant n'est accepté depuis le navigateur (protection contre l'accès indirect à des données d'autrui).
- **Mode consultation (abonnement)** : si le compte est en lecture seule — sans abonnement Stripe actif ou après annulation —, toutes les écritures (`POST`, `PUT`, `DELETE`) renvoient **403 « Mode consultation »** ; seules les consultations (`GET`) passent (`erp_auth.py:526-530`). Si l'entreprise du tenant est carrément désactivée, la session est coupée (**401**). Ce contrôle s'applique à tous les endpoints du module.

### 1.4 Composants du module

1. **Bandeau de statistiques** — 4 compteurs **globaux** du tenant (Contacts, Entreprises, Avec courriel, Avec téléphone).
2. **Barre de commandes** — création, assistant IA, recherche avec effacement rapide.
3. **Liste des contacts** — tableau triable et redimensionnable sur ordinateur, cartes empilées sur mobile, avec pagination (20 par page).
4. **Modale « Nouveau Contact »** — formulaire complet de création.
5. **Modale « Modifier le contact »** — même formulaire, pré-rempli, avec protection de l'entreprise déjà liée.
6. **Assistant IA — Contacts** — clavardage qui lit vos données réelles et **propose** la création d'un contact, que vous confirmez avant exécution.

### 1.5 Ce que le module ne fait pas (pour cadrer les attentes)

- **Aucune notion de « type de contact » ni d'« interaction ».** La fiche d'un contact n'a **aucun** champ de catégorie, de type, ni de journal d'appels / courriels / rencontres. Les rôles et postes sont du **texte libre** (`role_poste`, `fonction`, `departement`). Les activités et interactions commerciales (appels, courriels, rencontres avec pointage) appartiennent au module **CRM et opportunités** (manuel 06), pas ici.
- **Aucun export** (CSV ou PDF), **aucune impression**, **aucun téléversement** de fichier, de photo ou de carte de visite.
- **Aucune action de masse** : pas de sélection multiple, pas de suppression ni de modification groupée.
- **Aucune page de détail** dédiée : seulement la liste plus la modale de modification.
- **Aucune détection de doublon** : deux contacts avec le même courriel ou le même nom peuvent coexister ; utilisez la recherche avant de créer.
- **Aucune validation de format** : le courriel, le téléphone et le code postal sont des **champs de texte libre** ; le serveur ne contrôle que leur **longueur maximale**, pas leur forme (le courriel n'exige même pas de « @ »).
- **La suppression est définitive** (voir la section 2.10) : contrairement aux entreprises, qui sont simplement désactivées, un contact supprimé est **effacé physiquement** et ne peut pas être récupéré depuis l'interface.
- **L'assistant IA ne fait que créer** un contact (en version 1). Il ne modifie ni ne supprime aucune fiche.

---

## 2. Interface

### 2.1 Disposition générale

La page empile, du haut vers le bas (`ContactsPage.tsx`) :

1. les bandeaux d'alerte — rouge en cas d'erreur, vert en cas de succès (le message de succès disparaît seul après 3 secondes) ;
2. le titre « Contacts » et les 4 cartes de statistiques ;
3. la barre de commandes (création, assistant IA, recherche) ;
4. la liste — un **tableau** sur grand écran, des **cartes empilées** sur mobile — avec sa pagination.

Les modales (créer, modifier, assistant IA) s'ouvrent par-dessus la page.

### 2.2 Bandeau de statistiques (4 cartes)

Quatre compteurs, calculés sur **l'ensemble du tenant** par l'endpoint `GET /contacts/stats` — et **non** sur la seule page affichée (`companies.py:707-737`) :

| Carte | Icône / couleur | Calcul côté serveur |
|---|---|---|
| **Contacts** | `Users` / bleu | `COUNT(*)` — nombre total de contacts |
| **Entreprises** | `Building2` / violet | `COUNT(DISTINCT company_id)` parmi les contacts rattachés à une entreprise |
| **Avec courriel** | `Mail` / vert | Nombre de contacts dont le courriel est renseigné (non vide) |
| **Avec Tél.** | `Phone` / jaune | Nombre de contacts dont le téléphone est renseigné (non vide) |

> **À savoir.** Ces quatre chiffres sont **globaux** : ils ne changent **pas** quand vous tournez les pages ou lancez une recherche. La carte « Entreprises » compte le nombre d'**entreprises distinctes** représentées par vos contacts (un contact sans employeur n'y est pas compté). Les cartes « Avec courriel » et « Avec Tél. » vous aident à repérer les fiches incomplètes.

### 2.3 Barre de commandes

À gauche (`ContactsPage.tsx:227-255`) :

- **« Nouveau Contact »** (icône `Plus`, bouton principal) → ouvre la modale de création (section 2.8).
- **« Assistant IA »** (icône `Sparkles`) → ouvre la modale de l'assistant IA (section 2.11).

À droite :

- Un champ de **recherche** avec le texte d'invite « **Rechercher par nom, courriel, entreprise, rôle…** ». La recherche est **différée** de 300 millisecondes après votre dernière frappe (pour ne pas interroger le serveur à chaque lettre) et **revient toujours à la page 1**.
- Un bouton **« Effacer »** (icône `X`) apparaît dès que le champ contient du texte ; il vide la recherche et revient à la première page.

> **La recherche couvre quatre champs à la fois** (`companies.py:647-660`), exactement comme le promet le texte d'invite : le **nom complet** (prénom + nom de famille), le **courriel**, le **nom de l'entreprise** rattachée, et le **rôle ou poste**. Elle est insensible à la casse et cherche le terme n'importe où dans ces champs. Elle est aussi **sûre** : les caractères spéciaux que vous tapez sont traités comme du texte ordinaire, et le terme est borné à 100 caractères.
>
> La recherche s'exécute **côté serveur** sur l'ensemble du tenant, puis renvoie une page de résultats. Ce n'est donc pas un simple filtre de la page courante : vous trouvez un contact même s'il se trouve à la page 12.

### 2.4 Tableau (ordinateur, largeur `md` et plus)

Le tableau (`ContactsPage.tsx:263-343`) présente **six colonnes**. Cinq d'entre elles sont **triables** (cliquez l'en-tête pour trier, recliquez pour inverser) et **redimensionnables** (glissez la poignée entre deux en-têtes). Les largeurs par défaut sont : Nom 200 px, Entreprise 180 px, Rôle / Poste 160 px, Courriel 220 px, Téléphone 140 px (`ContactsPage.tsx:207`).

| Colonne | Clé de tri | Contenu affiché |
|---|---|---|
| **Nom** | `prenom` | Pastille d'**initiales** (première lettre du prénom + première lettre du nom) + « Prénom Nom » + badge bleu **« Principal »** si le contact est marqué comme contact principal |
| **Entreprise** | `companyNom` | Le nom de l'entreprise rattachée, ou `--` si le contact est autonome |
| **Rôle / Poste** | `rolePoste` | Le rôle ou poste, ou `--` |
| **Courriel** | `email` | Icône `Mail` + adresse courriel, ou `--` |
| **Téléphone** | `telephone` | Numéro mis en forme (par exemple `(514) 555-1234`), ou `--` |
| **Actions** | — | Bouton **Modifier** (crayon) + bouton **Supprimer** (corbeille) |

> **Le tri ne réorganise que la page affichée.** La liste est d'abord triée par le serveur selon le **nom de famille** (`companies.py:681`), puis découpée en pages de 20. Quand vous cliquez sur un en-tête, vous réordonnez les **20 lignes visibles** de la page courante. Pour parcourir tout le carnet dans un ordre précis, combinez tri et pagination, ou utilisez la recherche.

### 2.5 Cartes mobile (largeur inférieure à `md`)

Sur téléphone et petit écran (`ContactsPage.tsx:346-408`), le tableau cède la place à des **cartes empilées**. Chaque carte reprend le même contenu, condensé : pastille d'initiales, nom complet, badge « Principal » le cas échéant, rôle en sous-titre, puis une ligne secondaire avec l'entreprise (icône `Building2`), le courriel (icône `Mail`) et le téléphone (icône `Phone`). Les boutons **Modifier** et **Supprimer** sont présents sur chaque carte.

### 2.6 États vides

- **Recherche sans résultat** : le message « **Aucun résultat pour cette recherche** » s'affiche, accompagné d'un bouton **« Effacer »** pour repartir de la liste complète.
- **Carnet vide** (aucun contact) : une icône `Users` et le message « **Commencez par ajouter un contact.** », avec un bouton **« Nouveau Contact »**.

### 2.7 Pagination

La pagination n'apparaît que s'il y a **plus d'une page** de résultats. Chaque page contient **20 contacts** (`ContactsPage.tsx:61`). Une protection évite de rester bloqué sur une page devenue vide : si le total baisse sous la page courante (après des suppressions, par exemple), l'affichage revient automatiquement à la dernière page qui contient des données (`ContactsPage.tsx:118-121`).

### 2.8 Modale « Nouveau Contact »

Ouverte par le bouton **« Nouveau Contact »** (`ContactsPage.tsx:417-465`). Les champs, dans l'ordre :

| Champ | Obligatoire | Contrainte de longueur |
|---|---|---|
| **Prénom** | **Oui** | 100 caractères |
| **Nom de famille** | **Oui** | 100 caractères |
| **Courriel** | Non | 255 caractères |
| **Téléphone** | Non | 50 caractères |
| **Entreprise** (menu déroulant) | Non | option vide « -- Sélectionner -- » par défaut |
| **Rôle / Poste** | Non | 150 caractères |
| **Fonction** | Non | 150 caractères |
| **Département** | Non | 150 caractères |
| **Mobile** | Non | 50 caractères |
| **Adresse** (invite « 123 rue Exemple ») | Non | 300 caractères |
| **Ville** | Non | 120 caractères |
| **Province** (invite « QC ») | Non | 80 caractères |
| **Code postal** (invite « H0H 0H0 ») | Non | 20 caractères |
| **Notes** (zone de texte, 3 lignes) | Non | 10 000 caractères |
| **Contact principal** (case à cocher) | Non | booléen |

- Une note en bas de modale rappelle : « **\* Champs obligatoires** ».
- Les boutons sont **« Annuler »** et **« Enregistrer »**. Le bouton Enregistrer reste **désactivé** tant que le prénom **ou** le nom de famille est vide (`ContactsPage.tsx:460`).
- **Le menu déroulant « Entreprise » ne charge que les 100 premières entreprises** (`listCompanies({ perPage: 100 })`, `ContactsPage.tsx:90`), triées par nom. Si votre entreprise cible ne figure pas dans la liste (au-delà des 100 premières), créez d'abord le contact sans entreprise, puis rattachez-le depuis la modale de modification (qui, elle, sait préserver un lien existant ; voir la section 2.9).
- En cas de succès, un bandeau vert affiche « **Contact enregistré** » et la liste ainsi que les statistiques se rafraîchissent.

> **Rappel utile.** Cocher **« Contact principal »** à la création n'a d'effet que si le contact est aussi **rattaché à une entreprise**. Dans ce cas, le serveur retire automatiquement la marque « principal » aux autres contacts de la même entreprise, pour qu'il n'y ait **qu'un seul contact principal par entreprise** (`companies.py:795-799`).

### 2.9 Modale « Modifier le contact »

Ouverte par le bouton crayon d'une ligne (`ContactsPage.tsx:468-514`). Elle reprend **tous** les champs de la création, dans une disposition légèrement différente (prénom et nom sur la même ligne, l'étiquette du nom devient simplement « Nom »), et pré-remplit chaque valeur depuis la fiche existante, y compris les notes, l'adresse et l'état « Contact principal ».

- Les boutons sont **« Annuler »** et **« Enregistrer »** ; Enregistrer est désactivé si le prénom ou le nom est vide.
- En cas de succès, un bandeau vert affiche « **Contact modifié** ».

> **Protection de l'entreprise déjà liée (important).** Comme le menu déroulant ne charge que 100 entreprises, il pourrait arriver que l'entreprise actuellement rattachée au contact **ne s'y trouve pas** (par exemple la 150ᵉ entreprise, ou une entreprise désactivée). Pour éviter que l'enregistrement ne **réaffecte** silencieusement le contact à la première entreprise de la liste, la modale **injecte** une option qui préserve le lien existant (`ContactsPage.tsx:480-482`). Vous gardez donc le bon employeur même s'il n'apparaît pas dans les 100 chargés. C'est une différence de comportement avec la modale de création, qui ne connaît pas encore de lien à préserver.

### 2.10 Suppression d'un contact

Le bouton **corbeille** (dans le tableau ou une carte) déclenche une **confirmation du navigateur** : « **Supprimer ce contact ?** » (`ContactsPage.tsx:151-162`). Si vous confirmez, l'ERP appelle `DELETE /contacts/{id}`.

- **La suppression est définitive.** Le serveur exécute un effacement physique de la ligne (`DELETE FROM contacts`, `companies.py:945`). Il n'y a **pas** de corbeille ni de désactivation réversible, contrairement aux entreprises (qui passent en « Inactif »). **On ne peut pas annuler** une suppression depuis l'interface.
- **Contact lié ailleurs → refus clair (409).** Si le contact est référencé par un projet, un contrat, un bon de commande ou un autre document, le serveur **refuse** la suppression et renvoie le message : « **Ce contact est lié à un projet, contrat ou document. Dissociez-le avant de le supprimer.** » (`companies.py:953-959`). Dans ce cas, retirez d'abord le contact des documents concernés, puis réessayez.
- **Contact introuvable → 404** (« Contact non trouvé »).
- **Non-administrateur → 403** : le clic aboutit à un refus (voir la section 1.3).
- En cas de succès, la liste et les statistiques se rafraîchissent.

> **Bonne pratique.** Comme la suppression est irréversible et bloquée dès qu'un lien existe, préférez souvent la **modification** : videz ou corrigez les coordonnées plutôt que de supprimer, afin de conserver l'historique des documents qui pointent vers ce contact.

### 2.11 Assistant IA — Contacts

Ouvert par le bouton **« Assistant IA »**, dans une grande modale titrée « **Assistant IA — Contacts** », sous-titrée « Interroge tes contacts et crées-en de nouveaux sur confirmation. » (`ContactsPage.tsx:516-519`, `ContactsAssistantTab.tsx`).

**Fonctionnement (clavardage) :**

- Une zone de messages affiche l'échange sous forme de bulles.
- Une zone de saisie en bas : **Entrée** envoie le message, **Maj+Entrée** insère un saut de ligne. Le bouton **« Envoyer »** fait de même.
- À l'ouverture, un état vide propose trois exemples pour vous lancer :
  - « Qui sont mes contacts chez « Construction ABC » ? »
  - « Crée le contact Marie Gagnon, ingénieure, courriel marie@abc.com. »
  - « Liste les contacts sans adresse courriel. »
- Pendant le traitement, l'indicateur « **Analyse en cours…** » s'affiche.

**Ce que l'assistant peut faire :**

1. **Lire et répondre** : il interroge vos données réelles (les contacts et les entreprises) pour répondre à vos questions.
2. **Proposer la création d'un contact** : lorsqu'il comprend que vous voulez ajouter quelqu'un, il **n'écrit rien tout de suite**. Il affiche une **carte de proposition** avec l'aperçu des champs (nom, courriel, rôle…), marquée « **En attente de confirmation** », et deux boutons : **« Confirmer »** et **« Annuler »**. Ce n'est qu'à la confirmation que le contact est réellement créé, et l'assistant répond alors « **Fait. Contact créé.** ». La liste et les statistiques se rafraîchissent aussitôt.

> **Garde-fous.** L'assistant **ne crée un contact que sur votre confirmation explicite** ; il ne modifie ni ne supprime aucune fiche (la version 1 se limite à la **création**). En lecture, il est restreint à vos seules tables de contacts et d'entreprises et ne peut pas atteindre les données sensibles (paie, employés, utilisateurs, secrets, autres tenants). Voir les sections 4.8 et 5.5 pour les paramètres, les coûts et la sécurité.

---

## 3. Processus pas à pas

### 3.1 Consulter et rechercher un contact

1. Ouvrez le menu latéral → **GESTION** → **Contacts**.
2. La liste s'affiche (20 par page), triée par nom de famille. Les 4 cartes du haut vous donnent les totaux du tenant.
3. Pour retrouver quelqu'un, tapez dans le champ de recherche : un **nom**, un **courriel**, un **nom d'entreprise** ou un **rôle**. Les résultats se rafraîchissent environ 300 millisecondes après votre dernière frappe.
4. Cliquez un en-tête de colonne pour **trier** la page affichée ; glissez les poignées pour **élargir** une colonne.
5. Utilisez la pagination en bas pour parcourir les pages. Cliquez **« Effacer »** pour revenir à la liste complète.

### 3.2 Créer un contact rattaché à une entreprise

1. Cliquez **« Nouveau Contact »**.
2. Saisissez le **Prénom** et le **Nom de famille** (obligatoires).
3. Dans le menu déroulant **« Entreprise »**, choisissez l'employeur (les 100 premières entreprises, triées par nom).
4. Renseignez au besoin le courriel, le téléphone, le mobile, le rôle / poste, la fonction, le département et l'adresse.
5. Cochez éventuellement **« Contact principal »** pour en faire l'interlocuteur principal de cette entreprise.
6. Cliquez **« Enregistrer »**. Bandeau « Contact enregistré » ; la liste se met à jour.

> Si l'entreprise n'apparaît pas dans le menu (au-delà des 100 premières), voyez la section 3.4.

### 3.3 Créer un contact autonome (sans entreprise)

1. Cliquez **« Nouveau Contact »**.
2. Saisissez au minimum le **Prénom** et le **Nom de famille**.
3. Laissez le menu **« Entreprise »** sur « -- Sélectionner -- ».
4. Renseignez les coordonnées utiles, puis **« Enregistrer »**.

Le contact apparaît dans la liste avec `--` dans la colonne Entreprise. Vous pourrez le rattacher plus tard.

### 3.4 Rattacher un contact à une entreprise (ou changer son employeur)

1. Sur la ligne du contact, cliquez le bouton **crayon** (Modifier).
2. Dans **« Entreprise »**, choisissez la nouvelle entreprise. Si le contact était déjà lié à une entreprise absente du menu, cette dernière reste **préservée** (elle est réinjectée dans la liste) : vous ne perdez pas le lien existant par mégarde.
3. Cliquez **« Enregistrer »**.

**Effets côté serveur, automatiques et dans une seule transaction** (`companies.py:860-905`) :
- Si le contact était le **contact principal** de son ancienne entreprise, le pointeur « contact principal » de cette **ancienne** entreprise est remis à zéro (pour ne pas laisser un pointeur orphelin).
- S'il reste marqué « principal » **et** rattaché à sa nouvelle entreprise, les autres contacts de cette nouvelle entreprise perdent la marque « principal », afin de garder **un seul principal par entreprise**.

### 3.5 Désigner le contact principal d'une entreprise

Il existe **deux angles** pour la même idée de « personne-ressource principale » :

- **Depuis le module Contacts** — cochez **« Contact principal »** dans la modale de création ou de modification. Cela active le badge bleu **« Principal »** dans la liste et garantit l'unicité côté contacts (les autres sont démarqués).
- **Depuis le module Entreprises** (manuel 04) — la fiche d'une entreprise offre un menu **« Contact principal »** qui enregistre un pointeur côté entreprise (`companies.contact_principal_id`).

Ces deux réglages visent le même but mais sont **stockés séparément**. Le badge « Principal » de la page Contacts reflète la case cochée sur le contact. Pour une cohérence parfaite, désignez la personne des deux côtés. Le serveur veille déjà à ne pas laisser de pointeur orphelin quand un contact change d'entreprise (voir la section 3.4).

### 3.6 Modifier les coordonnées d'un contact

1. Cliquez le **crayon** sur la ligne du contact.
2. Modifiez le courriel, le téléphone, le mobile, l'adresse, les notes, etc.
3. Cliquez **« Enregistrer »**. Bandeau « Contact modifié ».

Seuls les champs autorisés sont pris en compte ; le serveur enregistre aussi la date de dernière modification (`updated_at`).

### 3.7 Supprimer un contact

1. Cliquez la **corbeille** sur la ligne (ou la carte).
2. Confirmez « Supprimer ce contact ? ».
3. Résultats possibles :
   - **Succès** : le contact disparaît de la liste (suppression **définitive**).
   - **« Ce contact est lié à un projet, contrat ou document… »** : dissociez-le d'abord des documents concernés, puis réessayez.
   - **Refus (403)** : vous n'êtes pas administrateur ; demandez à un administrateur du tenant.

### 3.8 Créer un contact avec l'assistant IA

1. Cliquez **« Assistant IA »**.
2. Décrivez la personne en langage naturel, par exemple : « Crée le contact Marie Gagnon, ingénieure chez Construction ABC, courriel marie@abc.com, mobile 514-555-1234. »
3. L'assistant affiche une **carte de proposition** (« En attente de confirmation ») avec l'aperçu des champs.
4. Vérifiez les valeurs, puis cliquez **« Confirmer »** (ou **« Annuler »** pour corriger votre demande).
5. À la confirmation, le contact est créé (« Fait. Contact créé. ») et la liste se rafraîchit.

> Vous pouvez aussi **interroger** vos contacts sans rien créer : « Qui sont mes contacts chez Construction ABC ? », « Liste les contacts sans courriel. »

---

## 4. Référence

### 4.1 Modèle de données — table `contacts` (par tenant)

| Colonne | Type | Obligatoire | Notes |
|---|---|---|---|
| `id` | entier (clé primaire) | — | Auto-incrémenté |
| `company_id` | entier, **nullable** | Non | Clé étrangère vers `companies.id` ; `NULL` = contact autonome ; une valeur ≤ 0 est normalisée en `NULL` |
| `prenom` | texte (≤ 100) | **Oui** | Rejeté si vide ou composé uniquement d'espaces |
| `nom_famille` | texte (≤ 100) | **Oui** | Idem |
| `email` | texte (≤ 255) | Non | **Aucune** validation de format |
| `telephone` | texte (≤ 50) | Non | Texte libre ; mis en forme à l'affichage |
| `mobile` | texte (≤ 50) | Non | Distinct du téléphone fixe |
| `role_poste` | texte (≤ 150) | Non | Ex. « Directrice des achats » |
| `fonction` | texte (≤ 150) | Non | Complément libre du rôle |
| `departement` | texte (≤ 150) | Non | Ex. « Comptabilité » |
| `adresse` | texte (≤ 300) | Non | Ajoutée à la volée sur les anciens schémas |
| `ville` | texte (≤ 120) | Non | Idem |
| `province` | texte (≤ 80) | Non | Idem |
| `code_postal` | texte (≤ 20) | Non | Idem |
| `est_principal` | booléen | Non | Défaut `false` ; unique par entreprise (réconcilié côté serveur) |
| `notes` | texte (≤ 10 000) | Non | Texte libre |
| `created_at` | horodatage | — | Fixé à la création |
| `updated_at` | horodatage | — | Mis à jour à chaque modification |

> **Migration automatique.** Sur les tenants anciens, certaines colonnes (`adresse`, `ville`, `province`, `code_postal`, `mobile`, `fonction`, `departement`, `est_principal`) peuvent manquer. La fonction `_ensure_contact_address_cols` (`companies.py:36-71`) les ajoute **automatiquement** au premier accès, une fois par schéma. Vous n'avez rien à faire.

### 4.2 Correspondance des noms (camelCase / snake_case)

Le client transforme automatiquement les noms entre l'interface (camelCase) et le serveur (snake_case) :

`companyId` ↔ `company_id` ; `nomFamille` ↔ `nom_famille` ; `rolePoste` ↔ `role_poste` ; `codePostal` ↔ `code_postal` ; `estPrincipal` ↔ `est_principal` ; `companyNom` ↔ `company_nom` (le nom de l'entreprise, calculé par jointure) ; `createdAt` ↔ `created_at`. Les autres champs gardent le même nom (`prenom`, `email`, `telephone`, `mobile`, `fonction`, `departement`, `adresse`, `ville`, `province`, `notes`).

### 4.3 Endpoints de l'API

Tous sont servis sous le préfixe `/api/erp/v1` et exigent l'authentification plus un contexte de tenant valide.

| Méthode et chemin | Fonction (fichier:ligne) | Garde | Rôle |
|---|---|---|---|
| `GET /contacts` | `list_contacts` (`companies.py:621`) | `get_current_user` | Lister et rechercher (paginé) |
| `GET /contacts/stats` | `contacts_stats` (`companies.py:707`) | `get_current_user` | 4 compteurs globaux |
| `POST /contacts` | `create_contact` (`companies.py:753`) | `get_current_user` | Créer un contact |
| `PUT /contacts/{id}` | `update_contact` (`companies.py:829`) | `get_current_user` | Modifier un contact |
| `DELETE /contacts/{id}` | `delete_contact` (`companies.py:934`) | **`require_tenant_admin_or_role()`** | Supprimer (admin) |
| `POST /contacts/ai/chat` | `contacts_ai_chat` (`contacts_ai.py:284`) | `get_current_user` + garde IA + crédits | Clavardage de l'assistant |
| `POST /contacts/ai/confirm-action` | `confirm_contacts_action` (`contacts_ai.py:423`) | `get_current_user` + garde IA + crédits | Confirmer la création proposée |

**Paramètres de `GET /contacts` :**

| Paramètre | Type | Défaut | Notes |
|---|---|---|---|
| `page` | entier ≥ 1 | 1 | Numéro de page |
| `per_page` | entier 1-100 | 20 | Taille de page (l'interface utilise 20) |
| `search` | texte | — | Recherche sur nom, courriel, entreprise, rôle |
| `company_id` | entier | — | Filtrer sur une entreprise précise |

Réponse : `{ items, total, page, per_page }`, triée par nom de famille puis identifiant. Chaque élément inclut `company_nom` (le nom de l'entreprise, par jointure). **Note** : il n'existe **pas** d'alias `limit` sur cet endpoint (contrairement à la liste des entreprises) ; utilisez `per_page`.

**Codes de réponse notables :**

| Code | Signification |
|---|---|
| **200** | Succès (avec message : « Contact créé », « Contact mis à jour », « Contact supprimé ») |
| **400** | « Contexte tenant manquant », « Entreprise introuvable » (entreprise liée inexistante dans le tenant), ou « Aucun champ à modifier » (mise à jour vide) |
| **403** | Écriture en mode consultation, ou suppression tentée par un non-administrateur |
| **404** | « Contact non trouvé » |
| **409** | Suppression refusée : le contact est lié à un projet, un contrat ou un document |
| **402** | Crédits IA épuisés (endpoints de l'assistant) |
| **429** | Trop de requêtes (limite de débit dépassée) |
| **503** | Service IA indisponible (client Anthropic non configuré) |

### 4.4 Validations et bornes

| Niveau | Règle | Effet |
|---|---|---|
| Interface | Bouton « Enregistrer » désactivé si prénom ou nom vide | Empêche l'envoi |
| Serveur (Pydantic) | `prenom` et `nom_famille` obligatoires, non vides après nettoyage des espaces | 422 sinon |
| Serveur | Longueurs maximales (voir 4.1) | 422 si dépassement |
| Serveur | `company_id` ≤ 0 → `NULL` | Normalisation silencieuse (contact détaché) |
| Serveur | Entreprise liée doit exister dans le tenant | 400 « Entreprise introuvable » |
| Serveur | Unicité du contact principal par entreprise | Les autres sont démarqués, en transaction |
| Serveur | `updated_at` renseigné à chaque modification | Horodatage automatique |

> **Le courriel n'est pas validé** : le champ accepte n'importe quel texte de 255 caractères ou moins (pas d'exigence de « @ » ni de domaine). Prenez soin de bien le saisir, car il sert au rapprochement automatique des courriels (voir la section 5.4).

### 4.5 Statistiques — mode de calcul

`GET /contacts/stats` agrège **tout le tenant** en une seule requête (`companies.py:724-737`) :
- **total** = `COUNT(*)` ;
- **companies** = `COUNT(DISTINCT company_id)` en ignorant les contacts autonomes ;
- **withEmail** = nombre de contacts au courriel non vide ;
- **withPhone** = nombre de contacts au téléphone non vide.

Ces chiffres sont **indépendants** de la recherche et de la pagination.

### 4.6 Recherche — portée exacte

La recherche (`companies.py:647-660`) applique un « contient » insensible à la casse sur quatre champs, combinés par « OU » :
1. le nom complet (`prenom` + `nom_famille`) ;
2. le courriel (`email`) ;
3. le nom de l'entreprise rattachée (`companies.nom`) ;
4. le rôle ou poste (`role_poste`).

Les champs vides sont gérés sans erreur, les caractères spéciaux sont neutralisés, et le terme est tronqué à 100 caractères. Le décompte total tient compte de la même jointure, si bien que la pagination reste exacte.

### 4.7 Limites de débit (par adresse IP, fenêtre de 60 secondes)

| Endpoint | Limite |
|---|---|
| `POST /contacts/ai/chat` | 20 par minute |
| `POST /contacts/ai/confirm-action` | 30 par minute |
| CRUD `/contacts` (non IA) | tranche générale de 1500 par minute |

Un dépassement renvoie **429** avec un en-tête `Retry-After: 60`.

### 4.8 Assistant IA — paramètres, sécurité et coûts

- **Modèle** : `claude-sonnet-4-6`, jusqu'à 8000 jetons par réponse, boucle d'outils limitée à 5 itérations (`contacts_ai.py`).
- **Deux outils** :
  - `recherche_bd` — lecture seule, **au plus 50 lignes**, restreinte à la liste blanche `{contacts, companies}`. Les tables sensibles (employés, paie, utilisateurs, courriels, crédits IA, secrets, jetons, autres tenants…) sont **bloquées**, tout comme les références à un autre schéma. Le moteur SQL sous-jacent n'autorise que des `SELECT` en lecture seule, avec délai maximal.
  - `proposer_contact` — **n'écrit rien** ; il valide les champs (mêmes règles que `ContactCreate`) et retourne une **proposition** à confirmer.
- **Écriture uniquement sur confirmation** : `POST /contacts/ai/confirm-action` **revalide** entièrement les champs, puis délègue à la fonction `create_contact` habituelle (mêmes gardes de tenant et d'entreprise). Aucun identifiant de tenant n'est accepté depuis le navigateur (protection contre l'accès indirect aux données d'autrui).
- **Périmètre version 1** : **création** de contact seulement. La modification par l'IA est prévue plus tard.
- **Coûts (crédits IA)** : chaque **tour de clavardage** est facturé au tenant au **coût réel du modèle majoré de 30 %** (`contacts_ai.py:399`), débité des crédits prépayés. La **confirmation** de la création, elle, **n'est pas facturée** : créer un contact via l'assistant ne coûte que le ou les tours de clavardage qui ont mené à la proposition. Si les crédits sont épuisés, le clavardage renvoie **402** ; une recharge automatique peut se déclencher selon la configuration du compte. Le service renvoie **503** si l'IA n'est pas configurée.

### 4.9 Composants et raccourcis d'interface

- **Composants** : `CommandBar`, `StatCard`, `Card`, `Modal` (taille `lg` pour les fiches, `xl` pour l'assistant), `Input`, `Select`, `Textarea`, `Button`, `Badge`, `Pagination`, `SortableHeader`, `MessageBubble` (assistant).
- **Comportements** : recherche différée de 300 ms ; tri client par colonne (`useSortable`) ; redimensionnement des colonnes (`useColumnResize`) ; protection anti-course qui ignore les réponses de requêtes dépassées ; mise en forme du téléphone (`formatPhone`).
- **Assistant** : **Entrée** pour envoyer, **Maj+Entrée** pour un saut de ligne.

---

## 5. Intégrations et FAQ

### 5.1 Avec le module Entreprises (manuel 04)

- Un contact pointe vers une entreprise par `company_id` (facultatif). La fiche d'une entreprise affiche **ses** contacts, triés avec le contact principal en tête.
- Une entreprise peut désigner un **contact principal** (pointeur `contact_principal_id`). Ce pointeur et la case « Contact principal » du contact sont **deux réglages distincts** mais complémentaires (voir la section 3.5). Le serveur nettoie le pointeur de l'ancienne entreprise quand un contact déménage.
- **Supprimer une entreprise** la désactive (elle passe « Inactif ») ; ses contacts **restent** dans le carnet. Leur colonne « Entreprise » peut afficher l'entreprise désactivée.

### 5.2 Avec le CRM et les opportunités (manuel 06)

Les contacts sont sélectionnables comme interlocuteurs d'une opportunité. C'est **là**, et non dans le module Contacts, que vivent les **interactions** et **activités** (appels, courriels, rencontres) et la qualification commerciale. Le module Contacts ne tient aucun journal d'interactions.

### 5.3 Avec les Soumissions, Projets et Contrats (manuels 08, 09)

Un contact peut être le « contact client » d'une soumission, d'un projet ou d'un contrat. Ces liens expliquent le refus **409** à la suppression : tant qu'un document pointe vers le contact, il faut d'abord l'en dissocier.

### 5.4 Avec les Courriels (manuel 23)

Le nom d'un contact et son **courriel** servent au **rapprochement automatique** des courriels entrants avec la bonne personne. D'où l'importance d'un courriel exact et bien orthographié, puisque le module ne valide pas son format.

### 5.5 Avec l'assistant IA (crédits)

L'assistant du module Contacts partage le même **portefeuille de crédits IA** que les autres assistants de l'ERP (`public.ai_prepaid_credits`). Le clavardage consomme des crédits ; la confirmation d'une création n'en consomme pas. Voir aussi le manuel 25 (Assistant IA — vue d'ensemble).

### 5.6 Foire aux questions

**Le module a-t-il des « types de contacts » ou un journal d'interactions ?**
Non. Il n'existe **aucun** champ de type ou de catégorie, ni de journal d'appels / courriels / rencontres. Le rôle, la fonction et le département sont du **texte libre**. Les interactions et le pipeline commercial sont dans le module CRM (manuel 06).

**Puis-je exporter ou imprimer mes contacts ?**
Non. Le module n'offre ni export (CSV, PDF), ni impression, ni téléversement de fichier. Pour un tirage massif, passez par un administrateur de la base de données.

**Puis-je sélectionner plusieurs contacts et les supprimer d'un coup ?**
Non. Il n'y a pas d'action de masse ; chaque suppression est individuelle et confirmée.

**Pourquoi mon bouton « Supprimer » donne-t-il une erreur ?**
Deux causes. Soit vous n'êtes **pas administrateur** (refus 403) : demandez à un administrateur. Soit le contact est **lié** à un projet, un contrat ou un document (refus 409) : dissociez-le d'abord.

**La suppression est-elle réversible ?**
**Non.** Contrairement aux entreprises (désactivées et réversibles), un contact supprimé est **effacé définitivement**. En cas de doute, modifiez la fiche plutôt que de la supprimer.

**Pourquoi une entreprise n'apparaît-elle pas dans le menu « Entreprise » à la création ?**
Le menu ne charge que les **100 premières** entreprises. Créez le contact sans entreprise, puis rattachez-le depuis la modale de **modification**, qui sait préserver le lien même hors de cette liste de 100.

**Le contact principal d'une entreprise est-il pré-rempli automatiquement dans une soumission ou une opportunité ?**
Non. Aucun pré-remplissage automatique. Vous choisissez l'interlocuteur au moment de créer le document.

**Puis-je donner à un contact un accès à l'ERP ?**
Non. Un contact est une **donnée**, pas un compte. Les comptes utilisateurs se gèrent dans l'Administration (manuel 28 / Configuration).

**Deux contacts peuvent-ils avoir le même courriel ou le même nom ?**
Oui. Il n'y a **aucune** contrainte d'unicité ni détection de doublon. Recherchez avant de créer pour éviter les doublons.

**La recherche par entreprise ou par rôle fonctionne-t-elle vraiment ?**
Oui. La recherche couvre bel et bien le **nom**, le **courriel**, le **nom de l'entreprise** et le **rôle / poste**, comme l'indique le texte d'invite. (C'était une limite des anciennes versions ; ce n'est plus le cas.)

**Quelle est la différence entre « Rôle / Poste », « Fonction » et « Département » ?**
Il n'y a pas de règle imposée : ce sont trois champs de texte libre. Convention utile : `Rôle / Poste` = le titre exact (« Directrice des achats ») ; `Fonction` = une catégorie plus générale (« Direction ») ; `Département` = l'unité (« Achats »). Seul le rôle / poste s'affiche dans la liste ; la fonction et le département se voient à l'ouverture de la fiche.

**Le badge « Principal » reflète-t-il le contact principal choisi côté entreprise ?**
Le badge reflète la case « Contact principal » **cochée sur le contact**. Pour une cohérence parfaite avec la fiche d'entreprise, désignez la personne des deux côtés (section 3.5).

**Combien de contacts puis-je enregistrer ?**
Aucune limite fixe. La performance dépend de la base de données ; la pagination reste à 20 par page.

**L'adresse d'un contact sert-elle à la facturation ou à la livraison ?**
Non. Les adresses de facturation et de livraison se gèrent au niveau de l'entreprise ou directement dans les documents (soumission, facture, bon de commande). L'adresse du contact est purement informative.

---

## 6. Récapitulatif

- **Mission** : carnet des **personnes physiques**, rattachées ou non à une entreprise ; module jumeau d'Entreprises (manuel 04), mais page distincte.
- **Accès** : menu latéral → groupe **GESTION** → **Contacts** (icône `Users`), adresse `/contacts`.
- **Code** : pas de `routers/contacts.py` — le CRUD vit dans **`companies.py`** (5 endpoints `/contacts` : liste, stats, création, modification, suppression) ; l'assistant IA est dans **`contacts_ai.py`** ; pas de `api/contacts.ts` — le client est dans **`companies.ts`**.
- **Statistiques** : 4 cartes **globales** au tenant (Contacts, Entreprises, Avec courriel, Avec Tél.) via `GET /contacts/stats` — indépendantes de la page et de la recherche.
- **Recherche** : couvre **nom + courriel + entreprise + rôle**, côté serveur, sur tout le tenant.
- **Liste** : tableau triable (par colonne, sur la page courante) et redimensionnable ; cartes sur mobile ; **20 par page** ; tri serveur par nom de famille.
- **Champs** : prénom et nom **obligatoires** ; courriel, téléphone, mobile, rôle, fonction, département, adresse, notes ; case **« Contact principal »** présente à la création **et** à la modification.
- **Rattachement** : `company_id` facultatif ; validé dans le tenant ; menu déroulant limité à **100** entreprises à la création, mais le lien existant est **préservé** en modification.
- **Contact principal** : **un seul par entreprise**, réconcilié automatiquement par le serveur, en transaction.
- **Suppression** : **définitive** (effacement physique), réservée aux **administrateurs** ; **refus 409** clair si le contact est lié à un projet, un contrat ou un document. Le bouton reste visible pour tous (refus 403 côté serveur pour un non-admin).
- **Assistant IA** : lit vos contacts / entreprises et **crée** un contact **sur confirmation** (version 1 = création seulement) ; le clavardage consomme des crédits, la confirmation non.
- **Sécurité** : isolation stricte par tenant ; mode consultation en lecture seule sur abonnement inactif ; aucune donnée d'un autre tenant accessible.
- **Absences à connaître** : pas de type / interaction, pas d'export ni d'impression, pas de téléversement ni de photo, pas d'action de masse, pas de page de détail, pas de détection de doublon, pas de validation de format du courriel.

---

**Fichiers sources vérifiés** :
- `ERP_REACT/backend/routers/companies.py` — CRUD des contacts (partagé avec Entreprises) : liste, statistiques, création, modification, suppression, gardes et transactions.
- `ERP_REACT/backend/routers/contacts_ai.py` — assistant IA (patron proposer / confirmer, liste blanche de lecture, facturation des crédits).
- `ERP_REACT/frontend/src/pages/ContactsPage.tsx` — page `/contacts` (liste, statistiques, recherche, modales).
- `ERP_REACT/frontend/src/components/contacts/ContactsAssistantTab.tsx` — modale de l'assistant IA.
- `ERP_REACT/frontend/src/api/companies.ts` (contacts) et `ERP_REACT/frontend/src/api/contactsAi.ts` (assistant) — clients d'API.
- `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json` — libellés `contacts.*` (lignes 1039-1118).

**Manuels liés** :
- Manuel 04 — Entreprises (clients et fournisseurs) : `03-gestion-entreprises.md`
- Manuel 06 — CRM et opportunités (interactions, pipeline) : `05-gestion-crm-opportunites.md`
- Manuel 08 — Soumissions (devis) : `07-ventes-soumissions.md`
- Manuel 09 — Projets : `08-ventes-projets.md`
- Manuel 23 — Courriels (rapprochement automatique par courriel) : `21-communication-emails.md`
- Manuel 25 — Assistant IA (vue d'ensemble) : `24-communication-assistant-ia.md`
- Manuel 28 — Configuration (comptes utilisateurs, catégories) : `30-configuration.md`

---

*Manuel ERP Constructo AI — Module 04 Contacts — v3.0 vérifié contre le code source — juillet 2026*
