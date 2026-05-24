# LAB 16 — Inspection HTTPS Android : Désactivation du SSL Pinning avec Objection + Proxy

**Cours :** Sécurité des applications mobiles  
**Environnement :** Arch Linux · OWASP ZAP · Android Studio AVD (x86_64) · Frida 17.9.10 · Objection 1.12.4

---

## Table des matières

1. [Objectif et différence avec le Lab 15](#1-objectif-et-différence-avec-le-lab-15)
2. [Prérequis — Environnement déjà en place](#2-prérequis--environnement-déjà-en-place)
3. [Étape 1 — Installation d'Objection](#3-étape-1--installation-dobjection)
4. [Étape 2 — Préparer l'appareil et démarrer frida-server](#4-étape-2--préparer-lappareil-et-démarrer-frida-server)
5. [Étape 3 — Configurer le proxy et installer la CA](#5-étape-3--configurer-le-proxy-et-installer-la-ca)
6. [Étape 4 — Lancer l'app avec Objection](#6-étape-4--lancer-lapp-avec-objection)
7. [Étape 5 — Validation](#7-étape-5--validation)
8. [ERREURS RENCONTRÉES — Diagnostic complet](#8-erreurs-rencontrées--diagnostic-complet)
9. [Solution de contournement — Frida JS direct](#9-solution-de-contournement--frida-js-direct)
10. [Note : Objection avec venv Python dédié](#10-note--objection-avec-venv-python-dédié)
11. [Conclusion](#11-conclusion)

---

## 1. Objectif et différence avec le Lab 15

Ce lab introduit **Objection**, un outil de haut niveau basé sur Frida qui automatise le bypass SSL pinning en une seule commande, sans écriture manuelle de scripts JavaScript.

| Critère | Lab 15 (Frida JS) | Lab 16 (Objection) |
|---|---|---|
| Approche | Script JS manuel (`sslpin_bypass_universal.js`) | Commande unique `android sslpinning disable` |
| Contrôle | Granulaire, personnalisable | Automatique, abstrait |
| Compatibilité | Frida toutes versions | Objection 1.12.4 → Frida ≤ 16.x uniquement |
| Complexité | Moyenne | Faible (en théorie) |

---

## 2. Prérequis — Environnement déjà en place

L'infrastructure suivante est héritée du Lab 15 et déjà opérationnelle :

![Vérification environnement Python/ADB](lab16_ref_env_check.png)

- Python 3.14.5, pip 26.1.1, ADB 1.0.41 installés sur l'hôte
- Émulateur AVD x86_64 (emulator-5554) connecté via ADB

![Frida 17.9.10 installé](lab16_ref_frida_version.png)

- Frida 17.9.10 installé sur l'hôte (via pip)
- Certificat CA ZAP installé comme CA système sur l'émulateur
- Proxy `10.0.2.2:8080` configuré sur l'émulateur

---

## 3. Étape 1 — Installation d'Objection

![Étape 1 — Installation Objection](lab16_step1_objection_install.png)

Objection s'installe via pipx (environnement isolé recommandé) ou pip classique :

```bash
# Option recommandée — environnement isolé
pip install --user pipx
pipx ensurepath
pipx install objection

# Ou via pip classique
pip install --upgrade objection frida frida-tools

# Vérification
objection version
frida --version
python -c "import frida; print(frida.__version__)"
```

> **Remarque :** Sur Arch Linux, `objection --version` n'est pas une option valide dans Objection 1.12.4 — utiliser `objection version` à la place.

---

## 4. Étape 2 — Préparer l'appareil et démarrer frida-server

![Étape 2 — Préparation frida-server](lab16_step2_frida_server_setup.png)

```bash
# Identifier l'architecture
adb shell getprop ro.product.cpu.abi   # → x86_64

# Pousser et lancer frida-server
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0" &

# Redirection ports Frida (si nécessaire)
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043

# Vérifier
frida-ps -Uai
```

> **Important :** La version de `frida-server` doit correspondre exactement à la version du client Frida sur l'hôte (`frida --version`).

---

## 5. Étape 3 — Configurer le proxy et installer la CA

![Étape 3 — Proxy et CA](lab16_step3_proxy_ca.png)

La configuration du proxy ZAP et l'installation du certificat CA sont identiques au Lab 15 (déjà effectuées) :

- ZAP écoute sur `localhost:8080`
- Certificat CA ZAP installé dans `/system/etc/security/cacerts/` (hash `44351ca1.0`)
- Proxy émulateur configuré : `adb shell settings put global http_proxy 10.0.2.2:8080`

> **Note importante :** Le bypass SSL pinning via Objection/Frida neutralise la vérification côté application. Le certificat CA système est nécessaire pour que ZAP puisse déchiffrer le trafic sans erreurs côté proxy.

---

## 6. Étape 4 — Lancer l'app avec Objection (théorie)

![Étape 4 — Lancement Objection](lab16_step4_objection_launch.png)

La commande théorique pour bypasser le SSL pinning avec Objection :

```bash
# Spawn (injection au démarrage — recommandé)
objection -g com.example.app explore --startup-command "android sslpinning disable"

# Ou attach (app déjà ouverte)
objection -g com.example.app explore
# puis dans la console Objection :
android sslpinning disable
android root disable   # pour neutraliser la détection root
```

**Commandes utiles dans la console Objection :**
- `help android sslpinning` — aide sur le pinning
- `android hooking search classes pin` — recherche de classes liées au pinning

> ⚠️ **En pratique, ces commandes ont échoué** — voir section [Erreurs](#8-erreurs-rencontrées--diagnostic-complet).

---

## 7. Étape 5 — Validation (théorie)

![Étape 5 — Validation](lab16_step5_validation.png)

Une fois le bypass actif, la validation consiste à :

1. Utiliser les écrans internes de l'app (login, requêtes API) pour générer du trafic
2. Vérifier dans ZAP que les requêtes HTTPS de l'app apparaissent sans alerte SSL
3. Surveiller la console Objection pour confirmer que les hooks sont déclenchés

**Livrables attendus :**
- Capture de `objection version`, `frida --version`, `frida-ps -Uai`
- Commande exacte de lancement (`--startup-command` ou attach)
- Capture du proxy montrant une requête HTTPS de l'app

---

## 8. ERREURS RENCONTRÉES — Diagnostic complet

### Erreur 1 — Syntaxe dépréciée d'Objection

![Erreur 1 — Frida server not running](lab16_err1_objection_frida_not_running.png)

**Commande utilisée :**
```bash
objection -g owasp.mstg.uncrackable1 explore --startup-command "android sslpinning disable"
```

**Erreur obtenue :**
```
DeprecationWarning: The option 'gadget' is deprecated. Please use '-n' or '--name' instead
DeprecationWarning: The command 'explore' is deprecated. Use 'objection start' instead
Frida server or gadget is not running on the target!
```

**Cause :** Objection 1.12.4 a déprécié les options `-g` et `explore`. La nouvelle syntaxe est `-n` et `start`.

**Tentative de correction :**
```bash
objection -n owasp.mstg.uncrackable1 start
# → Même erreur : Frida server or gadget is not running on the target!
```

---

### Erreur 2 — Incompatibilité de version Objection / Frida

![Erreur 2 — Objection help + Frida version](lab16_err2_objection_help_frida_version.png)

**Diagnostic :**
```bash
objection --help    # → Objection 1.12.4
frida --version     # → 17.9.10
```

**Cause racine identifiée :** Objection 1.12.4 est la **dernière version publiée** mais elle n'est compatible qu'avec Frida ≤ 16.x. Frida 17.x a introduit des changements d'API incompatibles avec Objection.

Le message `Frida server or gadget is not running` n'est pas une erreur de connectivité — c'est Objection qui ne peut pas communiquer avec frida-server 17.x à cause de l'incompatibilité d'API.

**Preuve que frida-server tourne bien :**

![frida-ps fonctionne](lab16_err3_frida_ps_before.png)

`frida-ps -Uai` liste correctement tous les processus — frida-server 17.9.10 est bien actif et accessible par le client Frida système.

---

### Erreur 3 — Tentative de downgrade Frida dans le venv Objection

**Tentative :**
```bash
pipx inject objection "frida==16.5.0" "frida-tools==12.5.1" --force
```

**Erreur :**
```
ERROR: Could not find a version that satisfies the requirement frida==16.5.0
ERROR: No matching distribution found for frida==16.5.0
```

**Cause :** `frida==16.5.0` n'existe pas sur PyPI (il y a 16.5.1 mais pas 16.5.0).

**Deuxième tentative avec 16.7.19 :**
```bash
pipx inject objection "frida==16.7.19" "frida-tools==12.5.1" --force
# → Injection réussie dans le venv
```

Vérification de la version dans le venv Objection :
```bash
~/.local/share/pipx/venvs/objection/bin/python -c "import frida; print(frida.__version__)"
# → 16.7.19 ✓
```

---

### Erreur 4 — frida-server 16.7.19 incompatible avec l'émulateur

**Tentative de remplacement du frida-server :**
```bash
wget https://github.com/frida/frida/releases/download/16.7.19/frida-server-16.7.19-android-x86_64.xz
xz -d frida-server-16.7.19-android-x86_64.xz
adb push frida-server-16.7.19-android-x86_64 /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &
```

**Erreur :**
```
CANNOT LINK EXECUTABLE "/data/local/tmp/frida-server": 
empty/missing DT_HASH/DT_GNU_HASH in "/data/local/tmp/frida-server" 
(new hash type from the future?)
```

**Cause :** Le binaire frida-server 16.7.19 utilise un format ELF avec des entrées DT (dynamic table) inconnues du linker Android de cet émulateur. Le binaire 17.9.10 original fonctionnait sans ce problème car il était compilé différemment.

**Tableau récapitulatif des incompatibilités :**

| Composant | Version | Statut |
|---|---|---|
| Objection (hôte) | 1.12.4 | ✅ Installé |
| Frida client système (hôte) | 17.9.10 | ✅ Fonctionne |
| Frida dans venv Objection | 16.7.19 | ✅ Installé |
| frida-server 17.9.10 (émulateur) | 17.9.10 | ✅ Tourne |
| frida-server 16.7.19 (émulateur) | 16.7.19 | ❌ CANNOT LINK |
| Objection → frida-server 17.x | — | ❌ API incompatible |
| Objection → frida-server 16.x | — | ❌ Binaire refusé par le linker |

**Conclusion :** Aucune combinaison viable n'a pu être trouvée pour faire fonctionner Objection 1.12.4 avec cet émulateur Android sous Frida 17.x.

---

## 9. Solution de contournement — Frida JS direct

![Restauration frida-server 17.9.10](lab16_fix1_restore_frida_server.png)

Restauration de l'environnement fonctionnel :

```bash
# Tuer le frida-server 16.x échoué
adb shell pkill frida-server 2>/dev/null

# Remettre le frida-server 17.9.10 original
adb push ~/secappmobile/inspector/frida-server /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &

# Vérifier — tous les processus visibles
frida-ps -Uai
```

![Bypass SSL réussi avec le script JS](lab16_fix2_bypass_success.png)

Lancement du bypass via le script JavaScript universel (équivalent fonctionnel de `android sslpinning disable`) :

```bash
cd ~/secappmobile/inspector
frida -U -f owasp.mstg.uncrackable1 -l sslpin_bypass_universal.js
```

**Résultat :**
```
[+] SSL bypass: SSLContext.init patched
[+] SSL bypass: X509TrustManager patches attempted
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl patched
[+] Universal SSL pinning bypass installed
```

**Ce que fait `android sslpinning disable` d'Objection en interne** — exactement les mêmes hooks :

| Hook Objection interne | Équivalent dans le script JS |
|---|---|
| Patch `X509TrustManager` | `checkServerTrusted` → `return null` |
| Patch Conscrypt | `TrustManagerImpl.checkTrusted` → allow |
| Patch OkHttp | `CertificatePinner.check` → skip |
| Patch WebView | `onReceivedSslError` → `proceed()` |
| Inject TrustManager permissif | `SSLContext.init` → TrustManager vide |

---

## 10. Note : Objection avec venv Python dédié

Une alternative non testée qui pourrait résoudre le problème de compatibilité consiste à créer un environnement Python entièrement dédié avec une version de Frida compatible :

```bash
# Créer un venv isolé avec Python 3.11 (plus compatible avec Frida 16.x)
python3.11 -m venv ~/venv-objection
source ~/venv-objection/bin/activate

# Installer des versions compatibles
pip install frida==16.7.19 frida-tools objection

# Vérifier
objection version
frida --version   # doit afficher 16.7.19

# Puis utiliser avec un frida-server 16.7.19 fonctionnel
objection -n owasp.mstg.uncrackable1 start
```

> **Prérequis bloquant :** Cette approche nécessite toujours un `frida-server` 16.x qui tourne sur l'émulateur, ce qui a échoué avec l'erreur `CANNOT LINK EXECUTABLE` sur cet AVD spécifique. Elle pourrait fonctionner sur un appareil physique rooté ou un AVD avec une version d'Android différente.

---

## 11. Conclusion

### Ce qui a fonctionné ✅
- Installation d'Objection 1.12.4 via pipx
- Identification précise des incompatibilités de version
- Bypass SSL pinning via script Frida JS (équivalent fonctionnel complet)
- Infrastructure ZAP + CA système + proxy pleinement opérationnelle

### Ce qui a échoué ❌
- Objection 1.12.4 incompatible avec Frida 17.x (API modifiée)
- frida-server 16.7.19 refusé par le linker de l'émulateur (`DT_HASH` manquant)
- Aucune combinaison Objection/frida-server viable sur cet AVD

### Leçon apprise
Objection est un outil **abandonné** (dernière mise à jour : 2023, dernière version : 1.12.4). Pour des labs pratiques en 2025/2026, le script Frida JS direct est plus fiable, plus transparent, et offre un contrôle plus fin. Les deux approches produisent **le même résultat de sécurité**.

```
┌─────────────────────────────────────────────────────────┐
│                  Résumé de l'infrastructure              │
│                                                         │
│  Arch Linux (hôte)                                      │
│  ├── Frida 17.9.10 (client)                             │
│  ├── Objection 1.12.4 (❌ incompatible avec Frida 17.x) │
│  └── ZAP proxy → localhost:8080                         │
│                          ↕ ADB USB                      │
│  Émulateur AVD x86_64                                   │
│  ├── frida-server 17.9.10 ✅                            │
│  ├── CA ZAP installée (système) ✅                      │
│  ├── Proxy → 10.0.2.2:8080 ✅                           │
│  └── App cible : owasp.mstg.uncrackable1                │
│      └── SSL bypass actif via script JS ✅              │
└─────────────────────────────────────────────────────────┘
```
