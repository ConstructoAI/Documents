# Module 04 — Entreprises (clients et fournisseurs)

> **Version** : 3.0 (refonte vérifiée contre le code source actuel, juillet 2026)
> **Code de référence** :
> - `ERP_REACT/backend/routers/companies.py` — CRUD entreprises et contacts ; 11 endpoints réels (`/companies*` et `/contacts*`), soft-delete entreprise, garde admin sur la suppression.
> - `ERP_REACT/backend/routers/entreprises_ai.py` — assistant IA Entreprises ; 2 endpoints (`/entreprises/ai/chat`, `/entreprises/ai/confirm-action`), patron proposer puis confirmer.
> - `ERP_REACT/frontend/src/pages/CompaniesPage.tsx` — page `/entreprises` (liste, panneau de détail, modales, statistiques).
> - `ERP_REACT/frontend/src/components/entreprises/EntreprisesAssistantTab.tsx` — modale de l'assistant IA.
> - `ERP_REACT/frontend/src/api/companies.ts` et `api/entreprisesAi.ts` — clients d'API.
> - Libellés : `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json`, clés `entreprises.*` (lignes 1119-1251), `contacts.*` (1039) et `stages.*` (1252-1277).
> **Préfixe des API** : `/api/erp/v1` (`erp_config.py:9`). Exemple réel : `GET /api/erp/v1/companies`.
> **Tables PostgreSQL** (par schéma de tenant) : `companies` (entité principale), `contacts` (entité couplée, aussi documentée au manuel 05) ; crédits de l'assistant IA sur `public.ai_prepaid_credits`.
> **Cadrage** : ce module gère le **référentiel des entités tierces** (clients, fournisseurs, sous-traitants, consultants, organismes) du tenant. Il **ne couvre pas** : le carnet de personnes physiques détaillé (manuel 05 — Contacts, module de menu distinct), le cycle commercial et le pipeline (manuel 06 — CRM et opportunités), les soumissions (manuel 08), les projets (manuel 09), la facturation (manuel 15).

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

Le module **Entreprises** est le carnet central des **organisations tierces** avec lesquelles votre entreprise fait affaire : clients (résidentiels, commerciaux, industriels, municipalités), fournisseurs de matériaux, sous-traitants spécialisés, consultants et ingénieurs, architectes, arpenteurs-géomètres, organismes de contrôle, institutions financières et assureurs.

Chaque fiche d'entreprise sert de **point d'ancrage** pour le reste de l'ERP. Une fois une entreprise créée, elle devient sélectionnable comme client dans les soumissions et les projets, et chaque fiche affiche en lecture seule :

- ses **contacts rattachés** (personnes physiques) ;
- ses **soumissions récentes** (les 5 dernières) ;
- ses **projets récents** (les 5 derniers).

Ces documents liés sont rattachés par **clé étrangère** (`client_company_id`), c'est-à-dire par un vrai lien fiable, et non par une simple correspondance de nom (`CompaniesPage.tsx:315-319`).

### 1.2 Accès par le menu latéral

- **Menu latéral** → groupe **« GESTION »** → **« Entreprises »** (icône `Building2`), situé entre « Tableau de Bord / Suivi » et « Contacts » (`Sidebar.tsx:48`, `nav.json:6-8`).
- **Adresse** : `/entreprises`.
- **Composant** : `CompaniesPage` (chargé à la demande, `App.tsx:99,223`), protégé par l'authentification (`ProtectedRoute`).
- **Titre de la page** : « Entreprises » (`CompaniesPage.tsx:638`).
- **Une seule vue principale** : il n'y a pas d'onglets. La navigation se fait par la recherche, le filtre de type, et le panneau de détail qui s'ouvre sur le côté.

> **« Contacts » est un module séparé.** La page Entreprises n'a pas d'onglet Contacts ; les personnes physiques ont leur propre page (`/contacts`, `Sidebar.tsx:49`, manuel 05). Les deux modules restent étroitement liés : le panneau de détail affiche les contacts d'une entreprise, la modale de création propose un menu déroulant « Contact principal », et l'assistant IA peut créer les deux. Voir la section 5.

### 1.3 Permissions et rôles

| Action | Garde côté serveur | Qui peut le faire |
|---|---|---|
| Consulter, rechercher (lecture) | `get_current_user` | Tout utilisateur authentifié du tenant |
| Créer une entreprise ou un contact | `get_current_user` | Tout utilisateur authentifié du tenant |
| Modifier une entreprise ou un contact | `get_current_user` | Tout utilisateur authentifié du tenant |
| **Supprimer (désactiver) une entreprise** | **`require_tenant_admin_or_role()`** | **Administrateur du tenant ou rôle équivalent** |
| **Supprimer un contact** | **`require_tenant_admin_or_role()`** | **Administrateur du tenant ou rôle équivalent** |
| Assistant IA (clavardage + confirmation) | `get_current_user` + garde IA + crédits | Tout utilisateur authentifié, si le solde de crédits IA le permet |

- **Isolation par tenant** : chaque requête applique `db.set_tenant(conn, user.schema)`. Sans contexte de tenant, l'API renvoie **HTTP 400 « Contexte tenant manquant »**. Une entreprise d'un tenant n'est jamais visible d'un autre tenant.
- La suppression exige un **administrateur** : la garde `require_tenant_admin_or_role()` autorise l'utilisateur dont le drapeau `is_admin` est vrai (relu côté serveur, donc infalsifiable), ou dont le rôle est `admin`, ou un super-administrateur de plateforme. Un employé ordinaire peut créer et modifier une fiche, mais **pas la désactiver**.
- **Mode consultation (abonnement)** : si le compte est en lecture seule (sans Stripe ou abonnement annulé), toutes les écritures (`POST`, `PUT`, `PATCH`, `DELETE`) renvoient **403 « Mode consultation »** ; seules les consultations (`GET`) passent. Si l'entreprise du tenant est désactivée, la session est coupée (**401**). Ce contrôle s'applique à tous les endpoints du module.

