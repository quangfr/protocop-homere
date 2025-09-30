# 🧭 Spécification — SIRH MSF (Prototype HTML standalone)

## 1) Contexte
- **Objectif** : proposer un **SIRH léger** pour managers RH de sites MSF, avec **liste Employés**, **liste Contrats**, **configuration de modèle de bulletin**, et **génération/verrouillage** des bulletins de paie **par site et par période**.
- **Utilisateurs** : managers RH de **site** (et, à terme, RH pays / paie).
- **Périmètre v1** :
  - Consultation & navigation **Employé ⇄ Contrat ⇄ Bulletins**.
  - **Validation** d’une période de paie (génère/verrouille les bulletins du site sélectionné).
  - **Édition locale** du **modèle de bulletin** (variables + formules) au niveau **site**.
  - **Avenant** sur un contrat (base, indemnités, date d’effet) + **frais de mission mensuels**.
- **Thème & UX** : thème **clair**, accent **rouge MSF** ⛑️, emojis métiers (🧑‍⚕️, 📝, 🧾, ⚙️).
- **Hypothèses** :
  - 2 pays (🇫🇷 **France** en **EUR**, 🇨🇭 **Suisse** en **CHF**), **6 sites** (3 par pays).
  - ~**100 employés** répartis sur les 6 sites.
  - **1–3 contrats** par employé (historiques), **1 seul actif** à un instant T.
  - **Périodes de paie existantes** : **janvier, février, mars 2025** (déjà générées/validées). **Avril 2025** = période **courante** (ouverte).
- **Hors périmètre v1** : authentification, sécurité d’entreprise, stockage serveur, conformité fiscale par pays, export comptable/banque.

---

## 2) Données (lisible, non-technique)

### 2.1 Entités & champs essentiels
- **Pays** : nom, code, **devise**, paramètres paie par défaut (ex. taux sociaux/impôt de démo).
- **Site (terrain)** : nom de ville, pays, devise par défaut, jours fériés (optionnel), **modèle de bulletin actif**.
- **Employé** 👤 : identité, coordonnées (tél, email, adresse), **statut** (Actif, En mission, Sorti), **qualifications** (avec dates), **dépendants** (nom, lien, naissance), alertes docs (visa, ID).
- **Contrat** 📝 : employé, site/pays, **poste**, **période** (début/fin), **type** (local/expat), **grille salariale** (version), **salaire de base**, **indemnités** (logement %, plafonds, hardship), **devise**, **statut validation** (brouillon/en revue/validé/clos), **avenants** (avant/après + date d’effet).
- **Grille salariale** 💸 (simplifiée démo) : version par **pays**, **base par poste**, période de validité.
- **Variables mensuelles** ⏱️ : par **(contrat, période)** — **frais de mission**, primes exceptionnelles, heures sup, absences (v1 : frais de mission).
- **Modèle de bulletin** ⚙️ : porté par **site** (hérite d’un **modèle par pays**), **variables éditables** (taux/ratios/plafonds) + **lignes** (libellé, section : gains/retenues/synthèse, **formule** lisible).
- **Bulletin de paie** 🧾 : (employé, contrat, site, **période**), lignes calculées, totaux (gains, retenues, **net à payer**), **statut** (Calculé/Validé/Payé), **modèle utilisé** (nom/version), **date de génération**.

### 2.2 Relations
- 1 **pays** → N **sites**.  
- 1 **site** → N **contrats** ; **0..1 modèle** actif (par version).  
- 1 **employé** → N **contrats** (mais **1 seul actif** à T).  
- 1 **contrat** → N **bulletins** (1 par période), N **variables mensuelles** (0..1 par période).  
- 1 **bulletin** est généré à partir d’**1 modèle** (version), **1 contrat**, **1 période**.

### 2.3 Règles métier clés
- **Unicité contrat actif** par employé (pas de chevauchement).
- **Validation de période** (site + mois) ⇒ **génère & verrouille** tous les bulletins **Calculés** de cette période/site via le **modèle actif** à l’instant T.  
  **Anciennes périodes restent figées** (pas d’effet rétroactif).
- **Modèle de bulletin** : modifiable (variables + formules), **prévisualisable** sur un cas, **activable** pour le site.  
- **Avenant** : capture **avant/après** + **date d’effet** ; contrôle basique anti-chevauchement ; met à jour base/indemnités futures.
- **Frais de mission** : saisis **par période**, injectés dans le calcul.

