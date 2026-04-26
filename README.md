# 🛡️ Lab de Sécurité Android : Rooting, Verified Boot et Analyse d'Impact

Ce laboratoire a pour but de comprendre l'impact du rooting sur un environnement Android, d'analyser les mécanismes de sécurité (Verified Boot, AVB) et d'introduire les standards de test de sécurité mobile (OWASP MASVS/MASTG).

> **Remarque pédagogique :** Le "rooting" est l'équivalent Android de l'accès administrateur sur Windows ou du "jailbreak" sur iOS. Il s'agit d'obtenir les privilèges les plus élevés (uid=0) sur le système.

---

## ⚠️ Règles de Sécurité et Périmètre

Ces règles sont **essentielles**. Travailler avec des données réelles ou sur un appareil personnel peut compromettre vos informations ou endommager (bricker) votre appareil.

* **Environnement :** Application de test + Laboratoire uniquement (AVD ou Device Labo dédié).
* **Données :** Fictives uniquement.
* **Réseau :** Isolé pour les tests.
* **Isolement :** Rien ne sort du périmètre de test. Ne **jamais** flasher ni manipuler un appareil personnel.

---

## 🛠️ Phase 1 : Préparation de l'Environnement

### 1. Démarrer un AVD (Android Virtual Device) propre
* Lancer Android Studio → Device Manager → Start (privilégier une API 29+ pour observer les protections modernes).
* Vérifier l'absence de compte personnel.

### 2. Vérification de l'outil ADB
ADB (Android Debug Bridge) est l'outil principal pour communiquer avec l'appareil.
```bash
adb devices
```
<img width="769" height="94" alt="image" src="https://github.com/user-attachments/assets/d28976d3-b52c-4c60-bdc7-3ed7dd2ffafe" />


### 3. Installation de l'application de test
```bash
adb install app-debug.apk
```

<img width="592" height="907" alt="image" src="https://github.com/user-attachments/assets/e107f15c-3668-4fec-b47d-fb9fffb72072" />

## 🔓 Phase 2 : Exploitation (Rooting) et Preuve d'Impact

Le rooting modifie les protections fondamentales de l'OS. Pour démontrer ce risque (violation de la règle OWASP MASVS STORAGE-1), nous avons ciblé l'application `com.example.starsgallery` dans un environnement virtuel contrôlé (AVD "Google APIs").

> **Note technique :** Les commandes ci-dessous ont été exécutées via PowerShell sous Windows. L'AVD a été préalablement lancé depuis le Device Manager d'Android Studio.

### Étape 1 : Preuve d'isolation (Sandbox Active)
Avant toute manipulation, nous vérifions que le système Android bloque l'accès aux données privées de l'application pour un utilisateur standard.

```powershell
.\adb shell id
# Résultat : uid=2000(shell)
.\adb shell "ls -l /data/data/com.example.starsgallery"
```
*Le système renvoie une erreur `Permission denied`, prouvant que la Sandbox fonctionne.*
<img width="1827" height="135" alt="image" src="https://github.com/user-attachments/assets/e0fc5781-499f-42a6-8195-77c5d36e17fa" />


### Étape 2 : Élévation de privilèges et altération du système
Nous exploitons l'environnement de développement pour passer Administrateur (`root`) et désactiver le mécanisme de vérification d'intégrité des fichiers (`dm-verity`).

```powershell
.\adb root
.\adb remount
.\adb disable-verity
```
*Le serveur ADB redémarre avec les privilèges root et remonte la partition système en lecture/écriture.*
<img width="889" height="90" alt="image" src="https://github.com/user-attachments/assets/906a2c4b-71bb-4dd9-ba1e-acb7eda4e27e" />


### Étape 3 : Post-exploitation (Contournement de la Sandbox)
Avec nos nouveaux privilèges, nous retournons explorer le répertoire privé de l'application cible.

```powershell
.\adb shell id
# Résultat : uid=0(root)
.\adb shell "ls -l /data/data/com.example.starsgallery/"
```
*La barrière de sécurité est tombée : nous pouvons désormais lister les dossiers internes tels que `databases`, `cache` ou `shared_prefs`.*
<img width="1714" height="78" alt="image" src="https://github.com/user-attachments/assets/338c1ade-b618-4429-b6ed-4100c73438e0" />

<img width="1138" height="116" alt="image" src="https://github.com/user-attachments/assets/024d7124-10cf-49a3-b0e7-fbd7432ead06" />

### Étape 4 : Exfiltration de données sensibles
Nous ciblons le dossier des préférences partagées pour y chercher les actions enregistrées par l'utilisateur (ici, la modification d'une note à 5 étoiles pour un acteur).

