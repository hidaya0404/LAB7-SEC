# TP MobSF - Analyse dynamique de DIVA

---

## 1. Introduction

**MobSF (Mobile Security Framework)** est un framework open source d'analyse de sécurité mobile capable d'effectuer des analyses statiques et dynamiques d'applications Android, iOS et Windows Phone. Il intègre des outils tels que Frida, un proxy HTTP/HTTPS, un moniteur de fichiers, un moniteur d'intents et un flux de logs en temps réel.

**DIVA (Damn Insecure and Vulnerable Android App)** est une application Android intentionnellement vulnérable, conçue à des fins pédagogiques. Elle regroupe un ensemble de challenges illustrant les vulnérabilités les plus communes dans le développement Android : stockage non sécurisé, journalisation sensible, contournement du contrôle d'accès, problèmes de validation d'entrée, etc.

L'objectif de ce TP est de mettre en pratique les techniques d'analyse dynamique d'applications Android en utilisant MobSF déployé dans un environnement virtualisé, afin d'observer les comportements de l'application à l'exécution et d'identifier ses vulnérabilités.

---

## 2. Objectifs du TP

- Réaliser une analyse statique de l'APK DIVA avec MobSF.
- Lancer une analyse dynamique sur un émulateur Android.
- Observer les logs runtime avec Logcat Stream.
- Surveiller les fichiers créés par l'application avec File Monitor.
- Observer les intents et composants Android avec Intent Monitor.
- Utiliser Frida pour instrumenter l'application dynamiquement.
- Configurer le proxy HTTP/HTTPS de MobSF pour intercepter le trafic réseau.
- Exporter un rapport d'analyse au format PDF ou JSON.

---

## 3. Architecture de l'environnement

L'architecture mise en place pour ce TP repose sur deux machines distinctes communicant en réseau local en mode **Bridged** :

```
Machine hote Windows
├── Android Studio / AVD Emulator (Android API 28, x86, sans Google Play)
├── ADB (Android Debug Bridge)
└── IP : 192.168.1.10

VM Mobexler
├── MobSF (deploye via Docker)
├── Platform-tools / ADB
└── IP : 192.168.1.59
```

**Flux de communication :**

```
[Emulateur Android] <--ADB--> [VM Mobexler / MobSF Docker]
                                        |
                               [Proxy HTTPS :1337]
                               [Frida Server (x86)]
                               [Interface Web MobSF]
```

L'émulateur Android est lancé sur la machine hôte Windows via Android Studio. La VM Mobexler, configurée en mode réseau Bridged, accède à l'émulateur via ADB sur le réseau local. MobSF est lancé dans un conteneur Docker sur Mobexler et expose son interface web sur le port 8000.

> **Note importante :** L'émulateur Android doit utiliser une image Android API <= 30 sans Google Play afin que le système de fichiers soit accessible en écriture, ce qui est nécessaire pour que Frida puisse s'injecter correctement dans l'application cible.

---

## 4. Preparation de l'environnement

Avant de démarrer l'analyse, plusieurs vérifications préliminaires ont été effectuées :

1. **Lancement de l'émulateur Android** depuis Android Studio (AVD Manager), avec une image compatible MobSF (Android 9 / API 28, architecture x86, sans Google Play Store).
2. **Connexion ADB** entre la VM Mobexler et l'émulateur en exécutant `adb connect <IP_hote>:5554` depuis Mobexler, puis vérification avec `adb devices`.
3. **Lancement de MobSF** via Docker sur la VM Mobexler :
   ```bash
   docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest
   ```
4. **Accès à l'interface MobSF** dans le navigateur à l'adresse `http://192.168.1.59:8000`.

---

## 5. Analyse statique de DIVA

L'APK `diva-beta.apk` a été téléversé dans l'interface MobSF via la page d'accueil. MobSF a automatiquement procédé à la décompilation et à l'analyse statique du fichier.

Les informations suivantes ont été détectées lors de l'analyse statique :

- **Nom de l'application :** Diva
- **Package :** `jakhar.aseem.diva`
- **Activité principale :** `jakhar.aseem.diva.MainActivity`
- **Target SDK :** 23 / Min SDK : 15
- **Version Android :** 1.0 (code 1)
- **Taille de l'APK :** 1,43 MB
- **Score de securite :** 36/100 — indiquant de nombreuses vulnerabilites
- **Trackers detectes :** 0/432 — aucune bibliotheque de tracking tieree

