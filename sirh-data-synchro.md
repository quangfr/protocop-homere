# Spécification — Prototype SIRH Synchro (2 sites + 1 central)

## 1) Contexte & objectifs
- **But** : Illustrer deux stratégies de synchronisation métier pour un SIRH avec paie, multi-sites, **fonctionnant hors-ligne** côté sites.
- **Périmètre** : 2 sites (A, B) + 1 central, **2 employés**, objets & actions :
  - **Contrats** : éditer le *modèle* (grille salaire fixe + HS), choisir le modèle d’un site.
  - **Bulletins** : clôture période, ajout de variables mensuelles (ABSNP, HS, FRAIS).
  - **Employés** : création, MAJ infos perso, affectation site-contrat, terminer contrat, amender contrat (grade, indemnités).
- **Stratégies** :
  - **S1 — Simple & Carrée** : central prioritaire (contrats & règles), peu de médiation.
  - **S2 — Fusion intelligente** : auto-fusion là où sûr, médiation ciblée sur vrais conflits.
- **Non-objectifs** : pas de connexion serveur réelle, pas de calculs paie, pas de persistance.

---

## 2) Données (modèle & jeux d’essai)

### 2.1 Modèle de données (simplifié)
- **Central**
  - `models`: `{ id, name, version, overtimeRate(%), salaryGrid{grade:€} }`
  - `employees`: `{ id, name, site, personal{address,rib}, contract{model,grade,allowance,endDate|null} }`
  - `variables`: `empId -> period -> [{type: 'ABSNP'|'HS'|'FRAIS', value:number}]`
  - `period`: ex. `"2025-09"`
  - `centralLog`: `string[]`
- **Site** (A/B)
  - `online: boolean`
  - `local`: *copie des employés du site (vue locale)*
  - `modelsCache`: *copie du catalogue central au dernier refresh*
  - `changes`: *file d’événements locaux à envoyer au central*
  - `log`: `string[]`
  - `localClosed`: `boolean` (pré-clôture locale)

### 2.2 Jeux d’essai
- **Modèles** :
  - `A = {name:"Modèle A", version:1, overtimeRate:25, salaryGrid:{1:1800,2:2000,3:2300}}`
  - `B = {name:"Modèle B", version:1, overtimeRate:30, salaryGrid:{1:1700,2:1950,3:2200}}`
- **Employés** :
  - `E1: Alice Martin` (site A) – `personal:{address:"Lyon", rib:"FR76-***-A"}`, `contract:{model:"A",grade:2,allowance:200,endDate:null}`
  - `E2: Bruno Dupont` (site B) – `personal:{address:"Villeurbanne", rib:"FR76-***-B"}`, `contract:{model:"B",grade:1,allowance:100,endDate:null}`

### 2.3 Types d’événements locaux (sites)
- `create {tempId, name}`  → création d’employé (id central attribué à la synchro)
- `personal {empId, field, value}` → MAJ infos perso (ex. address)
- `variable {empId, period, type, value}`
- `assign {empId, to}` → affectation à site
- `end {empId, date}` → proposition fin de contrat
- `amend {empId, grade, allowance}` → amendement contrat (brouillon en S2)
- `contractModel {empId, model}` → application d’un modèle (version centrale la plus récente)

---

## 3) Interface (UX/UI)

### 3.1 Disposition
- **En-tête** :
  - Sélecteur **Stratégie** (`S1|S2`)
  - Sélecteur **Période** (liste 2025-01…2025-09)
  - Bouton **Réinitialiser la démo**
  - **Badges KPI** (stratégie active, période, 2 employés, 2 sites + 1 central)
- **3 colonnes** (cartes) : `Site A` | `Central` | `Site B`
  - État **En ligne / Hors-ligne** (badge)
  - **Toolbar site** : *Basculer en ligne/hors-ligne*, **Sync → Central**
  - **Toolbar central** : **Sync global (tous sites)**, **Sceller la clôture**

### 3.2 Panneau Site (A/B)
- **Employé** (select) – liste des employés **locaux** du site.
- **Contrat / Modèle**
  - Select **modèle** (depuis cache local du catalogue central)
  - Bouton **Appliquer au contrat**
- **Bulletins / Variables mensuelles**
  - Select `Type` (ABSNP | HS | FRAIS) + Input `Valeur` + **Ajouter**
- **Employés / Gestion**
  - **Changer adresse** (input + bouton)
  - **Affecter au site** (select A/B + bouton)
  - **Terminer contrat** (date + bouton)
  - **Amender contrat** (grade, indemnités + bouton)
    - **S1** : bouton **grisé** (désactivé)
    - **S2** : actif (brouillon, arbitrage à la synchro)
  - **Créer employé** (nom + bouton)
  - **Pré-clôturer localement** (brouillon)