### 1.4 Composants du module

1. **Bandeau de statistiques** — 4 compteurs globaux du tenant (Total, Clients, Fournisseurs, Sous-traitants).
2. **Barre de commandes** — création, assistant IA, rafraîchissement, recherche, filtre par type.
3. **Liste des entreprises** — tableau sur ordinateur, cartes sur mobile, avec pagination (20 par page).
4. **Panneau de détail latéral** — coordonnées, contacts liés, soumissions et projets récents.
5. **Modales Créer / Modifier** — un même formulaire partagé pour les deux opérations.
6. **Assistant IA — Entreprises** — clavardage qui lit les données réelles et propose des créations sur confirmation.

### 1.5 Ce que le module ne fait pas (pour cadrer les attentes)

- **Aucun export** (CSV ou PDF) ni impression ni téléversement de fichiers, de logo ou de documents sur une fiche d'entreprise.
- **Aucune suppression définitive** depuis l'interface : la « suppression » est une désactivation réversible (voir section 3.5). La réactivation n'est pas non plus exposée dans l'interface.
- **L'assistant IA ne crée que** (entreprise ou contact). Il ne modifie ni ne supprime aucune fiche (voir section 2.8).
- **Aucune détection de doublon** : deux entreprises de même nom peuvent coexister ; utilisez la recherche avant de créer.
- **Aucune validation de format** des numéros de taxe, du code postal ou du téléphone : ce sont des champs de texte libre.
- **Aucune hiérarchie** société mère / filiale : une entreprise est une fiche à plat.

---

## 2. Interface

### 2.1 Disposition générale

La page empile, du haut vers le bas (`CompaniesPage.tsx`) :

1. les bandeaux d'alerte (erreur ou succès) ;
2. le titre « Entreprises » et les 4 cartes de statistiques ;
3. la barre de commandes ;
4. la liste (tableau sur grand écran, cartes empilées sur mobile) avec sa pagination.

Au clic sur une entreprise, un **panneau de détail** s'ouvre à droite (il occupe environ 40 % de la largeur sur ordinateur) ; sur mobile, il occupe tout l'écran avec un bouton « Retour à la liste ».

### 2.2 Bandeau de statistiques (4 cartes)

Quatre compteurs, calculés sur **l'ensemble du tenant** par l'endpoint `GET /companies/stats` — et non sur la seule page affichée (`CompaniesPage.tsx:641-646`, `companies.py:320-356`) :

| Carte | Couleur | Calcul côté serveur |
|---|---|---|
| **Total** | bleu | `COUNT(*)` des entreprises actives |
| **Clients** | bleu | `type_company` contient « Client » (`ILIKE '%Client%'`) |
| **Fournisseurs** | vert | `type_company` contient « Fournisseur » (`ILIKE '%Fournisseur%'`) |
| **Sous-traitants** | violet | `type_company` contient « Sous-traitant » (`ILIKE '%Sous-traitant%'`) |

> **À savoir — les 3 sous-cartes ne couvrent pas tout le Total.** La catégorisation se fait par correspondance de sous-chaîne sur le type (`companies.py:342-348`). Une entreprise de type « Entrepreneur général », « Architecte », « Municipalité », « Assureur »… compte dans **Total** mais dans **aucune** des trois sous-cartes. Il est donc normal que `Clients + Fournisseurs + Sous-traitants` soit **inférieur** au Total. Ce n'est pas une erreur d'affichage.
>
> Les entreprises désactivées sont **exclues** de ces compteurs par défaut, comme de la liste (voir section 3.5).

### 2.3 Barre de commandes

À gauche (`CompaniesPage.tsx:648-676`) :

- **« Nouvelle entreprise »** (icône `Plus`, bouton principal) → ouvre la modale de création.
- **« Assistant IA »** (icône `Sparkles`) → ouvre la modale de l'assistant IA (section 2.8).
- **« Rafraîchir »** (icône `RefreshCw` qui tourne pendant le chargement) → recharge la liste et les statistiques.

À droite :

- **Champ de recherche** (« Rechercher par nom, courriel, ville… ») avec un délai de 300 ms après la frappe avant de lancer la requête (`CompaniesPage.tsx:228-231`). Le serveur cherche sur le **nom**, le **courriel** et la **ville** (`companies.py:264-271`) ; les caractères génériques `%` et `_` sont neutralisés, la recherche est insensible à la casse.
- **Filtre par type** (menu déroulant) : « Tous les types » plus les 14 types d'entreprise (`FILTER_TYPE_OPTIONS`, `CompaniesPage.tsx:111-114`).

### 2.4 Liste des entreprises

**Sur ordinateur** — tableau (`CompaniesPage.tsx:687-768`). Les colonnes sont triables (tri effectué côté client, sur les 20 lignes de la page affichée) et redimensionnables :

| Colonne | Contenu | Clé de tri |
|---|---|---|
| **Nom** | nom de l'entreprise, avec le courriel en sous-ligne | `nom` |
| **Type** | badge coloré selon le type (`TYPE_COLORS`, `:116-135`) | `typeCompany` |
| **Contact** | le **téléphone** formaté (voir note) | `telephone` |
| **Ville** | ville | `ville` |
| **Actions** | boutons Modifier (crayon) et Supprimer (corbeille) | — |