<p align="center">
   <img width="945" height="207" alt="image" src="https://github.com/user-attachments/assets/88511b4d-5452-4563-a7b3-df4583108142" />

</p>

L'analyse statique permet d'identifier a priori les composants dangereux de l'application (activites exportees, permissions sensibles, stockage non chiffre, communications reseau en clair, etc.) avant meme d'executer l'application. Elle constitue la premiere etape essentielle avant de passer a l'analyse dynamique.

---

## 6. Lancement de l'analyse dynamique

Une fois l'analyse statique completee, l'analyse dynamique a ete lancee depuis le tableau de bord MobSF en cliquant sur **"Start Dynamic Analysis"**.

### 6.1 Creation de l'environnement dynamique

MobSF a immediatement commence la preparation de l'environnement en affichant le message suivant dans la console Docker :

<p align="center">
  <img width="945" height="88" alt="image" src="https://github.com/user-attachments/assets/46edc95b-6df7-4aaf-93ce-54ecb47f1c74" />

</p>

> `[INFO] 02/May/2026 15:57:55 - Creating Dynamic Analysis Environment for jakhar.aseem.diva`

Ce message confirme que MobSF a detecte l'emulateur via ADB et demarre le processus de configuration.

### 6.2 Initialisation de Frida et du proxy

MobSF a ensuite procede a l'installation de tous les composants necessaires a l'analyse dynamique :

<p align="center">
  <img width="945" height="328" alt="image" src="https://github.com/user-attachments/assets/7fb21747-5867-4d5d-b9b4-a860a9f7a305" />

</p>

Les etapes visibles dans les logs sont les suivantes :

| Etape | Detail |
|-------|--------|
| Detection Android | Version 9.0 identifiee, architecture x86 |
| Telechargement Frida | `frida-server-16.4.8-android-x86` telecharge |
| Installation RootCA | Certificat racine MobSF installe pour l'interception HTTPS |
| MobSFying | Configuration systeme de l'emulateur terminee |
| Proxy HTTPS | Demarrage du proxy sur le port 1337 |
| ADB Reverse TCP | Redirection TCP activee sur le port 1337 |
| Proxy global VM | Proxy global configure pour l'emulateur Android |
| Environnement pret | `Testing Environment is Ready!` |

### 6.3 Interface Dynamic Analyzer et lancement de DIVA

Une fois l'environnement pret, l'interface du Dynamic Analyzer s'est ouverte dans le navigateur. DIVA a ete lancee sur l'emulateur et le challenge **"3. Insecure Data Storage - Part 1"** a ete ouvert :

<p align="center">
  <img width="945" height="412" alt="image" src="https://github.com/user-attachments/assets/dcbc8e9d-44e3-4064-98cd-3ff1857bcca2" />

</p>

L'interface montre simultanement :
- L'emulateur Android a gauche avec DIVA ouverte sur le challenge Insecure Data Storage Part 1
- Le panneau central **Dynamic Analyzer** avec les messages de statut confirmant que l'environnement est operationnel
- L'editeur **Frida Code Editor** a droite avec un template `Java.perform()` pret a l'emploi

<p align="center">
  <img src="screenshots/06_dynamic_analyzer_pret.png" width="700">
</p>

Le message `Environment is ready for Dynamic Analysis` confirme que tous les agents MobSF sont actifs et que le streaming d'ecran fonctionne.

---

## 7. Exploration de l'application DIVA

L'application DIVA a ete exploree via l'emulateur. L'ecran principal liste l'ensemble des challenges disponibles :

<p align="center">
  <img width="520" height="850" alt="image" src="https://github.com/user-attachments/assets/ce144d51-805c-41e2-8d25-87853d7e70a9" />

</p>

Les challenges couvrent les categories suivantes :

- **1. Insecure Logging** — journalisation de donnees sensibles
- **2. Hardcoding Issues** — donnees sensibles codees en dur
- **3 a 6. Insecure Data Storage** (Parts 1 a 4) — stockage non securise
- **7 et 8. Input Validation Issues** — vulnerabilites de validation
- **9 a 11. Access Control Issues** — controle d'acces defaillant