### 2.4 Jeux de données de démo
- **Pays/Sites** :  
  - 🇫🇷 **France** : Paris, Lyon, Marseille (EUR)  
  - 🇨🇭 **Suisse** : Genève, Lausanne, Zurich (CHF)
- **Rôles** (ex.) : Infirmier·e, Médecin, Logisticien·ne, Admin RH, Pharmacien·ne, Tech WASH, Sage-femme.
- **Grilles (démo)** : FR-Std-v1 / CH-Std-v1 avec **base par rôle** (valeurs pédagogiques).
- **Modèles par pays (démo)** :
  - **Variables** : `indemn_housing_pct` (FR 20% / CH 25%), `housing_cap` (FR 800 / CH 1200), `rate_social` (FR 0,22 / CH 0,15), `rate_tax` (FR 0,12 / CH 0,10)
  - **Lignes type** :  
    - Gains : *Salaire de base* `= base_salary` ; *Indemnité logement* `= min(indemn_housing_pct * base_salary, housing_cap)` ; *Hardship* `= hardship` ; *Frais de mission* `= mission_expenses`  
    - Synthèse : *Total gains* `= base + housing + hardship + mission`  
    - Retenues : *Cotisations sociales* `= rate_social * gross` ; *Impôt* `= rate_tax * max(gross - social, 0)`  
    - Synthèse : *Net à payer* `= gross - social - tax`
- **Périodes** : **Jan, Fév, Mar 2025** (déjà **Validé**), **Avr 2025** (ouverte).

---

## 3) Interface

### 3.1 Layout & thèmes
- **Thème clair**, accent **rouge MSF** ⛑️ pour CTA/badges actifs.
- **Barre supérieure** (sticky) :  
  - **Sélecteur Site** (limité au périmètre utilisateur)  
  - **Sélecteur Période** (mois/année)  
  - **Bouton** **Valider la période**  
  - **Recherche globale** (nom, poste)
- **3 onglets** (maître-détail) : **Employés**, **Contrats**, **Configuration**.
- **Panneau gauche** : **liste compacte** ; **panneau droit** : **fiche détaillée**.

### 3.2 Onglet **Employés** 👩‍⚕️👨‍⚕️
- **Liste** (gauche) : avatar, **Nom**, **Poste**, **Site** (drapeau + ville), **statut** (badge), **contrat courant** (statut), **compteur bulletins** sur la période sélectionnée, indicateur ⚠️ docs/qualifs.
- **Filtres** rapides : statut, poste, alertes docs.
- **Fiche Employé** (droite) — sections :
  1) **Aperçu** : identité, coordonnées, dépendants, alertes, **résumé paie** période courante.  
  2) **Contrat courant** : poste, site, dates, **grille** (version), **base**, **indemnités** (valeurs site + overrides).  
  3) **Historique contrats** : versions/avenants.  
  4) **Fiches de paie** : liste par mois (statut, totaux, lien).  
  5) **Qualifs & docs** : compétences/langues/certificats (validité).  
  6) **Variables mensuelles** (période sélectionnée) : **Frais de mission** (montant + note).  
- **Actions** :
  - **Aperçu bulletin** avec le **modèle actif** (sans figer).
  - **Naviguer** vers la **Fiche Contrat** (contrat courant).

### 3.3 Onglet **Contrats** 📝
- **Liste** (gauche) : Employé, Poste, Site, **Période**, **Statut**, **Grille** (version), **Base**, badge Indemnités (défaut/modifiées), **Devise**, **# bulletins** (période courante).  
  **Filtres** : site, statut, fin ≤30j, grille, devise, “sans bulletin période”.
- **Fiche Contrat** (droite) — sections :
  1) **Infos générales** : employé, poste, site/pays, type, quotité, périodes.  
  2) **Rémunération** : grille (version), **base**, **indemnités** (logement %, plafonds, hardship), devise.  
  3) **Variables mensuelles** (période) : frais de mission, (primes, H. sup — option v2).  
  4) **Fiches de paie** : liste (ouvrir).  
  5) **Historique des versions** : avenants (avant/après, date d’effet).
- **Actions** :
  - **Amender (nouvelle version)** : **base** (alignée à la **grille**), **indemnités** vs valeurs par défaut site, **date d’effet** (contrôle chevauchement).  
  - **Clore** le contrat (définir fin + motif).  
  - **Naviguer** vers **Fiche Employé**.