> **Attention au libellé de la colonne « Contact ».** Elle affiche en réalité le **numéro de téléphone** de l'entreprise (clé de tri `telephone`), et non le nom d'un contact principal. Le libellé peut prêter à confusion.

- Un clic sur une ligne ouvre le panneau de détail.
- **Pagination** : 20 entreprises par page (`CompaniesPage.tsx:190`). Un composant de pagination apparaît au-delà d'une page. Une ligne indique le nombre total : « X entreprise(s) ».

**Sur mobile** — cartes empilées (`:771-824`) : nom, badge de type, bouton Supprimer, puis trois lignes courriel / téléphone / ville avec leurs icônes.

**États vides** (`:749-763` ; mobile `:810-823`) :

- Avec un filtre ou une recherche actifs → « Aucun résultat pour ces critères » plus un bouton **« Réinitialiser les filtres »**.
- Sans aucune donnée → « Commencez par ajouter une entreprise. » plus un bouton **« Nouvelle entreprise »**.

**Bandeaux** (`:635-636`) : un bandeau d'erreur (que vous pouvez fermer) et un bandeau de succès qui s'efface automatiquement après 3 secondes (`:242-246`).

### 2.5 Panneau de détail

Ouvert au clic sur une entreprise (`renderDetailPanel`, `CompaniesPage.tsx:473-631`).

**En-tête** : nom, badge de type, secteur d'activité, plus trois boutons : **Modifier**, **Supprimer**, **Fermer**.

**Coordonnées** (affichées si renseignées) :

- courriel (icône `Mail`) ;
- téléphone formaté (icône `Phone`) ;
- adresse concaténée « adresse, ville, province, pays » (icône `MapPin`) ;
- site web (icône `Globe`) ;
- « Paiement : {conditions} » ;
- notes (dans un encadré) ;
- « Créé le {date} ».

**Contacts liés** (`:554-579`) : titre « Contacts ({nombre}) » ; pour chaque contact, un cercle avec ses initiales, ses prénom et nom, un badge **« Principal »** s'il s'agit du contact principal, puis une ligne « rôle - courriel ».

**Soumissions récentes** (`:582-605`, icône `FileText`) : jusqu'à 5 soumissions (`GET /devis?clientCompanyId={id}&perPage=5`). Chaque ligne montre le numéro de la soumission, le nom du projet et un badge de statut. S'il n'y en a aucune : « Aucune soumission ».

**Projets récents** (`:607-628`, icône `FolderKanban`) : jusqu'à 5 projets (`GET /projects?clientCompanyId={id}&perPage=5`). Chaque ligne montre le nom du projet et un badge de statut. S'il n'y en a aucun : « Aucun projet ».

> Ces deux listes sont filtrées par le **lien de clé étrangère** `clientCompanyId` : elles ne montrent que les documents réellement rattachés à cette entreprise, pas une recherche approximative par nom.

### 2.6 Modale Créer / Modifier

Les deux opérations partagent le même formulaire (`renderFormFields`, `CompaniesPage.tsx:395-470`). Les modales sont intitulées « Nouvelle entreprise » ou « Modifier l'entreprise ». En bas : **Annuler** et **Enregistrer** (ce dernier est désactivé tant que le nom est vide, et protégé contre le double-clic).

Champs, dans l'ordre :

1. **Nom de l'entreprise** * — requis.
2. **Type d'entreprise** * — menu déroulant, 14 valeurs (section 2.7).
3. **Secteur d'activité construction** — menu déroulant, 18 valeurs par défaut (section 2.7).
4. **Adresse** (sous-titre), **Adresse (rue, numéro)**, **Ville**.
5. **Province / État**, **Code postal**.
6. **Pays**.
7. **Site Web**.
8. **Courriel** (type courriel), **Téléphone**.
9. **Numéro de TPS**, **Numéro de TVQ**.
10. **Conditions de paiement**.
11. **Contact principal** — menu déroulant : « Aucun » plus tous les contacts du tenant (jusqu'à 100), au format « prénom nom (entreprise) » (`CompaniesPage.tsx:219-224`).
12. **Notes sur l'entreprise** — zone de texte sur 3 lignes.

Une note rappelle « * Champs obligatoires ».

**Valeurs proposées par défaut à la création** (`:167-172,276`) : type = « Entrepreneur général », province = « Québec », pays = « Canada ». Côté serveur, les conditions de paiement valent « Net 30 » si vous ne les changez pas (`companies.py:160`).

Les messages d'erreur s'affichent **à l'intérieur de la modale** (au-dessus des boutons), pour rester visibles sans être masqués par l'arrière-plan (`:865,878`).

> **Correction par rapport aux anciennes versions.** Le courriel, le téléphone, les numéros de TPS et de TVQ, et les conditions de paiement **sont maintenant dans le formulaire** : il n'est plus nécessaire de passer par l'API pour les saisir.

### 2.7 Types d'entreprise et secteurs d'activité

**14 types d'entreprise** (`TYPE_ENTREPRISE_OPTIONS`, `CompaniesPage.tsx:65-80`) :

Entrepreneur général · Sous-traitant spécialisé · Promoteur immobilier · Fournisseur de matériaux · Consultant / Ingénieur · Architecte · Arpenteur-géomètre · Organisme de contrôle · Institution financière · Assureur · Client résidentiel · Client commercial · Client industriel · Municipalité.

**18 secteurs d'activité par défaut** (`SECTEUR_OPTIONS`, `CompaniesPage.tsx:83-103`) :

Construction résidentielle · Construction commerciale · Construction industrielle · Rénovation résidentielle · Rénovation commerciale · Excavation et terrassement · Fondations spécialisées · Charpenterie générale · Couverture et toiture · Plomberie et chauffage · Électricité du bâtiment · Isolation et étanchéité · Revêtements extérieurs · Finition intérieure · Aménagement paysager · Démolition · Location d'équipements · Transport construction.

> **Les secteurs sont configurables par tenant.** La liste du menu déroulant vient de votre configuration (`GET /config/supplier-categories`, `CompaniesPage.tsx:257-266` ; réglable dans **Configuration → Catégories de fournisseurs**, manuel 28). À défaut de configuration, ce sont les 18 secteurs ci-dessus. Si une fiche porte déjà un secteur qui ne figure pas dans la liste courante (valeur héritée), le menu **conserve** cette valeur pour ne pas l'écraser (`:406-409,419-423`).
>
> **Note technique.** Côté serveur, le type et le secteur sont des **chaînes de texte libres**, bornées seulement en longueur. Il n'y a pas de validation stricte contre les 14 types ou les 18 secteurs — ces listes n'existent que dans l'interface. En pratique, restez dans les menus déroulants pour que le filtre et les statistiques restent cohérents.

### 2.8 Assistant IA — Entreprises

Ouvert par le bouton « Assistant IA » de la barre de commandes ; modale « Assistant IA — Entreprises » (`CompaniesPage.tsx:888`, composant `EntreprisesAssistantTab.tsx`, backend `entreprises_ai.py`). C'est un assistant **avec confirmation humaine** : il **propose**, vous **confirmez** (`entreprises_ai.py:13-19`).

**Deux capacités :**

1. **Lire vos données** (outil `recherche_bd`). L'assistant peut interroger, en lecture seule, un ensemble restreint de tables : `companies`, `contacts`, `projects`, `opportunities`, `devis`, `factures`. Les données sensibles (paie, RH, employés, comptes d'utilisateurs, Stripe, courriels…) sont **refusées** par une protection dédiée (`_ENT_SENSITIVE_RE`, `entreprises_ai.py:79-86`).
2. **Créer sur confirmation** (outils `proposer_entreprise` et `proposer_contact`). L'assistant **n'écrit jamais directement**. Il affiche une **carte de proposition** (avec un aperçu des champs). Vous cliquez sur **Confirmer** ou **Annuler**. Seule la confirmation (`POST /entreprises/ai/confirm-action`) exécute la création, après une nouvelle validation côté serveur.