```powershell
.\adb shell "cat /data/data/com.example.starsgallery/shared_prefs/*.xml"
```
*Succès de l'attaque : le fichier XML est lu et affiche les données sensibles en clair, prouvant le danger d'un stockage local non chiffré sur un appareil compromis.*
<img width="1233" height="111" alt="image" src="https://github.com/user-attachments/assets/1ebe4662-4e25-41e1-8fea-ea29e06562dc" />


### Étape 5 : Vérification de l'état d'intégrité (Verified Boot)
Nous vérifions si le système a conscience que sa chaîne de confiance a été brisée par notre attaque.

```powershell
.\adb shell getprop ro.boot.verifiedbootstate
.\adb shell getprop ro.boot.veritymode
```
*L'état renvoyé confirme que les protections de bas niveau (dm-verity) ont été désactivées.*
<img width="990" height="86" alt="image" src="https://github.com/user-attachments/assets/1d36d42e-6fcd-44c8-a844-bafa73b6bb30" />

Analyse du résultat : Le terminal ne renvoie aucune valeur. Sur cet environnement virtuel (AVD API 30 Google APIs), les propriétés matérielles de Verified Boot ne sont pas implémentées au niveau du bootloader virtuel, renvoyant des valeurs nulles. Sur un appareil physique de laboratoire, nous aurions attendu l'état orange (bootloader déverrouillé) et une modification de l'état dm-verity.
## 🛡️ Phase 3 : Concepts de Sécurité Android

### Architecture de Sécurité
La sécurité Android repose sur plusieurs couches :
1.  **Sandboxing des applications :** Chaque app est isolée des autres.
2.  **Modèle de permissions :** Contrôle d'accès aux ressources sensibles.
3.  **Isolation et intégrité globale :** Protection contre les modifications non autorisées.

### Verified Boot & AVB (Android Verified Boot)
Verified Boot garantit que le système qui démarre est celui prévu par le fabricant, sans modifications malveillantes. Il repose sur une **Chain of Trust** (série de vérifications où chaque composant vérifie l'authenticité du suivant).

* **AVB (Version 2.0) :** Ajoute une vérification d'intégrité moderne et une protection contre le rollback (empêche d'installer d'anciennes versions vulnérables).
* **États Verified Boot :** * `Green` (Normal, vérifié)
    * `Yellow/Orange` (Avertissement, modifié mais fonctionnel)
    * `Red` (Danger, compromis)

---

## 🧪 Phase 4 : Analyse de Sécurité (OWASP)

### Matrice des Risques liés au Rooting
* **Intégrité non garantie** → conclusions biaisées sur la sécurité réelle.
* **Surface d'attaque accrue** → exposition à des menaces externes.
* **Données sensibles exposées** → violation potentielle de confidentialité.
* **Instabilité système** → tests non reproductibles.
* **Mélange comptes perso/test** → fuite possible d'informations.
* **Mauvais nettoyage** → persistance de données sensibles.
* **Réseau non isolé** → effets involontaires sur systèmes externes.
* **Traçabilité insuffisante** → impossible d'auditer les tests.

### Mesures Défensives Appliquées
* Réseau isolé.
* Données fictives uniquement.
* AVD dédié exclusivement aux tests.
* Wipe en fin de séance.
* Journal de configuration détaillé.
* Aucun compte personnel.
* Contrôle strict des APK installées.
* Horodatage + captures des étapes.

### OWASP MASVS & MASTG (Application Pratique)
En utilisant nos privilèges rootés, nous pouvons vérifier les exigences suivantes :

* **MASVS (Exigences) :**
    * `STORAGE-1` : Les données sensibles doivent être stockées de manière sécurisée (chiffrement).
    * `NETWORK-1` : Les communications réseau doivent utiliser TLS correctement configuré.
* **MASTG (Idées de tests) :**
    * Vérifier les fichiers de préférences (`/data/data/[package_name]/shared_prefs/`) pour y chercher des données en clair.
    * Analyser le trafic réseau et les logs (`adb logcat`) pour détecter des fuites d'informations.

---

## 🧹 Phase 5 : Remise à Zéro (Obligatoire)

Ne pas réinitialiser l'environnement compromet les tests futurs.

### Via Commandes (AVD)
```bash
adb emu avd stop
adb emu avd wipe-data


## 📚 Glossaire Rapide

* **ADB :** Android Debug Bridge.
* **AVD :** Android Virtual Device (émulateur).
* **Bootloader :** Programme chargeant l'OS au démarrage.
* **Root :** Utilisateur avec privilèges administrateur complets.
* **Sandbox :** Environnement d'exécution isolé.
```

