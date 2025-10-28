# 🧭 Spécification — SIRH MSF (Prototype HTML standalone)

## 1) Contexte
- **Objectif** : proposer un **prototype autonome** de consultation RH centré sur la **fiche employé**. L'interface combine une liste filtrable et une fiche détaillée générées côté navigateur à partir d'un jeu de données aléatoire.
- **Utilisateurs** : responsables RH de terrain souhaitant naviguer rapidement dans les informations principales d'un collaborateur.
- **Périmètre de cette démo** :
  - **Affichage** d'une liste d'employés filtrable par **terrain**, **type de contrat**, **catégorie socio-professionnelle**, **statut** et **recherche texte**.
  - **Consultation** d'une fiche employé riche : identité, coordonnées, contrat courant, historique de contrats, dépendants, congés, qualifications et bulletins de paie.
  - **Statistiques** synthétiques en pied de page (volume d'employés listés, filtres actifs...).
- **Thème & UX** : thème clair bleu, layout maître/détail, badges et accords pliables (accordéons) pour l'historique et les bulletins.
- **Hypothèses** :
  - Jeu de données **aléatoire** généré à l'initialisation (~60 employés).
  - **5 terrains** fictifs en RDC avec devise USD.
  - Contrats, bulletins et autres informations calculées aléatoirement mais cohérentes.
- **Hors périmètre** : gestion multi-onglets, modification des données, persistance, validation de périodes.

---

## 2) Données (lisible, non-technique)

### 2.1 Entités & champs essentiels
- **Terrain** : `code`, `name`, `flag`, `currency` (liste fixe `TERRAINS`).
- **Employé** 👤 : `id`, `civility`, `name`, `role`, `contractType`, `socioCategory`, `echelon`, `level`, `status`, coordonnées (`email`, `phone`, `address`), `birthDate`, `maritalStatus`, `terrain` (référence), `baseSalary`.
- **Contrat (historique)** 📝 : collection `contractHistory[]` avec `label`, `contractType`, `startDate`, `endDate`, `functionName`, `socioCategory`, `echelon`, `level`, `gridFunction`, `salaryGrid`, `baseSalary`, `currency`.
- **Contrat courant** : dernière entrée de `contractHistory`, exposée sous `currentContract`.
- **Dépendant** 🧒 : `id`, `name`, `relation`, `birthDate`, `active`, attributs optionnels (`childStatus`, `handicap`, `worker`).
- **Congé** 🌴 : `id`, `type`, `startDate`, `endDate`, `days`, `counted`.
- **Qualifications** 🎓 : `educationLevel`, `diploma`, `year`, `domains[]`, `languages[]` (nom + niveau), indicateurs `languageFlags` (fr/es/en).
- **Bulletin de paie** 🧾 : `id`, `period` (`YYYY-MM`), `status`, `method`, `currency`, `netAmount`, `lines[]` (libellé, montant, catégorie, tonalité visuelle).
- **Périodes de paie** : liste fixe `PERIODS` (`2025-01` → `2025-04`).

### 2.2 Relations
- 1 **terrain** → N **employés** (affectation directe via `employee.terrain`).
- 1 **employé** → N **contrats** (`contractHistory`) dont **1 actif** (`currentContract`).
- 1 **employé** → N **dépendants**, N **congés**, 1 bloc **qualifications**.
- 1 **employé** → N **bulletins** (1 par période de la liste `PERIODS`).

### 2.3 Règles métier clés
- **Contrat courant** : dernière entrée de l'historique, forcée en `endDate = null`.
- **Salaire** : incrément progressif sur les entrées d'historique (augmentation lors des renouvellements/amendements).
- **Bulletins** : statut "Payé" par défaut sauf dernier mois parfois "À payer" ; calcul des lignes dérivé du salaire de base et de montants aléatoires cohérents (gains/déductions/synthèse).
- **Congés** : chaque employé dispose de 1 à 3 entrées, `counted` indique la prise en compte dans la paie.
- **Dépendants** : 0 à 4 entrées, attributs spécifiques selon le lien (enfant vs adulte).
- **Filtres** : application combinée (terrain + recherche + selecteurs) sur la liste d'employés.

### 2.4 Jeux de données de démo
- **Terrains** : `CD_KIN1`, `CD_LUB1`, `CD_GOM1`, `CD_KIS1`, `CD_MBA1` (tous 🇨🇩, devise USD).
- **Contrats** : 2 à 3 entrées historiques par employé ; grilles nommées `GF_<code terrain>_V1` / `GS_<code terrain>_V1`.
- **Rôles** : tirés de `FUNCTIONS` (ex. Caissier, Analyste, Directeur de terrain, Technicien WASH...).
- **Bulletins** : 4 mois (janvier → avril 2025) avec lignes standards (gains, retenues, résumé) et possibilité d'acompte/prêt.
- **Autres référentiels** : types de contrat (`CDD`, `CDI`, `Consultant`, `Prestation`, `Stage`), catégories socio-prof (`EMP`, `AMT`, `CAD`), modes de paiement (`Virement`, `Cash`, `Mobile`), types de congés, niveaux linguistiques (`A2` → `C2`).

---

## 3) Interface

### 3.1 Layout & thèmes
- **Topbar sticky** avec :
  - **Logo / titre** "Homere Connect — Prototype".
  - **Sélecteur Terrain** (liste `TERRAINS`).
  - **Recherche globale** (champ texte).
  - **Filtres** par type de contrat, catégorie socio-professionnelle, statut.
- **Layout maître/détail** (`.layout`) : panneau gauche listant les employés, panneau droit affichant la fiche détaillée.
- **Carte employé** : avatar initiales, nom, poste, terrain (code + ville), statut badge, info salaire/contrat.
- **Thème visuel** : palette bleue (accent `#3C71A5`), cartes arrondies, accordéons pour sections historisées.

### 3.2 Onglet **Employés** 👩‍⚕️👨‍⚕️
- **Liste gauche** :
  - Affiche dynamiquement tous les employés filtrés.
  - Chaque ligne montre avatar, nom, poste, terrain, statut et contrat courant.
  - Survol avec élévation ; clic ⇒ charge la fiche détaillée.
- **Fiche droite** (après sélection) :
  1) **En-tête** : nom, rôle, terrain, statut, type de contrat, catégorie, compteur de dépendants.
  2) **Informations personnelles** : civilité, naissance, âge, coordonnées, adresse, statut marital, échelon/niveau.
  3) **Historique des contrats** : accordéons détaillant dates, grilles, salaire, catégories.
  4) **Personnes à charge** : tableau (nom, date de naissance, lien, statut enfant, handicap, travailleur, actif) ou message vide.
  5) **Congés** : tableau type, dates, durée, indicateur de comptabilisation paie.
  6) **Qualifications** : niveau d'études, diplôme, année, domaines, langues + drapeaux Oui/Non.
  7) **Bulletins de salaire** : accordéons par mois avec statut, net, mode de paiement, détail des lignes (gains/déductions/synthèse).