**Déroulement du clavardage** (`EntreprisesAssistantTab.tsx`) : un en-tête « Assistant IA — Entreprises » et un sous-titre ; un état d'accueil avec 3 exemples de questions :

- « Combien ai-je de fournisseurs à Québec ? »
- « Crée l'entreprise "Construction ABC", client, à Montréal. »
- « Ajoute un contact Jean Tremblay, directeur, pour cette entreprise. »

Vous tapez votre demande (Entrée pour envoyer, Maj+Entrée pour une nouvelle ligne). Les bulles de réponse affichent des métadonnées (profil, jetons, coût, temps). Sous chaque proposition, deux boutons **Confirmer** et **Annuler**, avec la mention « En attente de confirmation ». Des verrous empêchent le double envoi et la double confirmation. Après une confirmation réussie, la liste et les statistiques de la page se rafraîchissent automatiquement (`:889`).

**Limites de l'assistant :**

- **Création seulement** (entreprise et contact). La **modification** d'une fiche existante par l'IA **n'est pas implémentée** (`entreprises_ai.py:21-23`).
- Pour rattacher un contact à une entreprise, l'assistant doit d'abord retrouver l'identifiant de l'entreprise via `recherche_bd`.
- Le vocabulaire de type suggéré par l'IA peut différer des libellés du menu déroulant (par exemple « Fournisseur » ou « Client » au lieu de « Fournisseur de matériaux » ou « Client résidentiel »). Vérifiez et ajustez le type après création si besoin.

**Un assistant distinct existe pour la page Contacts** (`/contacts`, backend `contacts_ai.py`) ; il est indépendant de l'assistant Entreprises.

---

## 3. Processus pas à pas

### 3.1 Créer une entreprise

1. Menu latéral → **Entreprises** → bouton **« Nouvelle entreprise »**.
2. Saisir au minimum le **Nom** (obligatoire). Choisir le **Type** (par défaut « Entrepreneur général ») selon la nature de la relation.
3. Compléter, au besoin : secteur d'activité, adresse, ville, province (« Québec » par défaut), code postal, pays (« Canada » par défaut), site web, courriel, téléphone, numéros de TPS et de TVQ, conditions de paiement, contact principal, notes.
4. Cliquer sur **Enregistrer** → `POST /companies` → message « Entreprise créée » → la liste et les statistiques se rechargent.

> Aucun numéro n'est généré automatiquement (contrairement aux soumissions `DEV-` ou aux dossiers `DOS-`). L'entreprise est immédiatement utilisable comme client dans les soumissions et les projets.

### 3.2 Consulter et modifier une entreprise

1. Cliquer sur la ligne de l'entreprise dans la liste → le **panneau de détail** s'ouvre (coordonnées, contacts, soumissions et projets récents).
2. Cliquer sur **Modifier** (crayon) → la modale s'ouvre, préremplie.
3. Ajuster les champs voulus.
4. Cliquer sur **Enregistrer** → `PUT /companies/{id}` → message « Entreprise mise à jour ».

> La mise à jour est partielle : seuls les champs modifiés sont envoyés. Un envoi sans aucun changement renvoie « Aucun champ à modifier ». Le nom ne peut pas être vidé.

### 3.3 Rechercher, filtrer et trier

