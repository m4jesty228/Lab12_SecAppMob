# Lab 12 — Bypass de la Détection de Root Android : Medusa & Frida

**Auteur :** DOSSAH Yao Landry  
**Filière :** Génie CyberDefense et Systèmes de Télécommunications Embarquées (GCDSTE)  
**Établissement :** ENSA Marrakech

---

## Contexte pédagogique

Ce laboratoire porte sur le contournement des mécanismes de détection de root dans une application Android, à des fins d'analyse de sécurité défensive. L'objectif est de comprendre comment ces détections fonctionnent au niveau Java et natif, puis de les neutraliser via instrumentation dynamique avec **Medusa** (basé sur Frida) ou directement avec des scripts **Frida**. Ce type de technique est utilisé dans les audits MASVS pour valider la robustesse des mécanismes de résilience (`MASVS-RESILIENCE`).

> **Cadre éthique :** ces techniques sont appliquées uniquement sur une application de test autorisée dans le cadre du cours. Aucune application tierce non autorisée n'est ciblée.

---

## Environnement

| Composant | Détail |
|-----------|--------|
| OS hôte | Windows / Linux / macOS |
| Python | 3.8+ |
| Frida | 16.x (frida + frida-tools via pip) |
| ADB | Android Platform Tools |
| Appareil | Android 8.0+ (options développeur + débogage USB activés) |
| Outil principal | Medusa (instrumentation Android basée sur Frida) |
| Fallback | Scripts Frida purs (`bypass_root.js`, `bypass_native.js`) |
| Application cible | `com.example.rootcheck` (app de test autorisée) |

---

## Architecture du bypass

```
┌─────────────────────────────────────────────────────┐
│                  PC hôte                             │
│                                                      │
│  Medusa / frida-tools                                │
│       │                                              │
│       │ USB (ADB + Frida protocol)                   │
└───────┼──────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│              Appareil Android (rooté)                │
│                                                      │
│  frida-server (/data/local/tmp/)                     │
│       │                                              │
│       │ injection dans le processus cible            │
│       ▼                                              │
│  com.example.rootcheck                               │
│       │                                              │
│       ├── Java hooks                                 │
│       │    ├── Build.TAGS → "release-keys"           │
│       │    ├── File.exists() → false (chemins su)    │
│       │    ├── Runtime.exec() → bloqué si "su"       │
│       │    └── RootBeer.isRooted() → false           │
│       │                                              │
│       └── Native hooks (optionnel)                   │
│            ├── open() → bloqué sur /system/bin/su    │
│            ├── access() → bloqué                     │
│            └── stat() / lstat() → bloqué             │
└─────────────────────────────────────────────────────┘
```

---

## Comment fonctionne la détection de root

Les applications implémentent généralement deux couches de vérification :

**Couche Java :**
- Lecture de `android.os.Build.TAGS` — si la valeur est `test-keys`, l'appareil est considéré rooté.
- Appels `java.io.File.exists()` sur des chemins suspects (`/system/xbin/su`, `/system/bin/busybox`, etc.).
- Exécution de `Runtime.getRuntime().exec("su")` pour tester si `su` est accessible.
- Bibliothèques tierces comme **RootBeer** qui combinent plusieurs de ces vérifications.

**Couche native (C/C++) :**
- Syscalls `open`, `openat`, `access`, `stat`, `lstat` sur les mêmes chemins suspects.
- Lecture de `/proc/mounts` pour détecter des partitions montées en lecture/écriture.

**Principe du bypass :** hooker ces appels via Frida pour intercepter leurs arguments et retourner des réponses simulant un environnement non rooté — sans modifier l'APK.

---

## Étape 1 — Préparation de l'environnement

**Installation de Frida côté PC :**
```bash
pip install --upgrade frida frida-tools
frida --version
```

**Vérification de l'appareil :**
```bash
adb devices
# Résultat attendu : emulator-5554   device (ou l'identifiant réel)
```

**Identification de l'architecture CPU :**
```bash
adb shell getprop ro.product.cpu.abi
# Résultat : arm64-v8a / x86_64 selon l'appareil
```

**Déploiement de frida-server sur l'appareil :**
```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0 &"
```

**Vérification :**
```bash
frida-ps -Uai
# Liste toutes les applications installées sur l'appareil
```