- **Actions** : uniquement navigation et ouverture/fermeture des accordéons (pas de modification).

### 3.3 Onglet **Contrats** 📝
- **Non implémenté** dans ce prototype (aucun changement de contenu quand on clique : onglet inexistant).

### 3.4 Onglet **Configuration** ⚙️ (modèle de bulletin par **site**)
- **Non implémenté** dans ce prototype (pas d'édition ni d'aperçu de modèle).

### 3.5 Parcours critiques (acceptance)
- **Filtrer par terrain** : le sélecteur rafraîchit la liste et réinitialise la fiche (message "Bienvenue").
- **Recherche / filtres** : modifier la recherche ou un filtre déclenche un recalcul instantané de la liste et du compteur.
- **Consulter un employé** : clic sur une ligne ⇒ chargement complet de la fiche avec accordéons fonctionnels.
- **Naviguer dans les bulletins** : clic sur un bulletin ⇒ affiche/masque le détail des lignes correspondantes.

---

## 4) Technique

### 4.1 Livraison
- **Un seul fichier** `sirh-ui-msf.html` (HTML, CSS et JS embarqués). Ouvrable directement dans un navigateur moderne.

### 4.2 Architecture (démo)
- **Données en mémoire** : constantes (`TERRAINS`, `CONTRACT_TYPES`, `SOCIO_CATEGORIES`, etc.) et `state` (`employees`, `selectionId`).
- **Génération aléatoire** : fonctions utilitaires (`rand`, `pick`, `uid`) pour créer employés, dépendants, congés, qualifications, bulletins et historiques de contrat lors de `init()`.
- **Rendu** :
  - **Liste** générée via `renderList()` (filtre sur terrain sélectionné, recherche, selecteurs) + insertion HTML.
  - **Fiche** rendue par `renderDetail(employee)` avec templates string et binding des accordéons.
- **Interactions** : écouteurs `change`/`input` sur les filtres et la recherche ; clic sur éléments de liste pour sélectionner un employé ; accordéons pilotés par `data-target`.
- **Internationalisation** : formatage des dates/mois via `Intl.DateTimeFormat('fr-FR')` et `toLocaleDateString`.

### 4.3 Extensibilité (pistes v2)
- Ajouter de **nouveaux onglets** (Contrats, Configuration) ou actions (édition, validation) pour couvrir le périmètre fonctionnel complet.
- Remplacer les données aléatoires par une **API** ou un stockage local (`localStorage`).
- Introduire une **gestion d'état** plus structurée (store, frameworks) et une pagination/virtualisation pour de plus grands volumes.
- Prévoir un **moteur de calcul** configurable pour les bulletins (formules, paramètres). 

### 4.4 Définition de Fini (DoD)
- [ ] Liste Employés filtrable par terrain, recherche, type de contrat, catégorie, statut.
- [ ] Fiche employé complète avec sections contractuelles, paie, congés, qualifications.
- [ ] Accordéons fonctionnels pour historique contrats et bulletins.
- [ ] Statistiques de synthèse en pied de page.
- [ ] Génération aléatoire cohérente des données à l'initialisation.