1. Barre de commandes → **champ de recherche** : tapez un fragment de nom, de courriel ou de ville. La requête part 300 ms après la dernière frappe.
2. **Filtre par type** : choisissez un des 14 types ou « Tous les types ». Recherche et filtre se combinent.
3. **Tri** : cliquez sur un en-tête de colonne (Nom, Type, Contact/téléphone, Ville). Le tri s'applique **aux 20 lignes de la page affichée**, pas à la totalité du tenant.
4. Naviguez avec la pagination (20 par page).

### 3.4 Désigner un contact principal

Le contact principal est un lien qui part **de l'entreprise vers un contact**.

1. Créez d'abord le contact (page Contacts, manuel 05), ou via l'assistant IA.
2. Ouvrez l'entreprise → **Modifier** → menu déroulant **« Contact principal »** → sélectionnez la personne.
3. **Enregistrez.** Dans le panneau de détail, ce contact portera le badge **« Principal »**.

> **Un seul contact principal par entreprise** est garanti par le serveur : désigner un nouveau principal retire automatiquement le drapeau de l'ancien (`companies.py:794-799, 888-895`).

### 3.5 Désactiver une entreprise (suppression douce)

1. Panneau de détail ou tableau → bouton **Supprimer** (corbeille).
2. Fenêtre de confirmation : « Voulez-vous vraiment supprimer cette entreprise ? ». Confirmer.
3. Résultat : `DELETE /companies/{id}` effectue une **désactivation** (`active = FALSE` et `statut = 'Inactif'`, `companies.py:556-600`). Le message de succès affiché est **« Entreprise désactivée »**.

Conséquences :

- La fiche **n'est jamais effacée physiquement**. Toutes les soumissions, projets, factures et opportunités déjà rattachés restent intacts.
- L'entreprise **disparaît de la liste et des statistiques** par défaut : celles-ci excluent les entreprises désactivées (`COALESCE(active, TRUE) = TRUE`, `companies.py:125-138`).
- **Seul un administrateur** peut effectuer cette action (garde `require_tenant_admin_or_role()`). Un employé ordinaire ne verra pas l'opération aboutir.

> **Réactivation** : elle n'est **pas exposée dans l'interface**. Il n'y a ni bouton de réactivation, ni case « inclure les inactives » dans la liste. Pour réactiver une fiche désactivée, contactez votre administrateur (opération à effectuer via l'API ou l'accès direct à la base).

### 3.6 Consulter les soumissions et projets liés

1. Cliquez sur l'entreprise → le panneau de détail charge automatiquement les **5 dernières soumissions** et les **5 derniers projets** réellement rattachés (par clé étrangère).
2. Pour la liste complète, ouvrez les modules Soumissions (manuel 08) ou Projets (manuel 09) et filtrez par ce client.

### 3.7 Interroger vos données avec l'assistant IA

1. Barre de commandes → **« Assistant IA »**.
2. Posez une question en langage naturel, par exemple « Combien ai-je de fournisseurs à Québec ? » ou « Liste mes clients de Montréal ».
3. L'assistant interroge vos données (lecture seule, tables autorisées uniquement) et répond. Cette interrogation **consomme des crédits IA** (voir section 4.6).

### 3.8 Créer une entreprise ou un contact avec l'assistant IA

1. Ouvrez l'assistant IA.
2. Formulez la demande, par exemple « Crée l'entreprise "Construction ABC", client, à Montréal. » ou « Ajoute un contact Jean Tremblay, directeur, pour Construction ABC. ».
3. L'assistant affiche une **carte de proposition** avec l'aperçu des champs. **Rien n'est encore enregistré.**
4. Vérifiez, puis cliquez sur **Confirmer** (ou **Annuler**).
5. À la confirmation, l'entité est créée et la page se rafraîchit.

> La confirmation ne rappelle pas le modèle d'IA : elle exécute une écriture en base pure. Elle **ne consomme pas de crédit** supplémentaire (seul le clavardage en consomme, voir section 4.6).

### 3.9 Configurer la liste des secteurs

Les secteurs proposés dans le formulaire viennent de **Configuration → Catégories de fournisseurs** (manuel 28). Ajoutez-y vos propres secteurs pour qu'ils apparaissent dans le menu déroulant du formulaire d'entreprise. À défaut, les 18 secteurs par défaut s'appliquent.

---

## 4. Référence

### 4.1 Endpoints de l'API

Tous préfixés par `/api/erp/v1`. Sauf mention, la garde est `get_current_user` (utilisateur authentifié, limité à son tenant).

| Méthode | Chemin | Rôle | Garde / notes |
|---|---|---|---|
| GET | `/companies` | Liste paginée + recherche + filtre de type | `list_companies` (`companies.py:234`) |
| GET | `/companies/stats` | Compteurs Total / Clients / Fournisseurs / Sous-traitants | `companies_stats` (`:320`) ; déclaré avant `/{id}` |
| GET | `/companies/{id}` | Une entreprise + son tableau `contacts[]` | `get_company` (`:372`) ; 404 si introuvable |
| POST | `/companies` | Créer une entreprise | `create_company` (`:428`) |
| PUT | `/companies/{id}` | Mise à jour partielle | `update_company` (`:490`) ; 404 si aucune ligne |
| DELETE | `/companies/{id}` | **Désactivation** (soft-delete) | `delete_company` (`:557`) — **`require_tenant_admin_or_role()`** |
| GET | `/contacts` | Liste des contacts (filtre `company_id`, recherche) | `list_contacts` (`:621`) |
| GET | `/contacts/stats` | Compteurs de contacts | `contacts_stats` (`:707`) |
| POST | `/contacts` | Créer un contact | `create_contact` (`:753`) |
| PUT | `/contacts/{id}` | Mettre à jour un contact | `update_contact` (`:829`) |
| DELETE | `/contacts/{id}` | **Supprimer définitivement** un contact | `delete_contact` (`:934`) — **`require_tenant_admin_or_role()`** ; 409 si référencé |
| POST | `/entreprises/ai/chat` | Assistant IA : lecture + propositions | `entreprises_ai_chat` (`:356`) ; **débite des crédits** |
| POST | `/entreprises/ai/confirm-action` | Assistant IA : exécute la création confirmée | `confirm_entreprises_action` (`:506`) ; **ne débite rien** |