---

## 8. Test du challenge Insecure Data Storage

Le challenge **"3. Insecure Data Storage - Part 1"** a ete ouvert dans DIVA. L'objectif est d'identifier comment et ou l'application stocke les informations d'identification saisies par l'utilisateur.

L'interface presente deux champs de saisie :
- `Enter 3rd party service user name`
- `Enter 3rd party service password`

Apres avoir saisi des donnees de test et clique sur **SAVE**, les informations ont ete enregistrees par l'application. Ce challenge illustre le stockage de donnees sensibles dans les **SharedPreferences** Android en clair, un fichier XML accessible dans le repertoire prive de l'application (`/data/data/jakhar.aseem.diva/shared_prefs/`).

Le **File Monitor** de MobSF permet d'observer en temps reel les fichiers crees ou modifies lors de l'execution de l'application, revelant directement le chemin et le contenu des donnees stockees.

> **Risque de securite :** Le stockage de mots de passe ou de jetons d'authentification en clair dans les SharedPreferences, des bases SQLite non chiffrees ou des fichiers texte est une vulnerabilite critique. Toute application malveillante ayant acces root au terminal peut lire ces donnees.

---

## 9. Observation des logs runtime (Logcat Stream)

Le **Logcat Stream** de MobSF permet de visualiser en temps reel les logs systeme de l'emulateur Android pendant l'execution de DIVA. Ces logs sont collectes via ADB.

<p align="center">
  <img width="945" height="185" alt="image" src="https://github.com/user-attachments/assets/4eed06b7-d108-4740-ae22-8a84a2959b84" />

</p>

Les logs captures montrent :

- Le lancement de `jakhar.aseem.diva.MainActivity` avec intent `android.intent.action.MAIN`
- L'activite de l'`ActivityManager` gerant le cycle de vie de l'application
- Les operations `dex2oat` effectuees par Frida pour instrumenter les classes Java en memoire
- Les compilations de fichiers `.dex` de l'application dans le cache Frida (`/data/data/jakhar.aseem.diva/cache/frida...`)

Ces logs confirment que l'instrumentation dynamique par Frida est effective et que l'application est bien monitored. Pour confirmer une fuite de donnees via le challenge **Insecure Logging** (challenge 1), il faudrait filtrer les logs pour les tags specifiques emis par `LogActivity` et y rechercher les valeurs saisies en clair par l'utilisateur.

---

## 10. Utilisation de Frida

Frida est un framework d'instrumentation dynamique qui permet de hooker des methodes Java ou natives d'une application Android en cours d'execution, sans modifier l'APK. MobSF integre nativement plusieurs scripts Frida preconfigures.

### 10.1 Scripts Frida par defaut

<p align="center">
<img width="420" height="91" alt="image" src="https://github.com/user-attachments/assets/f7ba61db-dde6-4745-9d5c-9f8236781c10" />

</p>

Les scripts suivants ont ete actives dans l'interface Dynamic Analyzer :

| Script | Fonction |
|--------|----------|
| **API Monitoring** | Surveille les appels d'API sensibles (crypto, reseau, fichiers) |
| **SSL Pinning Bypass** | Contourne le certificate pinning pour intercepter le trafic HTTPS |
| **Root Detection Bypass** | Contourne les verifications de root dans l'application |
| **Debugger Check Bypass** | Desactive la detection de debogueur |
| **Clipboard Monitor** | Surveille le contenu du presse-papiers |

Des scripts auxiliaires sont egalement disponibles : **Enumerate Loaded Classes**, **Capture Strings**, **Capture String Comparisons**, **Enumerate Class Methods**, **Search Class Pattern**, **Trace Class Methods**.

### 10.2 Frida Code Editor

L'editeur de code Frida integre a MobSF permet d'ecrire des scripts personnalises utilisant l'API `Java.perform()` pour hooker directement des methodes de l'application :

```javascript
Java.perform(function() {
    // Use send() for logging
});
```

### 10.3 Logs Frida

<p align="center">
 

</p>

Les **Frida Logs** confirment le chargement de tous les scripts actives :

