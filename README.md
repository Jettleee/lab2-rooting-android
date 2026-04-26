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
*Si la commande ne renvoie rien, vérifiez que l'émulateur est lancé.*

### 3. Installation de l'application de test
```bash
adb install app-debug.apk
```
*Définir 3 scénarios simples et répétables (ex: Ouvrir l'accueil, Rechercher un item, Ouvrir un détail) et les documenter avec des captures d'écran.*

---

## 🔓 Phase 2 : Rooting et Manipulation Système

Le rooting modifie les protections et la confiance du système. Utile en laboratoire pour observer le stockage et analyser les comportements bas niveau, il augmente drastiquement la surface d'attaque.

### Commandes de Rooting (Émulateur)
```bash
emulator -avd NOM_AVD -writable-system
adb root                # Active le serveur adb en mode root
adb remount             # Monte /system en lecture/écriture
```

### Vérifications de l'état
```bash
adb shell id                                   # Chercher uid=0(root)
adb shell getprop ro.boot.verifiedbootstate    # Attendu: "orange" ou "yellow" si modifié
adb shell getprop ro.boot.veritymode           # Vérifier l'état de verity
adb shell getprop ro.boot.vbmeta.device_state  # État du vbmeta
adb shell "su -c id"                           # Tester la disponibilité de 'su'
```

### Option Permissive (Désactiver Verity)
"Verity" vérifie l'intégrité du système de fichiers. Le désactiver permet des modifications mais supprime la garantie d'intégrité.
```bash
adb disable-verity
adb reboot
adb remount
```

### Journalisation (Traçabilité)
Toujours garder des traces de vos actions :
```bash
adb logcat -d | tail -n 200 > logcat_root_check.txt
```

---

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
```
*Preuve exigée : Capture d'écran de l'assistant initial Android au redémarrage.*

---

## 📚 Glossaire Rapide

* **ADB :** Android Debug Bridge.
* **AVD :** Android Virtual Device (émulateur).
* **Bootloader :** Programme chargeant l'OS au démarrage.
* **Root :** Utilisateur avec privilèges administrateur complets.
* **Sandbox :** Environnement d'exécution isolé.
```