**Résultat observé :** `frida-ps -Uai` retourne la liste des applications avec leurs PIDs. L'application cible `com.example.rootcheck` est visible dans la liste.

---

## Étape 2 — Installation de Medusa

```bash
git clone <URL_depot_Medusa>
cd Medusa
pip install -r requirements.txt
python medusa.py --help
```

Medusa expose des modules prêts à l'emploi pour les bypasses courants. Le module pertinent pour ce lab est généralement nommé `root-bypass`, `anti-root` ou `rootcloak` selon la version.

---

## Étape 3 — Bypass avec Medusa

**Lancement de l'application avec injection du module root-bypass dès le démarrage :**
```bash
python medusa.py --usb --spawn com.example.rootcheck --module root-bypass
```

**Alternative — attachement à un processus déjà lancé :**
```bash
medusa --usb --attach "com.example.rootcheck" --module root-bypass
```

**Logs observés dans la console Medusa :**
```
[+] Build.TAGS -> release-keys
[+] RootBeer.isRooted -> false
[+] File.exists bypass for /system/xbin/su
[+] File.exists bypass for /system/bin/busybox
[+] Runtime.exec hooks installed
[+] Java bypass installed
```

**Résultat observé dans l'application :** les vérifications qui affichaient "Root detected" passent à "Not rooted". Les fonctionnalités précédemment bloquées deviennent accessibles.

---

## Plan B — Frida pur (si Medusa indisponible)

Deux scripts couvrent les deux couches de détection.

**`bypass_root.js`** — hooks Java (Build.TAGS, File.exists, Runtime.exec, RootBeer) :
```bash
frida -U -f com.example.rootcheck -l bypass_root.js --no-pause
```

**`bypass_root.js` + `bypass_native.js`** — hooks Java et natifs combinés :
```bash
frida -U -f com.example.rootcheck -l bypass_root.js -l bypass_native.js --no-pause
```

`bypass_native.js` intercepte les syscalls `open`, `openat`, `access`, `stat`, `lstat` au niveau natif et retourne `-1` (erreur) pour tout accès aux chemins suspects (`/system/bin/su`, `/system/xbin/busybox`, etc.).

**Logs Frida observés :**
```
[+] Build.TAGS -> release-keys
[+] RootBeer.isRooted -> false
[+] File.exists bypass for /system/xbin/su
[+] Blocked open on /system/bin/su
[+] Blocked access on /system/xbin/busybox
[+] Runtime.exec hooks installed
[+] Java bypass installed
```

---

## Résultats de validation

| Test | Sans bypass | Avec bypass |
|------|-------------|-------------|
| `Build.TAGS` | `test-keys` → root détecté | `release-keys` → OK |
| `File.exists(/system/xbin/su)` | `true` → root détecté | `false` → OK |
| `RootBeer.isRooted()` | `true` → root détecté | `false` → OK |
| `Runtime.exec("su")` | succès → root détecté | bloqué → OK |
| `open(/system/bin/su)` (natif) | succès → root détecté | `-1` (ENOENT) → OK |
| Accès à l'application | bloqué | débloqué |

---

## Points clés retenus

- La détection de root opère sur deux couches indépendantes — Java et native. Un bypass incomplet qui ne couvre que la couche Java laisse les checks natifs actifs, ce qui suffit à certaines applications pour bloquer l'accès.
- Frida injecte du code JavaScript dans le processus à l'exécution sans modifier l'APK — c'est un bypass purement en mémoire, invisible à une analyse statique.
- `--spawn` injecte les hooks **avant** le démarrage du code applicatif — indispensable pour les applications qui effectuent leurs vérifications dans `Application.onCreate()` ou dans des constructeurs statiques.
- `--attach` est utilisé quand `--spawn` provoque un crash dû à une injection trop précoce.
- La résistance à ces bypasses (`MASVS-RESILIENCE`) repose sur l'obfuscation, les checks natifs multiples, et l'intégrité de l'environnement d'exécution — un bypass Frida reste détectable par une application qui vérifie la présence de frida-server lui-même.

---

*Lab réalisé dans le cadre du cours Sécurité des Applications Mobiles — ENSA Marrakech, Filière GCDSTE*