- `Loaded Frida Script - api_monitor`
- `Loaded Frida Script - root_bypass`
- `Loaded Frida Script - dump_clipboard`
- `Loaded Frida Script - debugger_check_bypass`
- `Loaded Frida Script - ssl_pinning_bypass`

Les tentatives de bypass SSL Pinning ont scanne de nombreuses bibliotheques tierces (`okhttp`, `okhttp3`, `DataTheorem trustkit`, `Appcelerator`, `Apache Cordova`, etc.) — aucune n'etant presente dans DIVA, ce qui est attendu. Les logs montrent egalement les chemins d'acces aux fichiers et au cache de l'application (`/data/user/0/jakhar.aseem.diva/files`, `/data/user/0/jakhar.aseem.diva/cache`, etc.).

### 10.4 API Monitor

<p align="center">
 <img width="945" height="223" alt="image" src="https://github.com/user-attachments/assets/977e4f54-1d65-4a58-b6d4-04c32688ee04" />

</p>

L'**API Monitor** de MobSF (rafraichi toutes les 10 secondes) capture les appels d'API effectues par l'application via les hooks Frida. Un appel a ete detecte :

| Classe | Methode | Arguments |
|--------|---------|-----------|
| `dalvik.system.DexClassLoader` | `$init` | Chemin vers les fichiers `.dex` du cache Frida |

Cet appel correspond au chargement des classes Frida dans le contexte de l'application, confirmant le bon fonctionnement de l'instrumentation.

---

## 11. Network Traffic et Proxy HTTPS

MobSF configure automatiquement un proxy HTTP/HTTPS sur le port **1337** de l'emulateur lors du lancement de l'analyse dynamique. Ce proxy permet d'intercepter l'integralite du trafic reseau de l'application testee.

La configuration du proxy est confirmee par les logs Docker :
- `Starting HTTPS Proxy on 1337`
- `Enabling ADB Reverse TCP on 1337`
- `Setting Global Proxy for Android VM`

DIVA etant une application d'apprentissage locale, elle ne genere pas de trafic reseau significatif lors des challenges testes (Insecure Logging, Insecure Data Storage). Cependant, le proxy MobSF est operationnel et pret a intercepter toute communication HTTP/HTTPS qu'une application cible effectuerait vers des serveurs externes, ce qui est particulierement utile pour detecter des transmissions de donnees non chiffrees ou des fuites d'informations sensibles vers des API tierces.

---

## 12. Intent Monitor et Access Control

Les **intents** sont le mecanisme de communication inter-composants d'Android. Un intent mal protege peut permettre a une application malveillante de lancer des composants (activites, services, broadcast receivers) qui ne devraient pas etre accessibles depuis l'exterieur.

L'**Intent Monitor** de MobSF capture en temps reel les intents emis et recus par l'application pendant l'analyse dynamique. Il est particulierement pertinent pour les challenges **Access Control Issues** de DIVA (challenges 9, 10 et 11), qui illustrent les risques des activites exportees accessibles sans permission.

Les composants Android declares comme `exported="true"` sans permission appropriee peuvent etre invoques directement via `adb shell am start` ou par toute application installee sur le terminal, constituant une faille de controle d'acces majeure.

---

## 13. Export du rapport

MobSF permet de generer un rapport complet de l'analyse en cliquant sur le bouton **"Generate Report"** depuis l'interface de l'analyse statique ou dynamique. Le rapport peut etre exporte aux formats :

- **PDF** : rapport lisible destine a la documentation et aux audits de securite
- **JSON** : format structuré exploitable par des outils d'integration continue ou des scripts d'automatisation

Le rapport inclut l'ensemble des resultats de l'analyse statique (permissions, composants, code suspect, certificats) et dynamique (API hookees, fichiers crees, logs, trafic reseau intercepte).

---

## 14. Resultats obtenus