- **Journal du site** (log défilant, derniers en tête)

### 3.3 Panneau Central
- **Employé central** (select) + **Amendement contrat** (grade, indemnités, bouton)
- **Catalogue modèles (central)** :
  - Select **modèle** + inputs `overtimeRate`, `salaryGrid[grade=2]`
  - **Mettre à jour le modèle (↑ version)** → incrémente `version` (les sites n’ont la nouvelle version qu’après refresh/sync)
- **Journal central** (log défilant)

### 3.4 États & contraintes UX
- **Hors-ligne site** :
  - Toute action **crée un événement local** sauf *Sync → Central* (grisé).
  - Boutons **grisés** lorsqu’une action nécessite explicitement une connexion (ex. Sync).
- **Changement de stratégie** : met à jour un badge & règles de résolution.
- **Après Sync** : rafraîchit les caches (`local`, `modelsCache`) depuis le central.

---

## 4) Technique (comportements & règles)

### 4.1 Cycle de vie
1. **Init** : hydratation des listes (employés, modèles, période), logs “Démarrage”.
2. **Actions locales** (sites) : ajoutent des **événements** dans `changes` + trace dans le log du site.
3. **Sync → Central** (site) :
   - Si **hors-ligne** → log “Sync impossible”.
   - Sinon : dépile `changes` et applique au **central** selon la **stratégie**.
   - Log central + log site pour chaque événement traité.
   - **Refresh** des caches du site depuis le central (snap-back cohérent).
4. **Sync global** (central) : chaîne `Sync → Central` pour A puis B.
5. **Clôture** (central) :
   - **S1** : log “Synchro obligatoire avant clôture” (illustratif).
   - **S2** : log “Pré-contrôles locaux + arbitrages appliqués”.

### 4.2 Règles de résolution — S1 vs S2

| Cas | S1 — Simple & Carrée | S2 — Fusion intelligente |
|---|---|---|
| **Variables** (ABSNP/HS/FRAIS) | *Fusion par type* (la valeur locale **remplace** la précédente pour la période) | *Auto-fusion* : si même type existe → **addition/union** (ex. HS cumulées) |
| **Infos perso** (ex. adresse) | Site met à jour central (pas de médiation) | *Dernier à écrire gagne* + journalisation ; champs sensibles (RIB) pourraient déclencher médiation (log) |
| **Affectation site** | Validée par le central (contrôles simples) | Idem + auto-vérif règles ; médiation si impact paie inter-sites |
| **Fin de contrat** | Si divergence : **central prioritaire** ; sinon, accepte la proposition du site | Si divergence : **médiation simulée** (ex. conserve date centrale) ; sinon, accepte |
| **Amendement contrat** (grade/indemnités) | **Interdit côté site** (bouton grisé) | *Brouillon site* ; à la synchro : **grade central prioritaire**, **indemnités site acceptées** (ex. règle de compromis) |
| **Modèle de contrat** (application) | Applique **dernière version** centrale | Idem, avec contrôle de compatibilité + log de médiation si versionnage litigieux |
| **Création employé** | Id site → **id central attribué** ; dédoublonnage simplifié | Idem ; *auto-merge* si doublon détecté (heuristique NIR/email + nom), sinon médiation |

> **Note** : Les “médiations” sont **illustrées via les logs** (pas d’écran dédié dans ce prototype). Un écran 3-colonnes (Site/Central/Diff) peut être ajouté ultérieurement.

### 4.3 États, permissions, habilitations
- **Central** : toujours “en ligne”.
- **Sites** : bascule **en ligne / hors-ligne** (impacte seulement le bouton *Sync → Central*).
- **S1** : actions *contrat structurel* (amendements) **réservées** au central.
- **S2** : sites peuvent *proposer* ; le central **arbitre par règle** (voir tableau).

### 4.4 Journaux (logs) — Exemples
- Site : `🏠 Adresse de Alice mise à jour localement → "12 rue des Lilas".`
- Site : `🔄 Envoi 3 changement(s) au central…`
- Central : `🧾 Variables 2025-09 mises à jour pour Alice (HS=3) — règle S2: auto-fusion.`
- Central : `🧮 Conflit amendement Alice (S2): grade central conservé (2), indemnités site appliquées (250).`
- Site : `🟨 Médiation: grade central gardé, indemnités locales appliquées.`
- Central : `📚 Modèle Modèle A mis à jour → v2 (HS=27%, G2=2050€).`

### 4.5 Extensibilité (optionnel)
- Écran **Médiation 3-colonnes** (Site / Central / Résultat) pour champs conflictuels.
- Table **“champs non-fusionnables”** configurable (grade, salaire de base, date d’embauche…).
- Hooks de **pré-clôture** (check manquants / blocants).
- Règles de **priorisation temporelle** (timestamps, auteurs).