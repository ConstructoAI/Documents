# Module 33 — Aide et ressources

> **Version** : 3.0 (refonte vérifiée contre le code source, juillet 2026)
> **Code de référence** : `frontend/src/components/layout/Sidebar.tsx:244-297` (bloc « Aide & Ressources » de la barre latérale — trait de séparation 245, en-tête repliable 246-257, tableau des **4 liens** 259-262, rendu de chaque lien 263-297) ; `frontend/src/components/layout/TopBar.tsx:312-320` (bouton **Assistant IA**, seul point d'aide *in-app* de la barre supérieure) ; `Sidebar.tsx:382-389` (pied du tiroir **mobile** : courriel + téléphone + version) ; i18n `frontend/src/i18n/locales/fr/nav.json:41-47` et `en/nav.json:41-47`
> **Tables PostgreSQL** : **aucune**. **Aucun** point d'entrée FastAPI, **aucune** route interne React (`/aide` n'existe pas), **aucune** garde de rôle, **aucun** effet d'argent, **aucun** appel réseau vers le serveur de l'ERP.
> **Cadrage** : « Aide et ressources » n'est **pas un module fonctionnel**. C'est un **bloc statique de 4 liens** rendu au bas de la barre latérale gauche, sous tous les groupes de navigation et au-dessus de la section Super-Admin. Trois de ces liens **sortent de l'application** (chaîne YouTube, deux fichiers Markdown sur GitHub) et un quatrième déclenche le **composeur téléphonique** du support. Toute la logique vit côté client ; c'est pourquoi ce manuel est légitimement plus court que ceux des vrais modules. Pour l'aide **assistée par IA à l'intérieur** de l'ERP, ce n'est pas cette section : c'est le bouton **Assistant IA** de la barre supérieure (module distinct, voir `24-communication-assistant-ia.md`).