### 3.4 Onglet **Configuration** ⚙️ (modèle de bulletin par **site**)
- **Variables éditables** (types : montant, %, bool, formule) : `indemn_housing_pct`, `housing_cap`, `rate_social`, `rate_tax`, etc.
- **Lignes & formules** (éditeur type “tableur” lisible) : section, libellé, identifiant, **formule** (cf. glossaire 2.4).
- **Aperçu** : appliquer le **modèle** du site sur un **cas employé + période** (rendu bulletin à droite).
- **Activer** le modèle pour le site (ne **modifie pas** les bulletins déjà validés).
- **Rappels UX** : info-bulles sur variables, avertissement sur effets non rétroactifs.

### 3.5 Parcours critiques (acceptance)
- **Valider période** : “Site = Genève, Période = Avril 2025” ⇒ génère **1 bulletin/contrat actif** du site, statut **Validé**, **inchangé** sur Jan–Mar.  
- **Modifier modèle** puis **Activer** ⇒ seul **prochain** “Valider période” l’utilisera.  
- **Saisir frais de mission** (contrat, période) ⇒ visible dans **aperçu** puis dans bulletin **Validé**.  
- **Avenant** (base + logement % + plafond + hardship, date d’effet) ⇒ impacte les **périodes à partir de la date** (v1 démo : application immédiate).

---

## 4) Technique

### 4.1 Livraison
- **Un seul fichier** `index.html` (HTML + CSS + JS embarqués). Ouvrable directement dans un navigateur moderne.

### 4.2 Architecture (démo)
- **Données en mémoire** (JS) :  
  - `COUNTRIES`, `GRIDS`, `DEFAULT_MODELS` (démo),  
  - `DB.employees[]`, `DB.contracts[]`, `DB.payroll[period]{}`, `DB.configBySite[siteKey]`, `DB.variablesByContractPeriod[key]`.
- **Seed** : génération aléatoire ~100 employés, 1 contrat actif/employé, bulletins **Validés** pour **Jan–Mar 2025**.
- **Sélecteurs** : **Site** (`FR:Paris`…​) & **Période** (`YYYY-MM`).  
- **Formules** : mini-moteur (type `new Function`) **restreint au scope** des variables autorisées (`base_salary`, `indemn_housing_pct`, `housing_cap`, `hardship`, `mission_expenses`, `rate_social`, `rate_tax`, + symboles intermédiaires comme `base`, `housing`, `gross`, `social`, `tax`, `net`; fonctions `min`, `max`).  
  > ⚠️ Démo uniquement (pas de sandbox robuste ; ne pas utiliser en production).
- **Calcul bulletin** : `computePayslip(contract, period, model, variables)` retourne lignes + totaux (arrondis 2 décimales), **modèle** référencé.
- **Validation période** : (site, période) ⇒ (re)calcule tous les bulletins **au statut Validé** avec le **modèle actif** du site ; **fige** le résultat pour cette période.
- **Navigation** : maître-détail par onglet ; **recherche** full-text simple (nom/poste) ; **filtres** basiques.
- **Performances** (démo) : 100 employés / 6 sites ⇒ rendu instantané sans virtualisation.

### 4.3 Extensibilité (pistes v2)
- **Persistance** : `localStorage` (démo) ou backend REST (Node/Python) + DB (Postgres).  
- **Sécurité** : auth/rôles (JWT) ; audit étendu.  
- **Fiscalité** : tables de cotisations/IR réelles par pays ; multi-devises & taux de change.  
- **Exports** : PDF bulletins, CSV variables/paie, interface banque.  
- **Qualifs/Docs** : upload & workflow d’expiration (alertes).  
- **Moteur de formules** sécurisé (parser, AST, sandbox).

### 4.4 Définition de Fini (DoD)
- [ ] Liste Employés filtrable + fiche employé complète.  
- [ ] Liste Contrats filtrable + fiche contrat (avenant opérationnel).  
- [ ] Onglet Configuration : édition **variables** & **formules**, **aperçu**, **activation**.  
- [ ] **Validation période** génère/verrouille bulletins **uniquement** pour le **site+période** sélectionnés.  
- [ ] **Jan–Mar 2025** visibles comme **déjà validés** ; **Avr 2025** ouverte.  
- [ ] Navigation **Employé ⇄ Contrat** et consultation **bulletins** par les deux entrées.  
- [ ] Thème clair + accent rouge MSF + emojis.