> **Asymétrie à connaître** : supprimer une **entreprise** est une désactivation réversible (`active = FALSE`), tandis que supprimer un **contact** est une **suppression définitive** (avec blocage 409 si le contact est encore lié à un projet, un contrat ou un document).

### 4.2 Champs et validations du formulaire d'entreprise

| Champ (formulaire) | Colonne | Obligatoire | Longueur max | Valeur par défaut |
|---|---|---|---|---|
| Nom de l'entreprise | `nom` | Oui | 200 | — |
| Type d'entreprise | `type_company` | Oui (menu) | 100 | « Entrepreneur général » |
| Secteur d'activité | `secteur_activite` | Non | 150 | — |
| Adresse (rue, numéro) | `adresse` | Non | 300 | — |
| Ville | `ville` | Non | 120 | — |
| Province / État | `province` | Non | 80 | « Québec » |
| Code postal | `code_postal` | Non | 20 | — |
| Pays | `pays` | Non | 80 | « Canada » |
| Site Web | `site_web` | Non | 300 | — |
| Courriel | `email` | Non | 255 | — |
| Téléphone | `telephone` | Non | 50 | — |
| Numéro de TPS | `numero_tps` | Non | 50 | — |
| Numéro de TVQ | `numero_tvq` | Non | 50 | — |
| Conditions de paiement | `payment_terms` | Non | 100 | « Net 30 » |
| Contact principal | `contact_principal_id` | Non | — | « Aucun » |
| Notes | `notes` | Non | 10000 | — |

- Le **nom** est validé côté serveur : un nom vide ou composé uniquement d'espaces est refusé (HTTP 422).
- Le **type** et le **secteur** ne sont pas validés contre une liste fermée (texte libre borné en longueur).
- Un **contact principal** inexistant est refusé proprement (HTTP 400), sans erreur technique.

### 4.3 Les 14 types d'entreprise