| Element teste | Outil MobSF utilise | Resultat observe | Risque de securite |
|---------------|---------------------|------------------|--------------------|
| Analyse statique APK | Static Analyzer | Score securite 36/100, package `jakhar.aseem.diva`, SDK 23 | Nombreuses vulnerabilites identifiees avant execution |
| Insecure Logging | Logcat Stream | Logs runtime confirment l'activite de `MainActivity` et l'instrumentation Frida | Fuite potentielle de donnees sensibles dans les logs systeme |
| Insecure Data Storage | File Monitor | Challenge Part 1 ouvert, sauvegarde de credentials visible | Stockage en clair dans SharedPreferences accessible |
| Frida | Frida Code Editor + Scripts | 5 scripts charges (api_monitor, root_bypass, ssl_bypass, etc.) | Contournement des protections runtime de l'application |
| Runtime Logs | Logcat Stream | Operations dex2oat Frida visibles, cycle de vie DIVA monitore | Activite complete de l'application tracee en temps reel |
| API Monitor | Frida API Monitoring | Appel `DexClassLoader.$init` detecte | Chargement dynamique de classes surveille |
| Network Traffic | Proxy HTTPS :1337 | Proxy configure et operationnel (port 1337, ADB Reverse TCP) | Interception du trafic reseau chiffre possible |
| Intent Monitor | Intent Monitor | Disponible pour l'observation des composants exportes | Composants accessibles sans permission identifies |
| Rapport exportable | Generate Report | Rapport PDF/JSON disponible depuis le tableau de bord | Documentation complete pour audit de securite |

---

## 15. Difficultes rencontrees

La mise en place de l'environnement d'analyse dynamique avec MobSF presente plusieurs difficultes techniques :

**Compatibilite de l'emulateur :** MobSF exige un emulateur Android avec une image systeme sans Google Play Store afin que la partition `/system` soit montee en lecture-ecriture. Les images "Google APIs" ou "Google Play" ne sont pas compatibles car elles bloquent l'ecriture sur le systeme, ce qui empeche l'installation du serveur Frida et du certificat racine MobSF.

**Version de l'API Android :** L'analyse dynamique MobSF est limitee aux emulateurs Android API <= 30. Les versions plus recentes d'Android introduisent des restrictions de securite supplementaires qui bloquent certains mecanismes d'instrumentation de Frida.

**Connexion ADB entre Mobexler et l'emulateur :** La communication ADB entre la VM Mobexler et l'emulateur Windows peut etre perturbee par des configurations firewall ou des problemes de resolution IP. Le mode reseau Bridged est indispensable pour que les deux machines soient sur le meme sous-reseau.

**Telechargement du serveur Frida :** MobSF telecharge automatiquement le binaire `frida-server` compatible avec l'architecture de l'emulateur lors de la creation de l'environnement dynamique. Une connexion internet fonctionnelle depuis la VM Mobexler est donc requise.

**Latence du streaming d'ecran :** Le streaming de l'ecran de l'emulateur dans l'interface MobSF peut presenter des latences ou des interruptions selon les performances reseau et materielles de l'environnement.

---

## 16. Conclusion

Ce TP a permis de mettre en oeuvre un environnement complet d'analyse de securite mobile en combinant MobSF, Frida et un emulateur Android dans une architecture reseau virtualisee. L'analyse statique de l'APK DIVA a fourni une premiere cartographie des vulnerabilites de l'application, confirmee par un score de securite de 36/100, avant meme toute execution.

L'analyse dynamique a permis d'observer le comportement de l'application a l'execution : l'instrumentation par Frida a ete confirmee par le chargement des scripts (api_monitor, ssl_pinning_bypass, root_bypass, etc.), les Frida Logs et l'API Monitor ont trace les appels effectues par l'application, et le Logcat Stream a fourni une visibilite complete sur l'activite runtime de DIVA.

Les challenges Insecure Logging et Insecure Data Storage illustrent deux vulnerabilites tres repandues dans les applications Android reelles : la journalisation de donnees sensibles en clair dans les logs systeme, et le stockage non chiffre de credentials dans les SharedPreferences. Ces vulnerabilites sont facilement exploitables sur un terminal compromis ou dans un environnement de test.

MobSF se confirme comme un outil d'audit complet et accessible, permettant en une seule plateforme de couvrir l'analyse statique, l'instrumentation dynamique, l'interception reseau et la surveillance des composants Android. La maitrise de cet outil constitue une competence fondamentale pour tout professionnel de la securite mobile.

---

*Rapport genere dans le cadre d'un travail pratique de cybersecurité mobile Android.*