---

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Interface](#2-interface)
3. [Workflows pas à pas](#3-workflows-pas-à-pas)
4. [Référence](#4-référence)
5. [Intégrations et FAQ](#5-intégrations-et-faq)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Vue d'ensemble

### 1.1 Mission de la section

Offrir, depuis n'importe quel écran de l'ERP, un **point d'entrée unique et toujours visible** vers l'aide et la documentation, sans rien construire à l'intérieur de l'application :

- **Vidéos** — la chaîne YouTube officielle de Constructo AI (tutoriels, démonstrations, nouveautés).
- **Manuel** — l'index de la documentation utilisateur (ce dépôt), hébergé publiquement sur GitHub.
- **Téléphone** — le numéro du support, qui ouvre directement le composeur de l'appareil.
- **Liens utiles** — une page de références sélectionnées pour l'entrepreneur au Québec (RBQ, CCQ, CNESST, Revenu Québec, etc.).

C'est un **panneau d'orientation**, pas une base de connaissances intégrée. Il ne contient aucun texte d'aide en propre : il **pointe** vers du contenu qui vit ailleurs (YouTube, GitHub, la ligne téléphonique du support).

### 1.2 Ce que la section fait — et ne fait pas

| La section **fait** | La section **ne fait pas** |
|---|---|
| Afficher 4 raccourcis d'aide au bas de la barre latérale | Héberger une page d'aide *dans* l'ERP (aucune route `/aide`) |
| Ouvrir la chaîne vidéo et la documentation dans un nouvel onglet | Chercher dans la documentation (aucun moteur de recherche d'aide) |
| Déclencher un appel vers le support (lien `tel:`) | Ouvrir un formulaire de contact ou un système de billets |
| Rester visible pour **tout** utilisateur connecté | Filtrer par rôle (un employé voit les mêmes liens qu'un admin) |
| Se replier / se déployer d'un clic sur son en-tête | Se configurer par l'interface (les URL sont codées en dur) |
| — | Afficher des bulles d'aide contextuelles, un tour guidé ou un bouton « ? » |
| — | Suivre la progression de lecture, mémoriser des favoris, ou tracer les clics |

### 1.3 Un point important : ce n'est pas un « module » au sens habituel

Contrairement aux autres chapitres de ce manuel, il n'y a **rien à décrire côté serveur** :

- **0 point d'entrée FastAPI** — aucun routeur `aide`, `help`, `support`, `contact` ou `feedback` dans `ERP_REACT/backend/`.
- **0 table ou colonne PostgreSQL** touchée par la section.
- **0 route interne React** — les liens sont des balises `<a href>`, **pas** des liens de navigation interne (`NavLink`).
- **0 garde de permission** propre à la section (ni `is_admin`, ni `super_admin`, ni mode consultation).
- **0 intégration payante** — aucun Stripe, aucun crédit IA, aucune facturation déclenchée par ces 4 liens.

Le seul durcissement présent est côté client : l'attribut `rel="noopener noreferrer"` sur les liens externes (protection contre le détournement d'onglet, *tabnabbing*).

### 1.4 Accès

- **Emplacement** : barre latérale gauche, tout en bas, après le dernier groupe de modules (voir `Sidebar.tsx:244`).
- **En-tête** : le libellé « **AIDE & RESSOURCES** » (déjà en majuscules dans la traduction, `fr/nav.json:41`).
- **Toujours disponible** : la barre latérale n'apparaît que dans le shell protégé (après ouverture de session), donc la section est présente sur **toutes** les pages de l'ERP.
- **Aucune adresse interne** : chaque clic ouvre un nouvel onglet vers un site externe (YouTube ou GitHub) ou lance le composeur téléphonique — l'ERP lui-même reste ouvert derrière.

### 1.5 Permissions

**Aucune.** Les 4 liens sont rendus **hors** de la boucle de filtrage par rôle (`canSeeItem`, `Sidebar.tsx:144-152`), qui ne s'applique qu'aux groupes de navigation (`NAV_GROUPS`). Résultat : **tout utilisateur authentifié** — administrateur, propriétaire, comptable, employé sans privilège, super-administrateur — voit exactement les mêmes 4 ressources. Les cibles (YouTube, GitHub) sont publiques.

---

## 2. Interface

Toute l'interface de la section tient dans le bas de la barre latérale (`Sidebar.tsx:244-297`), avec un comportement qui varie selon le mode d'affichage.

### 2.1 L'en-tête « AIDE & RESSOURCES »

Source : `Sidebar.tsx:245-257`.

- Un **trait de séparation** horizontal (`border-t border-white/10`) marque le début du bloc, juste au-dessus du titre (`Sidebar.tsx:245`).
- Le titre est un **bouton repliable** (`<button>`) : cliquer dessus replie ou déploie la liste des 4 liens (`toggleGroup('nav.aideRessources')`, `Sidebar.tsx:247-248`).
- Style : petit texte gris clair en majuscules avec interlettrage (`text-[11px] font-semibold uppercase tracking-wider text-white/60`, `Sidebar.tsx:249`) — cohérent avec le thème marine foncé de la barre latérale (`erp-sidebar`).
- À droite du titre, un **chevron** (`ChevronDown`, taille 12) indique l'état : il pivote d'un quart de tour (`-rotate-90`) quand la section est repliée (`Sidebar.tsx:252-255`).
- **État par défaut = déployé** : au premier affichage, la clé `nav.aideRessources` est indéfinie dans l'état local, ce qui donne « non replié » (`?? false`, `Sidebar.tsx:254, 258`). Les 4 liens sont donc visibles d'emblée.
- Cet état de repli est une **préférence d'affichage de session** (état React local, `Sidebar.tsx:120`) : il tient tant que l'application reste ouverte, mais **revient à « déployé » après un rechargement complet** de la page (il n'est pas mémorisé sur le poste).

### 2.2 Les 4 liens

Source : `Sidebar.tsx:259-262`. Chaque entrée est une balise `<a href>` (donc une **sortie** de l'application, pas une navigation interne).

| Ordre | Libellé (FR) | Clé i18n | Destination | Type | Icône | Élément de fin |
|---|---|---|---|---|---|---|
| 1 | **Vidéos** | `nav.helpResources.videos` | `https://www.youtube.com/channel/UC3EGXYQNj5UYGiyNfiiom_A` | Externe (nouvel onglet) | `Video` (18) | Glyphe « lien externe » (`ExternalLink`, 10, atténué) |
| 2 | **Manuel** | `nav.helpResources.manuel` | `https://github.com/ConstructoAI/Documents/blob/main/README.md` | Externe (nouvel onglet) | `BookOpen` (18) | Glyphe « lien externe » |
| 3 | **Téléphone** | `nav.helpResources.telephone` | `tel:+19365871141` | **Non externe** (composeur) | `Phone` (18) | **Texte** `1 936 587-1141` |
| 4 | **Liens utiles** | `nav.helpResources.usefulLinks` | `https://github.com/ConstructoAI/Documents/blob/main/liens-utiles.md` | Externe (nouvel onglet) | `ExternalLink` (18) | Glyphe « lien externe » |

Règles de rendu (`Sidebar.tsx:263-295`) :

- **Libellé** : provient de la traduction, `t(link.labelKey)` (`Sidebar.tsx:264`) — il n'est plus codé en dur dans le composant.
- **Nouvel onglet** : seulement pour les liens externes. `target="_blank"` et `rel="noopener noreferrer"` sont posés **uniquement** si `external` est vrai (`Sidebar.tsx:269-270`). Le lien Téléphone (`external:false`) n'ouvre donc **aucun** onglet.
- **Élément de fin** (`Sidebar.tsx:288-292`) : si le lien est externe, un petit glyphe « lien externe » atténué s'affiche à droite ; sinon, s'il possède un texte de fin (`trailingText`), ce texte s'affiche (cas du Téléphone, qui montre le numéro `1 936 587-1141`) ; sinon, rien.
- Aucun de ces liens ne passe par le filtre de rôle : ils sont rendus **inconditionnellement** pour tout utilisateur connecté.

#### 2.2.1 Vidéos

Chaîne YouTube officielle de Constructo AI. On y trouve les tutoriels de démarrage, les démonstrations par module et les nouveautés. Le lien ouvre YouTube dans un nouvel onglet ; l'ERP reste ouvert derrière.

#### 2.2.2 Manuel

Pointe vers le **README.md** du dépôt public `ConstructoAI/Documents` — l'**index** de la documentation utilisateur (le même dépôt qui contient ce fichier). Ce README liste tous les manuels (Principal, Suivi, Gestion, Ventes, Opérations, Terrain, Communication, Outils, Configuration, Aide). C'est une ressource **légère** : le README est un sommaire, à partir duquel on ouvre le manuel du module voulu.

> Le contenu du manuel **quitte l'ERP** : il vit sur GitHub, exige une connexion Internet, et n'est pas rendu dans une visionneuse à l'intérieur de l'application.

#### 2.2.3 Téléphone

Lien `tel:+19365871141`. C'est le **seul lien non externe** de la section (`external:false`, `Sidebar.tsx:261`) : il n'ouvre pas d'onglet ; il demande au système d'exploitation de composer le numéro (utile surtout sur mobile ou avec un logiciel de téléphonie sur ordinateur). Le numéro s'affiche en texte à droite du libellé : **1 936 587-1141**.

#### 2.2.4 Liens utiles

Pointe vers **liens-utiles.md** du même dépôt GitHub. C'est une page de références **sélectionnées** pour l'entrepreneur au Québec, organisée par thèmes : réglementation et licences (RBQ, GCR), main-d'œuvre et conventions (CCQ), santé et sécurité (CNESST), fiscalité (Revenu Québec, ARC), achats publics et appels d'offres, subventions et financement, codes et normes (CNB, CSA), registres de probité, associations et ordres professionnels, lois de référence, veille sectorielle et calculateurs en ligne.

> Comme le Manuel, cette page **quitte l'ERP** vers GitHub. Son contenu évolue indépendamment de l'application (les URL y ont été validées à sa propre date de révision).

### 2.3 Comportement selon le mode d'affichage

Source : `Sidebar.tsx:246, 258, 271-280`.

| Mode | En-tête « AIDE & RESSOURCES » | Les 4 liens |
|---|---|---|
| **Bureau, barre déployée** (`lg:w-56`) | Visible et repliable | Icône + libellé + élément de fin |
| **Bureau, barre réduite** (`lg:w-14`) | **Masqué** (l'en-tête ne s'affiche qu'en mode déployé) | Rendus **en icônes seules** ; le libellé (et le numéro, pour le Téléphone) apparaît en **infobulle** au survol (attribut `title`, `Sidebar.tsx:271`) |
| **Mobile** (tiroir plein écran) | Visible et repliable | Icône + libellé, avec cibles tactiles agrandies (`min-h-[44px]`, `Sidebar.tsx:277`) |

Détails :

- **Barre réduite** : même repliée, la section montre ses 4 icônes (`Video`, `BookOpen`, `Phone`, `ExternalLink`), chacune avec une infobulle. C'est la seule façon de garder l'aide accessible quand la barre est en mode icônes.
- **Mobile** : le tiroir se referme automatiquement lors d'une navigation *interne*, mais ces liens externes ne changent pas de route (ils ouvrent un onglet ou le composeur), donc le tiroir se comporte comme attendu.

### 2.4 Le pied du tiroir mobile (coordonnées de support)

Source : `Sidebar.tsx:382-389`. **Uniquement en mode mobile.**

Au bas du tiroir mobile — et **nulle part sur bureau** — on trouve un pied qui affiche les vraies coordonnées de support :

- **Version** : « Constructo AI ERP AI v1.0 » (`Sidebar.tsx:383`).
- **Courriel** : `mailto:info@constructoai.ca`, affiché **info@constructoai.ca** (`Sidebar.tsx:385`). C'est le **seul endroit de la barre latérale** qui expose l'adresse de courriel en lien cliquable.
- **Téléphone** : `tel:+19365871141`, affiché **1 (936) 587-1141** (`Sidebar.tsx:387`).

> **À noter** : sur **bureau**, le pied de la barre latérale (`Sidebar.tsx:344-353`) ne contient **que** le bouton pour replier / déplier la barre — ni courriel, ni version. Le seul contact présent dans la barre latérale de bureau est donc le lien **Téléphone** de la section Aide.

### 2.5 Le bouton d'aide *in-app* : Assistant IA (barre supérieure)

Source : `TopBar.tsx:312-320`.

L'aide **assistée à l'intérieur** de l'ERP ne se trouve pas dans la barre latérale, mais dans la **barre supérieure** : un bouton en forme d'étincelles (`Sparkles`, taille 18) qui mène à l'Assistant IA (`navigate('/assistant-ia')`). Son libellé, son infobulle et son étiquette d'accessibilité valent tous « **Assistant IA** » (`t('nav.assistantIa')`).

- **Accès global** : ce bouton est présent sur toutes les pages, à gauche de la recherche. Sur mobile, c'est une icône de 44 × 44 ; le texte « Assistant IA » n'apparaît qu'à partir d'un très grand écran (point de rupture `xl`).
- **Point d'entrée unique** : l'Assistant IA n'est **pas** dans la barre latérale (absent de `NAV_GROUPS`). Ce bouton de la barre supérieure est le **seul** moyen visible d'ouvrir `/assistant-ia`.
- **Module distinct** : l'Assistant IA a son propre serveur (`routers/ai.py`) et sa propre facturation en crédits IA — il ne fait **pas** partie de la section Aide et ressources. Voir `24-communication-assistant-ia.md`.

> Il n'y a **aucun bouton « ? » ni « Aide »** dédié dans la barre supérieure. Le seul autre bouton d'apparence utilitaire est l'engrenage (`Settings`) qui mène à la Configuration (`TopBar.tsx:539-544`) — sans rapport avec l'aide.

---

## 3. Workflows pas à pas

Ces procédures découlent directement du comportement décrit à la section 2. Aucune n'entraîne d'appel au serveur de l'ERP.

### 3.1 Ouvrir les tutoriels vidéo

1. Dans la barre latérale gauche, descendre jusqu'à la section « AIDE & RESSOURCES ».
2. Cliquer sur **Vidéos**.
3. Un nouvel onglet s'ouvre sur la chaîne YouTube de Constructo AI. L'ERP reste ouvert dans l'onglet précédent.

### 3.2 Consulter le manuel utilisateur

1. Section « AIDE & RESSOURCES » → **Manuel**.
2. Un nouvel onglet s'ouvre sur le README de la documentation (GitHub).
3. Depuis le sommaire, ouvrir le manuel du module recherché. Astuce : utiliser la recherche du navigateur (`Ctrl+F` / `Cmd+F`) sur la page GitHub pour trouver un mot-clé.

> Une connexion Internet est requise (le contenu est hébergé sur GitHub, hors de l'ERP).

### 3.3 Appeler le support

1. Section « AIDE & RESSOURCES » → **Téléphone** (numéro affiché : 1 936 587-1141).
2. Sur un appareil doté de téléphonie (mobile, ou logiciel d'appel sur ordinateur), le composeur s'ouvre avec le numéro pré-rempli. Aucun nouvel onglet ne s'ouvre.
3. Sur un poste sans application de téléphonie, le clic peut ne rien déclencher : composer alors le numéro manuellement.

### 3.4 Écrire au support par courriel

Le courriel n'est **pas** l'un des 4 liens de la section ; il se trouve ailleurs dans l'application :

- **Sur mobile** : ouvrir le tiroir de navigation (menu), descendre jusqu'au pied ; toucher **info@constructoai.ca** (`Sidebar.tsx:385`). L'application de courriel s'ouvre.
- **Avant l'ouverture de session** : l'adresse `info@constructoai.ca` figure aussi sur les pages de connexion et d'inscription (`LoginPage.tsx:173`, `RegisterPage.tsx:113`, `B2bLoginPage.tsx:76`, `B2bRegisterPage.tsx:156`).
- **Sur bureau, après connexion** : l'adresse n'apparaît pas dans la barre latérale. La recopier depuis ce manuel ou depuis le pied mobile : **info@constructoai.ca**.

### 3.5 Ouvrir les liens utiles (organismes du Québec)

1. Section « AIDE & RESSOURCES » → **Liens utiles**.
2. Un nouvel onglet s'ouvre sur la page GitHub des références (RBQ, CCQ, CNESST, Revenu Québec, subventions, normes, etc.).
3. Cliquer le lien de l'organisme voulu, qui s'ouvre à son tour.

### 3.6 Replier ou déployer la section

1. Cliquer sur l'en-tête « AIDE & RESSOURCES ».
2. Le chevron pivote et la liste des 4 liens se masque (replié) ou réapparaît (déployé).
3. Ce réglage vaut pour la session en cours ; un rechargement complet de la page ramène la section à l'état « déployé ».

### 3.7 Retrouver l'aide quand la barre est en mode icônes

1. Si la barre latérale est réduite (icônes seulement), l'en-tête « AIDE & RESSOURCES » est masqué, mais les 4 icônes restent visibles.
2. Survoler une icône pour afficher son infobulle (par exemple « Téléphone 1 936 587-1141 »).
3. Cliquer l'icône voulue — le comportement (nouvel onglet ou composeur) est identique au mode déployé.

### 3.8 Obtenir de l'aide assistée dans l'ERP (Assistant IA)

1. Dans la **barre supérieure**, cliquer l'icône **étincelles** (Assistant IA), à gauche de la recherche.
2. La page `/assistant-ia` s'ouvre à l'intérieur de l'ERP.
3. Poser la question en langage naturel. L'Assistant peut consulter vos données (projets, factures, clients) et proposer des actions.

> C'est un **module distinct**, facturé en crédits IA. Ce n'est pas un canal officiel pour signaler un bogue : pour cela, utiliser le courriel ou le téléphone du support.

### 3.9 Choisir le bon canal selon le besoin

| Besoin | Canal recommandé |
|---|---|
| Apprendre à se servir d'un module | **Vidéos** ou **Manuel** |
| Trouver une référence officielle (RBQ, CCQ, CNESST, taxes) | **Liens utiles** |
| Question rapide sur mes données dans l'ERP | **Assistant IA** (barre supérieure) |
| Problème technique, bogue, facturation, intégration | **Courriel** info@constructoai.ca ou **Téléphone** 1 936 587-1141 |
| Découverte / questions avant de s'abonner | Chatbot public « Sylvain » sur la page de connexion (voir 5.2) |

---

## 4. Référence

### 4.1 Points d'entrée du serveur

**Aucun.** La section n'appelle aucun point d'entrée FastAPI. Il n'existe aucun routeur `aide`, `help`, `support`, `contact` ni `feedback` dans `ERP_REACT/backend/`. Chaque clic est traité **entièrement dans le navigateur** : ouverture d'un onglet externe ou déclenchement du composeur (`tel:`).

| Élément | État |
|---|---|
| Points d'entrée (méthode + chemin) | Aucun |
| Préfixe de montage | Sans objet |
| Tables / colonnes PostgreSQL | Aucune |
| Validations / bornes / quotas | Sans objet |
| Rôles requis / gardes (`is_admin`, `super_admin`, mode consultation) | Aucun — visible pour tout utilisateur connecté |
| Intégrations (Stripe, QuickBooks, Vapi, IA) | Aucune dans la section |
| Effets d'argent (facturation, crédits) | Aucun |
| Flux de travail / statuts / calculs | Aucun |

### 4.2 Les 4 liens (définition exacte)

Source : `Sidebar.tsx:259-262`.

| # | Libellé FR / EN | Clé i18n | `href` | `external` | Icône (lucide) | Fin |
|---|---|---|---|---|---|---|
| 1 | Vidéos / Videos | `nav.helpResources.videos` | `https://www.youtube.com/channel/UC3EGXYQNj5UYGiyNfiiom_A` | `true` | `Video` (18) | glyphe externe |
| 2 | Manuel / Manual | `nav.helpResources.manuel` | `https://github.com/ConstructoAI/Documents/blob/main/README.md` | `true` | `BookOpen` (18) | glyphe externe |
| 3 | Téléphone / Phone | `nav.helpResources.telephone` | `tel:+19365871141` | `false` | `Phone` (18) | texte `1 936 587-1141` |
| 4 | Liens utiles / Useful links | `nav.helpResources.usefulLinks` | `https://github.com/ConstructoAI/Documents/blob/main/liens-utiles.md` | `true` | `ExternalLink` (18) | glyphe externe |

### 4.3 Traductions (i18n)

Source : `fr/nav.json:41-47` et `en/nav.json:41-47`. Parité FR/EN complète (le titre + 4 libellés). Aucune chaîne côté serveur.

| Clé | Français | Anglais |
|---|---|---|
| `nav.aideRessources` | AIDE & RESSOURCES | HELP & RESOURCES |
| `nav.helpResources.videos` | Vidéos | Videos |
| `nav.helpResources.manuel` | Manuel | Manual |
| `nav.helpResources.telephone` | Téléphone | Phone |
| `nav.helpResources.usefulLinks` | Liens utiles | Useful links |

### 4.4 Comportement par mode d'affichage

| Aspect | Barre déployée (bureau) | Barre réduite (bureau) | Tiroir mobile |
|---|---|---|---|
| En-tête « AIDE & RESSOURCES » | Visible, repliable | Masqué | Visible, repliable |
| Liens | Icône + libellé + fin | Icône seule + infobulle | Icône + libellé (cible 44 px) |
| Pied avec courriel + version | Non | Non | **Oui** (`Sidebar.tsx:382-389`) |
| Largeur | `lg:w-56` | `lg:w-14` | `85vw` (max 280 px) |

### 4.5 Coordonnées et canaux de support

| Canal | Valeur | Où le trouver | Serveur ERP impliqué ? |
|---|---|---|---|
| Vidéos (YouTube) | Chaîne officielle | Lien 1 de la section | Non (externe) |
| Manuel (GitHub) | `ConstructoAI/Documents` README | Lien 2 de la section | Non (GitHub public) |
| Liens utiles (GitHub) | `ConstructoAI/Documents` liens-utiles.md | Lien 4 de la section | Non |
| Téléphone | **1 936 587-1141** (`tel:+19365871141`) | Lien 3 de la section + pied mobile | Non (composeur du système) |
| Courriel | **info@constructoai.ca** | Pied mobile + pages de connexion/inscription | Non (application de courriel) |
| Assistant IA (in-app) | `/assistant-ia` | Bouton étincelles de la barre supérieure | Oui — **module distinct** (`routers/ai.py`) |
| Chatbot « Sylvain » (avant connexion) | Page de connexion publique | `app.constructoai.ca` | Oui — **module distinct** (`routers/voice.py`) |

> **Format du numéro** : même numéro partout (`+19365871141`), mais présentation différente — le lien de la section montre `1 936 587-1141` (`Sidebar.tsx:261`), tandis que le pied mobile et le chatbot public affichent `1 (936) 587-1141` (`Sidebar.tsx:387`). Simple différence de présentation.

### 4.6 Configurabilité

| Réglage | Valeur | Modifiable par l'utilisateur ? |
|---|---|---|
| URL des 4 liens | Codées en dur (`Sidebar.tsx:259-262`) | Non |
| Numéro de téléphone | `+19365871141` (codé en dur) | Non |
| Adresse de courriel | `info@constructoai.ca` (codée en dur) | Non |
| Libellés | Traductions `nav.helpResources.*` | Non (nécessite une mise à jour des fichiers de langue) |
| Ordre / nombre de liens | 4, fixes | Non |

> Modifier un lien ou un libellé = éditer le code (`Sidebar.tsx` ou les fichiers `nav.json`), reconstruire l'interface, puis redéployer. **Aucune interface d'administration** ne permet de changer ces valeurs.

### 4.7 Sécurité

- **Anti-détournement d'onglet** : `rel="noopener noreferrer"` est posé sur les 3 liens externes (`Sidebar.tsx:270`) — la page cible ne peut pas accéder à la fenêtre d'origine.
- **Aucun suivi** : le clic ne déclenche aucun événement analytique côté ERP.
- **Aucun état d'erreur, de chargement ni de réessai** : les liens sont statiques. Le seul « échec » possible est côté navigateur (site externe indisponible, ou absence d'application de téléphonie / de courriel).

### 4.8 Ce qui n'existe pas (aide-mémoire)

| Fonction | Présente ? |
|---|---|
| Route interne `/aide` ou page d'aide dans l'ERP | Non |
| FAQ intégrée / base de connaissances recherchable | Non |
| Visionneuse du manuel dans l'application | Non (le manuel est sur GitHub) |
| Formulaire de contact / système de billets | Non |
| Clavardage en direct (*live chat*) dans le shell | Non |
| Bouton « ? » dans la barre supérieure | Non |
| Bulles d'aide contextuelles centralisées / tour guidé | Non |
| Lien « Courriel » parmi les 4 liens de la section | Non (le courriel est au pied mobile) |
| Courriel dans la barre latérale de **bureau** | Non (mobile seulement) |
| Filtrage des liens par rôle | Non (tous les voient) |

---

## 5. Intégrations et FAQ

### 5.1 Lien avec l'Assistant IA

L'Assistant IA est la seule aide **assistée à l'intérieur** de l'ERP, mais il ne fait pas partie de cette section : c'est un module autonome (serveur `routers/ai.py`), atteignable par le bouton étincelles de la barre supérieure (`TopBar.tsx:312-320`). Il consomme des **crédits IA** et peut consulter vos données. Détails : `24-communication-assistant-ia.md`.

### 5.2 Lien avec le chatbot public « Sylvain »

Avant l'ouverture de session, la page de connexion publique (`app.constructoai.ca`) offre un assistant conversationnel — « Sylvain » (invite système `backend/sylvain_prompt.py`, serveur `routers/voice.py`, préfixes `/voice` et `/api/voice`). Son rôle est d'orienter les visiteurs et de **rediriger les questions techniques ou d'intégration** vers le courriel **info@constructoai.ca** ou le téléphone **1 (936) 587-1141**, et vers les **vidéos** YouTube pour la découverte. Il **normalise** l'affichage du numéro (`+1 936-587-1141` → `19365871141`, `voice.py:356`). Ce chatbot est **distinct** de la section Aide et ressources et n'est pas accessible depuis le shell connecté.

### 5.3 Le dépôt GitHub `ConstructoAI/Documents`

Les liens **Manuel** et **Liens utiles** pointent tous deux vers le dépôt **public** `github.com/ConstructoAI/Documents` — le même dépôt qui héberge ce manuel. Conséquences pratiques :

- **Public** : toute personne disposant du lien peut lire ces pages, sans compte.
- **Externe** : le contenu vit hors de l'ERP ; il exige Internet et n'est pas vérifiable depuis le code de l'application.
- **Évolutif** : la documentation peut être mise à jour indépendamment des versions de l'ERP.
- Pour signaler une erreur dans la documentation : écrire à **info@constructoai.ca** (ou ouvrir un billet sur le dépôt).

### 5.4 Ce qui n'est PAS possible

- **Aucune page d'aide interne** : pas de route `/aide`, pas de FAQ intégrée, pas de recherche dans la documentation, pas de visionneuse du manuel dans l'application.
- **Aucun système de billets, formulaire de contact ou clavardage en direct** : le support humain se résume au **courriel** et au **téléphone**.
- **Aucun bouton « ? »**, aucun tour guidé, aucune bulle d'aide contextuelle centralisée dans cette section.
- **Aucune configuration par l'interface** : les 4 URL, le numéro et le courriel sont codés en dur ; les changer exige une modification du code et un redéploiement.
- **Aucun filtrage par rôle** : un employé sans privilège voit exactement les mêmes ressources qu'un administrateur.
- **Aucun lien « Courriel »** parmi les 4 liens ; sur bureau, le courriel n'apparaît nulle part dans la barre latérale (mobile et pages de connexion seulement).

### 5.5 FAQ

**« Aide et ressources » est-il un module comme les autres ?**
Non. C'est un bloc statique de 4 liens dans la barre latérale (`Sidebar.tsx:244-297`). Il n'a ni page interne, ni serveur, ni base de données.

**Pourquoi les liens ouvrent-ils un nouvel onglet ?**
Pour préserver l'état de l'ERP (formulaire en cours, navigation) pendant qu'on consulte la ressource externe. Le lien Téléphone, lui, ne fait qu'ouvrir le composeur (pas d'onglet).

**Où est le manuel : dans l'application ou en ligne ?**
En ligne. Le lien « Manuel » ouvre le README du dépôt GitHub `ConstructoAI/Documents`. Il n'y a pas de visionneuse intégrée ; une connexion Internet est nécessaire.

**Le lien « Manuel » semble plus court que les autres ressources. Est-ce normal ?**
Oui. Le lien pointe vers le **README** (l'index de la documentation), pas vers un module précis. À partir de cet index, on ouvre le manuel détaillé du module voulu.

**Comment joindre un humain au support ?**
Par **courriel** (info@constructoai.ca) ou par **téléphone** (1 936 587-1141). Il n'y a pas de clavardage en direct ni de système de billets.

**Je ne vois pas le courriel du support sur mon ordinateur.**
Sur bureau, le courriel n'est pas dans la barre latérale : il figure au **pied du tiroir mobile** et sur les **pages de connexion**. Adresse : info@constructoai.ca. Le seul contact présent dans la barre latérale de bureau est le lien **Téléphone**.

**Le numéro s'affiche différemment à deux endroits — est-ce le même ?**
Oui. `1 936 587-1141` (lien de la section) et `1 (936) 587-1141` (pied mobile) composent tous deux `+19365871141`. Simple différence de présentation.

**Un employé voit-il moins de ressources qu'un administrateur ?**
Non. Les 4 liens ne sont soumis à aucun filtrage de rôle : tout utilisateur connecté voit exactement la même chose.

**Puis-je modifier ces liens depuis l'interface d'administration ?**
Non. Ils sont codés en dur dans `Sidebar.tsx`. Les changer exige une modification du code, une reconstruction et un redéploiement.

**Y a-t-il une aide contextuelle (bulles) sur chaque champ ?**
Pas de système centralisé ici. Certains écrans ont des infobulles ponctuelles, mais il n'existe pas de bouton « ? » universel par champ.

**L'Assistant IA est-il le canal de support pour les bogues ?**
Non. L'Assistant IA (bouton étincelles de la barre supérieure) répond à des questions métier sur vos données, mais pour un bogue ou une question technique/facturation, passez par le courriel ou le téléphone.

**Où se trouve l'Assistant IA ? Je ne le vois pas dans le menu de gauche.**
Il n'est **pas** dans la barre latérale. Son seul point d'entrée est l'icône **étincelles** dans la barre supérieure, à gauche de la recherche.

**Cette section va-t-elle s'enrichir ?**
Toute évolution serait documentée dans une version future de ce manuel. En l'état, c'est un bloc de 4 liens statiques.

---

## 6. Récapitulatif

- **Cadrage** : « Aide et ressources » (`nav.aideRessources` = « AIDE & RESSOURCES ») n'est **pas un module fonctionnel** — c'est un **bloc statique de 4 liens** au bas de la barre latérale (`Sidebar.tsx:244-297`). Aucune route interne, aucun serveur, aucune base de données.
- **Aucun backend** : 0 point d'entrée FastAPI, 0 table PostgreSQL, 0 garde de rôle, 0 effet d'argent, 0 appel réseau vers l'ERP.
- **Les 4 liens** : **Vidéos** (YouTube), **Manuel** (README GitHub), **Téléphone** (`tel:+19365871141`, affiché 1 936 587-1141), **Liens utiles** (GitHub liens-utiles.md). Trois sont externes (`target="_blank"` + `rel="noopener noreferrer"`) ; le Téléphone ouvre le composeur (`external:false`).
- **Toujours visibles** : la section n'est pas filtrée par rôle — administrateur comme employé voient les mêmes ressources.
- **En-tête repliable**, déployé par défaut ; l'état de repli est une préférence de session, réinitialisée au rechargement complet.
- **Modes d'affichage** : barre déployée (icône + libellé) ; barre réduite (icônes + infobulles, en-tête masqué) ; mobile (cibles 44 px + **pied avec courriel info@constructoai.ca, téléphone 1 (936) 587-1141 et version v1.0**).
- **Courriel du support** : présent **au pied mobile** et sur les **pages de connexion** — **pas** dans la barre latérale de bureau, et **pas** parmi les 4 liens.
- **Aide assistée dans l'ERP** : le bouton **Assistant IA** (étincelles) de la barre supérieure (`TopBar.tsx:312-320`) — **module distinct**, seul point d'entrée vers `/assistant-ia`.
- **Chatbot public « Sylvain »** (avant connexion) oriente les visiteurs et redirige les questions techniques vers le courriel / le téléphone — hors de la section.
- **N'existe pas** : page d'aide interne, FAQ intégrée, recherche dans la documentation, formulaire de contact, système de billets, clavardage en direct, bouton « ? », tour guidé, configuration par l'interface.
- **Non configurable** : les 4 URL, le numéro et le courriel sont codés en dur ; les changer = éditer le code + reconstruire + redéployer.

---

**Sources vérifiées** : `frontend/src/components/layout/Sidebar.tsx` (bloc « Aide & Ressources » 244-297 : trait 245, en-tête repliable 246-257, tableau des 4 liens 259-262, rendu 263-297 ; filtre de rôle `canSeeItem` 144-152 dont la section est exclue ; pied mobile 382-389 ; pied bureau 344-353), `frontend/src/components/layout/TopBar.tsx:312-320` (bouton Assistant IA), `frontend/src/i18n/locales/fr/nav.json:41-47` et `en/nav.json:41-47` (traductions), `backend/` (aucun routeur `aide`/`help`/`support`/`contact`/`feedback` — vérifié), `backend/sylvain_prompt.py` + `backend/routers/voice.py` (chatbot public « Sylvain », préfixes `/voice` et `/api/voice`, normalisation du numéro `voice.py:356`), `frontend/src/pages/LoginPage.tsx:173`, `RegisterPage.tsx:113`, `B2bLoginPage.tsx:76`, `B2bRegisterPage.tsx:156` (courriel de support pré-connexion).

**Manuels liés** :
- Assistant IA (aide assistée in-app, crédits IA) — `24-communication-assistant-ia.md`
- Agent vocal (téléphonie, chatbot vocal) — `23-communication-agent-vocal.md`
- Configuration (abonnement, moyen de paiement, préférences) — `30-configuration.md`
- Web (recherche en direct + liens utiles Québec) — `29-outils-web.md`

---

*Manuel 29 — Aide et ressources — Constructo AI ERP — révision 3.0, juillet 2026*