| # | Type | Usage typique |
|---|---|---|
| 1 | Entrepreneur général | Valeur par défaut ; entreprise qui orchestre des sous-traitants |
| 2 | Sous-traitant spécialisé | Couvreur, plombier, électricien… |
| 3 | Promoteur immobilier | Développeur de projets (condos, locatifs) |
| 4 | Fournisseur de matériaux | Quincaillerie, béton, bois, acier |
| 5 | Consultant / Ingénieur | Ingénierie civile, structure, mécanique |
| 6 | Architecte | Cabinet d'architecture |
| 7 | Arpenteur-géomètre | Arpentage, certificat de localisation |
| 8 | Organisme de contrôle | RBQ, CNESST, contrôle qualité |
| 9 | Institution financière | Banque, caisse, courtier hypothécaire |
| 10 | Assureur | Assurance chantier, responsabilité civile |
| 11 | Client résidentiel | Particulier propriétaire |
| 12 | Client commercial | Entreprise ou commerce |
| 13 | Client industriel | Usine, parc industriel |
| 14 | Municipalité | Ville, canton, MRC (donneur d'ouvrage public) |

> Seuls les types contenant « Client », « Fournisseur » ou « Sous-traitant » alimentent les sous-cartes de statistiques (voir 2.2).

### 4.4 Les 18 secteurs d'activité par défaut

Construction résidentielle · Construction commerciale · Construction industrielle · Rénovation résidentielle · Rénovation commerciale · Excavation et terrassement · Fondations spécialisées · Charpenterie générale · Couverture et toiture · Plomberie et chauffage · Électricité du bâtiment · Isolation et étanchéité · Revêtements extérieurs · Finition intérieure · Aménagement paysager · Démolition · Location d'équipements · Transport construction.

Liste enrichissable par tenant dans Configuration (section 3.9).

### 4.5 Calcul des statistiques

| Compteur | Formule (`companies.py:342-348`) |
|---|---|
| Total | `COUNT(*)` des entreprises non désactivées |
| Clients | `COUNT(*) FILTER (WHERE type_company ILIKE '%Client%')` |
| Fournisseurs | `COUNT(*) FILTER (WHERE type_company ILIKE '%Fournisseur%')` |
| Sous-traitants | `COUNT(*) FILTER (WHERE type_company ILIKE '%Sous-traitant%')` |

Conséquence : `Total ≠ Clients + Fournisseurs + Sous-traitants` dès qu'il existe des types hors de ces trois familles (Entrepreneur général, Architecte, Municipalité, Assureur, etc.).

### 4.6 Assistant IA — bornes et facturation

| Paramètre | Valeur |
|---|---|
| Modèle | `claude-sonnet-4-6` |
| Jetons de sortie max | 8000 |
| Itérations d'outils par tour | 5 au maximum |
| Historique conservé | 12 tours (corps borné à 40 messages) |
| Tables lisibles | `companies`, `contacts`, `projects`, `opportunities`, `devis`, `factures` |
| Limite de débit (par IP) | Clavardage : 20 ; confirmation : 30 |
| Coût du clavardage | `(jetons_entrée × 0,003 + jetons_sortie × 0,015) / 1000 × 1,30` (majoration de 30 %), en USD |
| Crédits | Débit sur `public.ai_prepaid_credits` ; **402** si le solde est épuisé |
| Confirmation d'action | Ne rappelle pas le modèle : **aucun crédit débité** |

> **Point d'attention (facturation).** Le clavardage débite les crédits **sans clé d'idempotence** dédiée (`entreprises_ai.py:482`) : un renvoi accidentel ou un double-clic peut débiter deux fois. La seule protection est la limite de débit par IP. Évitez de renvoyer une même question en rafale.

### 4.7 Gardes et mode consultation

- **Lecture, création, modification** : tout utilisateur authentifié du tenant.
- **Suppression (entreprise et contact)** : administrateur du tenant ou rôle équivalent (`require_tenant_admin_or_role()`).
- **Contexte de tenant absent** → HTTP 400 « Contexte tenant manquant ».
- **Mode consultation** (abonnement en lecture seule) → toute écriture renvoie **403 « Mode consultation »** ; entreprise du tenant désactivée → **401** (session coupée).
- **Aucune limite de débit dédiée** sur les endpoints CRUD `/companies` et `/contacts` (seuls les deux endpoints de l'IA en ont).

### 4.8 Limites et ce qui n'existe pas

| Sujet | État |
|---|---|
| Export CSV / PDF, impression | Absent de ce module |
| Téléversement de fichiers / logo / documents | Absent |
| Suppression définitive d'une entreprise | Impossible depuis l'interface (désactivation seule) |
| Réactivation d'une entreprise | Non exposée dans l'interface |
| Tri global côté serveur | Non : tri côté client sur la page affichée (20 lignes) |
| Modification / suppression par l'IA | Non : l'assistant ne fait que créer |
| Accès de l'IA à la paie / RH / employés | Refusé (protection anti-exfiltration) |
| Détection de doublons | Absente |
| Validation de format (taxes, code postal, téléphone) | Absente (texte libre) |
| Édition du statut hors désactivation | Aucun contrôle dédié dans le formulaire |
| Hiérarchie société mère / filiale | Absente |

### 4.9 Champs présents en base mais masqués de l'interface

La fiche `companies` porte, en base, des colonnes volontairement **non affichées et non modifiables** ici (liste blanche `_PUBLIC_COMPANY_COLS`, `companies.py:82-92`) :

- `mot_de_passe_hash` — mot de passe de connexion des clients du **portail B2B / C2B** (la table `companies` sert aussi d'identité au portail) ;
- `qbo_*` — secrets et état de synchronisation **QuickBooks** ;
- `credit_limit`, `ca_total` — données commerciales internes ;
- `tax_number_*`, `representant_code`, `source_acquisition` — champs internes.

Toute nouvelle colonne ajoutée à cette table est **exclue par défaut** de l'interface tant qu'elle n'est pas ajoutée à la liste blanche (comportement sûr face à l'évolution du schéma).

---

## 5. Intégrations et FAQ

### 5.1 Liens vers les autres modules

| Module lié | Lien technique | Ce que ça signifie |
|---|---|---|
| **Contacts** (manuel 05) | `contacts.company_id` ↔ `companies.contact_principal_id` | Une entreprise a N contacts ; elle en désigne un principal |
| **CRM et opportunités** (manuel 06) | `opportunities.company_id` | Le pipeline commercial s'appuie sur l'entreprise comme client |
| **Soumissions** (manuel 08) | `devis.client_company_id` | Les 5 dernières soumissions apparaissent dans le panneau de détail |
| **Projets** (manuel 09) | `projects.client_company_id` | Les 5 derniers projets apparaissent dans le panneau de détail |
| **Comptabilité et factures** (manuel 15) | `factures.client_company_id` | La facturation cible l'entreprise cliente |
| **Bons de commande** (manuel 14) | fournisseur = entreprise | Un fournisseur est une entreprise de ce référentiel |
| **Portail B2B / C2B** | `companies.mot_de_passe_hash` | La même table sert d'identité de connexion au portail |
| **QuickBooks** (Configuration, manuel 28) | colonnes `qbo_*`, colonne canonique `active` | La synchronisation lit `active` (d'où l'importance du soft-delete) |

### 5.2 Foire aux questions

**Pourquoi mon Total ne correspond-il pas à la somme Clients + Fournisseurs + Sous-traitants ?**
C'est normal. Les trois sous-cartes ne couvrent que les types contenant « Client », « Fournisseur » ou « Sous-traitant ». Un « Entrepreneur général », un « Architecte » ou une « Municipalité » comptent dans le Total mais dans aucune sous-carte (section 2.2).

**La colonne « Contact » affiche un numéro de téléphone, pas un nom. Est-ce un bug ?**
Non, c'est le comportement voulu : cette colonne montre le téléphone de l'entreprise. Le libellé est simplement peu explicite. Le nom du contact principal, lui, apparaît dans le panneau de détail.

**J'ai « supprimé » une entreprise mais elle n'a pas vraiment disparu ?**
La suppression est une **désactivation** réversible : la fiche passe à `active = FALSE` / `statut = 'Inactif'` et sort de la liste et des statistiques, mais elle reste en base et tous ses documents liés sont préservés. Le message affiché est d'ailleurs « Entreprise désactivée ».

**Comment réactiver une entreprise désactivée ?**
L'interface ne propose pas cette action. Adressez-vous à votre administrateur ; la réactivation se fait au niveau de l'API ou de la base.

**Je ne suis pas administrateur et le bouton Supprimer ne fait rien ?**
La désactivation exige un rôle d'administrateur. Un employé ordinaire peut créer et modifier une fiche, mais pas la désactiver.

**Puis-je exporter la liste des entreprises en CSV ou en PDF ?**
Pas depuis ce module : aucun export ni impression n'y est prévu.

**Puis-je joindre un logo ou des documents à une fiche ?**
Non, aucun téléversement n'est possible sur une fiche d'entreprise.

**Où mettre le NEQ (numéro d'entreprise du Québec) ou un numéro de licence RBQ ?**
Il n'y a pas de champ dédié dans ce module. Consignez-les dans les **Notes**. Les attestations RBQ / CCQ relèvent du module Conformité (manuel 17).

**Une même organisation peut-elle être à la fois cliente et fournisseur ?**
Oui. Une seule fiche `companies` peut être référencée comme cliente (factures, projets) et comme fournisseur (bons de commande). Choisissez le type le plus représentatif et précisez l'autre rôle dans les Notes.

**L'assistant IA peut-il modifier ou supprimer une fiche ?**
Non. Il ne fait que **créer** (entreprise ou contact), et toujours après votre confirmation. Il n'accède pas non plus aux données de paie, RH ou employés.

**Le tri des colonnes couvre-t-il toutes mes entreprises ?**
Non : il ne trie que les 20 lignes de la page affichée. Pour un tri global, affinez d'abord avec la recherche et le filtre de type.

**La recherche porte sur quels champs ?**
Sur le nom, le courriel et la ville uniquement. Elle ne cherche pas dans le téléphone, le code postal, les notes ni les numéros de taxe.

**Puis-je créer deux entreprises de même nom ?**
Oui, aucune contrainte d'unicité ni détection de doublon. Recherchez avant de créer pour éviter les doublons.

**Renommer une entreprise met-il à jour les soumissions et factures déjà émises ?**
Les documents déjà émis peuvent conserver le nom au moment de leur création. Le lien de rattachement (clé étrangère) reste correct, mais rééditez le document si son affichage doit refléter le nouveau nom.

---

## 6. Récapitulatif

- **Mission** : référentiel central des entreprises tierces (clients, fournisseurs, sous-traitants, consultants, organismes) du tenant.
- **Accès** : menu latéral, groupe GESTION → **Entreprises** (icône `Building2`), adresse `/entreprises`.
- **Statistiques** : 4 cartes **globales** (Total, Clients, Fournisseurs, Sous-traitants) via `GET /companies/stats` ; les 3 sous-cartes ne couvrent pas tout le Total (catégorisation par sous-chaîne du type).
- **Liste** : tableau triable (page courante), pagination 20 par page ; la colonne « Contact » affiche le **téléphone**.
- **Panneau de détail** : coordonnées, contacts liés, 5 dernières soumissions et 5 derniers projets **rattachés par clé étrangère** (fiable).
- **Formulaire** : nom obligatoire ; courriel, téléphone, TPS, TVQ et conditions de paiement **sont dans le formulaire** ; défauts « Entrepreneur général » / « Québec » / « Canada » / « Net 30 ».
- **14 types** d'entreprise et **18 secteurs** par défaut (secteurs **configurables par tenant**).
- **Suppression = désactivation** (`active = FALSE`, message « Entreprise désactivée »), **réservée aux administrateurs** ; les fiches désactivées sont **exclues** de la liste et des statistiques ; pas de suppression définitive ni de réactivation dans l'interface.
- **Assistant IA** : lit vos données (tables restreintes) et **crée** entreprises et contacts **sur confirmation** ; il ne modifie ni ne supprime rien. Le clavardage consomme des crédits IA ; la confirmation, non.
- **Contacts** est un **module séparé** (manuel 05), servi par le même backend et fortement lié à Entreprises.
- **Sécurité** : isolation stricte par tenant ; mode consultation en lecture seule sur abonnement inactif ; données sensibles (mot de passe B2B, secrets QuickBooks, données commerciales internes) masquées de l'interface.
- **Absences à connaître** : pas d'export / impression, pas de téléversement, pas de détection de doublon, pas de validation de format, pas de hiérarchie, pas de tri serveur global.

---

**Fichiers sources vérifiés** :
- `ERP_REACT/backend/routers/companies.py` (CRUD entreprises et contacts, soft-delete, statistiques, gardes)
- `ERP_REACT/backend/routers/entreprises_ai.py` (assistant IA, patron proposer / confirmer, facturation des crédits)
- `ERP_REACT/frontend/src/pages/CompaniesPage.tsx` (page `/entreprises`)
- `ERP_REACT/frontend/src/components/entreprises/EntreprisesAssistantTab.tsx` (modale de l'assistant)
- `ERP_REACT/frontend/src/api/companies.ts` et `ERP_REACT/frontend/src/api/entreprisesAi.ts` (clients d'API)
- `ERP_REACT/frontend/src/i18n/locales/{fr,en}/crm.json` (clés `entreprises.*`, `contacts.*`, `stages.*`)

**Manuels liés** :
- Manuel 05 — Contacts (personnes physiques) : `05-gestion-contacts.md`
- Manuel 06 — CRM et opportunités : `06-gestion-crm-opportunites.md`
- Manuel 08 — Soumissions : `08-ventes-soumissions.md`
- Manuel 09 — Projets : `09-ventes-projets.md`
- Manuel 14 — Bons de commande (fournisseurs) : `14-operations-bons-de-commande.md`
- Manuel 15 — Comptabilité et factures : `15-operations-comptabilite.md`
- Manuel 17 — Conformité (RBQ / CCQ / CNESST) : `17-terrain-conformite.md`
- Manuel 25 — Assistant IA (vue d'ensemble) : `25-communication-assistant-ia.md`
- Manuel 28 — Configuration (catégories de fournisseurs, QuickBooks) : `28-configuration.md`
