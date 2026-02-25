# 🚑 Projet : Scanner des Prescriptions Médicales de Transport (Cerfa 11574)

## 📖 Présentation du Projet

Cette application mobile (fait sur WINDEV Mobile) permet de numériser, traiter et analyser automatiquement les prescriptions médicales de transport (Cerfa 11574).
Le système est capable de redresser l'image, d'identifier la version du document (des anciennes versions 01 aux plus récentes 07) et de lire l'état de la case ALD (Affection de Longue Durée) grâce à une analyse dynamique des pixels.

---

## ⚙️ Architecture et Flux de Traitement (Workflow)

1. **Prise de vue :** L'utilisateur prend un scan de document.
2. **Redressement (Native Java) :** Le code détecte la bande noire supérieure du Cerfa, calcule l'angle d'inclinaison et recadre parfaitement le document.
3. **Détection de Version :** L'OCR lit le coin supérieur gauche pour déterminer l'édition du Cerfa (ex: 01, 07).
4. **Ciblage ALD :** Selon la version détectée, l'algorithme cible des coordonnées précises pour vérifier le taux de remplissage des cases "OUI" et "NON".
5. **Validation :** L'utilisateur valide les données extraites avant l'envoi vers la base de données.

---

## 🗂️ Structures de Données

### `ST_ResultatOCR`

Structure centrale agissant comme un conteneur de données pour centraliser les retours de l'analyse OCR.

- `VersionDetectee` (chaîne) : Numéro de la version (ex: "06") ou vide si non reconnue.
- `EstALD` (booléen) : Résultat de l'analyse de la case (Vrai si exonérante).
- `ImageDessinee` (Image) : Le scan final redressé et recadré.

---

## 🛠️ Description des Procédures Principales

### 1. Gestion de l'Interface et du Flux

- **`CallBackPhoto(nNbImages, tabImagesScannees)`**
  - _Rôle :_ Intercepte le retour de l'appareil photo natif.
  - _Technique :_ Sauvegarde l'image originale et lance le traitement lourd dans un thread secondaire généré dynamiquement (`"Thread_" + DateHeureSys()`) pour éviter de figer l'interface graphique (UI) et empêcher les conflits de processus.

- **`SuiteTraitementOCR(imgDeTravail)`**
  - _Rôle :_ Chef d'orchestre de l'analyse. Affiche le sablier de chargement et le dialogue de validation final.
  - _Technique :_ Implémente un "Flag d'exécution" (`gbAnalyseEnCours`) couplé à la désactivation du bouton principal. Cette exclusion mutuelle (sécurité anti-rebond) empêche le lancement de traitements simultanés.

### 2. Traitement d'Image Avancé

- **`_Jv_RedresserEtRogner_ToutEnUn(imageBytes)`** _(Code Natif Java)_
  - _Rôle :_ Nettoie le scan brut pour en faire un document parfait.
  - _Technique :_ Utilise une méthode de multi-lasers virtuels et de régression linéaire pour trouver la bande noire du Cerfa. Applique une matrice de rotation pour corriger l'angle, puis rogne l'image selon des marges définies à l'emporte-pièce.

- **`AnalyseRemplissageZone(imgSource, nX_Pct, nY_Pct, nL_Pct, nH_Pct)`**
  - _Rôle :_ Détermine si une case spécifique est cochée.
  - _Technique :_ Découpe la zone ciblée et effectue un étalonnage dynamique de la luminosité (pour s'adapter aux ombres ou aux flashs). Fixe un seuil à 70% de la luminosité du papier pour compter le pourcentage de "pixels d'encre".

### 3. OCR et Extraction de Données

- **`LancerTraitementOCR(imgSource)`**
  - _Rôle :_ Coordonne la lecture de l'image. Isole la zone haut-gauche pour lancer la lecture du texte de la version, puis appelle le système de vérification de l'ALD.

- **`RecupereVersionCerfa(sTexteOCR)`**
  - _Rôle :_ Extrait fiablement la version du Cerfa, malgré les erreurs fréquentes de l'OCR.
  - _Technique :_ Nettoyage maximal via un tableau de caractères parasites passé à la fonction `Remplace`. Utilisation d'Expressions Régulières (Regex) assouplies pour pallier les confusions courantes de l'OCR (ex: '7' lu comme 'T', '4' lu comme '0').

- **`ALDFinder(imgSource, sVersion)`**
  - _Rôle :_ Cherche les cases ALD au bon endroit.
  - _Technique :_ Utilise une structure conditionnelle (`SELON sVersion`) pour adapter les coordonnées X/Y de recherche, car la mise en page des vieux Cerfa (01, 02) diffère des versions modernes (06, 07).

---

## 🔒 Sécurités Intégrées

- **Blocage des scans illisibles :** Si la version n'est pas détectée ou si l'angle de rotation excède 15°, le traitement est avorté proprement avec un message demandant à l'utilisateur de recommencer.
- **Tolérance aux erreurs matérielles :** Le calcul de luminosité dynamique évite les faux positifs liés aux mauvais éclairages.
