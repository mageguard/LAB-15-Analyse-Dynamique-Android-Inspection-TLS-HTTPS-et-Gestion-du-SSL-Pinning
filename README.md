# LAB-15-Analyse-Dynamique-Android-Inspection-TLS-HTTPS-et-Gestion-du-SSL-Pinning
# 🔬 LAB 15 — Analyse Dynamique Android : Inspection TLS/HTTPS & SSL Pinning Bypass

> **Niveau :** Avancé  
> **Durée estimée :** 4 à 8 heures  
> **Outils :** Frida · Burp Suite Community / mitmproxy · ADB · Android Studio

---

## ⚠️ Avertissement éthique

> Ces techniques doivent être utilisées **uniquement dans un cadre légal** :
> tests sur vos propres applications/appareils, formation, ou audit de sécurité explicitement autorisé.  
> Le contournement de l'SSL pinning est réservé à l'**inspection de trafic à des fins de sécurité**.  
> **Ne jamais exposer des données réelles ni déployer ces méthodes en production.**

---

## 🎯 Objectifs du lab

À la fin de ce lab, tu sauras :

- ✅ **Installer et vérifier Frida** côté PC et `frida-server` sur Android
- ✅ **Mettre en place un proxy** (Burp Suite ou mitmproxy) avec certificat CA sur l'appareil
- ✅ **Neutraliser l'SSL pinning via hooks Java** (TrustManager / Conscrypt / OkHttp / WebView)
- ✅ **Diagnostiquer et compléter le bypass** si le pinning est natif (BoringSSL / OpenSSL)
- ✅ **Valider le bypass** en capturant du trafic HTTPS déchiffré dans le proxy

---

## 🧠 Concepts clés

### Qu'est-ce que le SSL Pinning ?

Le **SSL Pinning** (ou Certificate Pinning) est un mécanisme de sécurité où l'application embarque en dur le certificat ou la clé publique du serveur attendu. Même avec un certificat CA valide (comme celui d'un proxy), l'app refuse la connexion si le certificat ne correspond pas au pin.

```
Sans pinning :  App → [CA proxy valide] → OK → trafic intercepté ✅
Avec pinning :  App → [CA proxy valide] → ❌ (pin ne correspond pas) → connexion refusée
```

### Couches de pinning possibles

| Couche | Technologie | Difficulté de bypass |
|---|---|---|
| Java standard | `TrustManager` / `HostnameVerifier` | ⭐ Facile |
| OkHttp | `CertificatePinner` | ⭐⭐ Moyen |
| Conscrypt / WebView | Interne Android | ⭐⭐ Moyen |
| Natif | BoringSSL / OpenSSL (`.so`) | ⭐⭐⭐ Difficile |

---

## 🛠️ Outils utilisés

| Outil | Rôle | Lien |
|---|---|---|
| **Frida** | Framework de hooking dynamique | [frida.re](https://frida.re) |
| **frida-server** | Daemon Frida côté Android | [Releases GitHub](https://github.com/frida/frida/releases) |
| **Burp Suite Community** | Proxy d'interception HTTPS | [portswigger.net](https://portswigger.net/burp/communitydownload) |
| **mitmproxy** *(alternative)* | Proxy open source en CLI | [mitmproxy.org](https://mitmproxy.org) |
| **objection** | Wrapper Frida automatisé | [github.com/sensepost/objection](https://github.com/sensepost/objection) |
| **Android Studio** | Émulateur + ADB | [developer.android.com](https://developer.android.com/studio) |
| **apktool / jadx** | Analyse statique de l'APK | Voir Lab 14 |

---

## 📁 Structure du projet

```
lab15-ssl-pinning/
│
├── README.md                        ← Ce fichier
│
├── setup/
│   ├── frida-server-android         ← Binaire frida-server pour l'architecture cible
│   └── burp-certificate.der         ← Certificat CA Burp exporté
│
├── scripts/
│   ├── bypass_trustmanager.js       ← Hook Java TrustManager
│   ├── bypass_okhttp.js             ← Hook OkHttp CertificatePinner
│   ├── bypass_conscrypt.js          ← Hook Conscrypt/WebView
│   ├── bypass_native.js             ← Hook BoringSSL natif
│   └── universal_bypass.js          ← Script combiné (recommandé pour démarrer)
│
├── captures/
│   └── traffic_decrypted.pcap       ← Captures de trafic déchiffré (Burp / mitmproxy)
│
└── writeup/
    └── findings.md                  ← Notes d'analyse et résultats
```

---

## 🚀 Mise en place de l'environnement

### Étape 1 — Installation de Frida côté PC

```bash
# Installer Frida via pip
pip install frida-tools

# Vérifier l'installation
frida --version
frida-ls-devices
```

### Étape 2 — Déploiement de frida-server sur Android

```bash
# 1. Identifier l'architecture de l'émulateur/appareil
adb shell getprop ro.product.cpu.abi
# Résultat probable : x86_64, arm64-v8a, x86, armeabi-v7a

# 2. Télécharger frida-server correspondant depuis :
# https://github.com/frida/frida/releases
# Ex : frida-server-16.x.x-android-x86_64.xz

# 3. Décompresser et pousser sur l'appareil
unxz frida-server-16.x.x-android-x86_64.xz
adb push frida-server-16.x.x-android-x86_64 /data/local/tmp/frida-server

# 4. Donner les permissions et lancer
adb shell chmod 755 /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &

# 5. Vérifier que Frida communique avec l'appareil
frida-ps -U
# → Doit lister les processus Android en cours
```

### Étape 3 — Configuration du proxy Burp Suite

#### 3a. Configurer l'écouteur Burp

Dans Burp Suite → **Proxy** → **Proxy settings** → **Add** :
- Bind to port : `8080`
- Bind to address : **All interfaces**

#### 3b. Installer le certificat CA Burp sur Android

```bash
# Exporter le certificat depuis Burp :
# Proxy → Proxy settings → Import/Export CA Certificate → Certificate in DER format
# Sauvegarder sous : setup/burp-certificate.der

# Convertir en PEM
openssl x509 -inform DER -in setup/burp-certificate.der -out burp-certificate.pem

# Pour Android 7+ (API 24+), le cert doit aller dans le trust store système
# Sur émulateur rooted :
adb push burp-certificate.pem /sdcard/
adb shell
su
cp /sdcard/burp-certificate.pem /system/etc/security/cacerts/$(openssl x509 -inform PEM -subject_hash_old -in /sdcard/burp-certificate.pem | head -1).0
chmod 644 /system/etc/security/cacerts/<hash>.0
```

#### 3c. Configurer le proxy Wi-Fi sur l'émulateur

Dans Android → **Paramètres** → **Wi-Fi** → Modifier le réseau → **Proxy manuel** :
- Hôte proxy : `10.0.2.2` *(IP hôte depuis l'émulateur)*
- Port : `8080`

---

## 📖 Modules du Lab

---

### Module 1 — Bypass TrustManager Java

Le contournement le plus simple : remplacer le `TrustManager` de l'application par un qui accepte tous les certificats.

**Script : `scripts/bypass_trustmanager.js`**

```javascript
Java.perform(function () {

  // Bypass X509TrustManager
  var TrustManager = Java.registerClass({
    name: 'com.lab15.bypass.TrustManager',
    implements: [Java.use('javax.net.ssl.X509TrustManager')],
    methods: {
      checkClientTrusted: function (chain, authType) {},
      checkServerTrusted: function (chain, authType) {},
      getAcceptedIssuers:  function () { return []; }
    }
  });

  // Remplacer le SSLContext de l'app
  var SSLContext = Java.use('javax.net.ssl.SSLContext');
  var TrustManagerArray = Java.array('javax.net.ssl.TrustManager', [TrustManager.$new()]);

  SSLContext.init.overload(
    '[Ljavax.net.ssl.KeyManager;',
    '[Ljavax.net.ssl.TrustManager;',
    'java.security.SecureRandom'
  ).implementation = function (km, tm, sr) {
    this.init(km, TrustManagerArray, sr);
  };

  // Bypass HostnameVerifier
  var HostnameVerifier = Java.use('javax.net.ssl.HttpsURLConnection');
  HostnameVerifier.setDefaultHostnameVerifier.implementation = function (verifier) {
    var bypassVerifier = Java.registerClass({
      name: 'com.lab15.bypass.HostnameVerifier',
      implements: [Java.use('javax.net.ssl.HostnameVerifier')],
      methods: {
        verify: function (hostname, session) { return true; }
      }
    });
    this.setDefaultHostnameVerifier(bypassVerifier.$new());
  };

  console.log("[+] TrustManager bypass actif");
});
```

```bash
# Lancer le hook
frida -U -f com.target.app -l scripts/bypass_trustmanager.js --no-pause
```

---

### Module 2 — Bypass OkHttp CertificatePinner

OkHttp implémente son propre mécanisme de pinning via `CertificatePinner`. Il faut hook directement sa méthode de vérification.

**Script : `scripts/bypass_okhttp.js`**

```javascript
Java.perform(function () {

  // OkHttp3 - CertificatePinner
  try {
    var CertificatePinner = Java.use('okhttp3.CertificatePinner');
    CertificatePinner.check.overload('java.lang.String', 'java.util.List')
      .implementation = function (hostname, peerCertificates) {
        console.log('[+] OkHttp3 CertificatePinner.check() bypassed pour : ' + hostname);
        return;
      };

    // Variante pour certaines versions d'OkHttp
    CertificatePinner.check.overload('java.lang.String', '[Ljava.security.cert.Certificate;')
      .implementation = function (hostname, certs) {
        console.log('[+] OkHttp3 CertificatePinner.check() (v2) bypassed pour : ' + hostname);
        return;
      };

    console.log('[+] OkHttp3 bypass actif');
  } catch (e) {
    console.log('[-] OkHttp3 non trouvé : ' + e);
  }

  // OkHttp2 (anciennes apps)
  try {
    var OkHttp2Pinner = Java.use('com.squareup.okhttp.CertificatePinner');
    OkHttp2Pinner.check.overload('java.lang.String', '[Ljava.security.cert.Certificate;')
      .implementation = function (hostname, certs) {
        console.log('[+] OkHttp2 bypass pour : ' + hostname);
        return;
      };
    console.log('[+] OkHttp2 bypass actif');
  } catch (e) {
    console.log('[-] OkHttp2 non trouvé : ' + e);
  }
});
```

---

### Module 3 — Bypass Conscrypt / WebView

Android utilise Conscrypt comme provider TLS par défaut depuis API 29. Les WebViews ont leur propre pile TLS.

**Script : `scripts/bypass_conscrypt.js`**

```javascript
Java.perform(function () {

  // Conscrypt TrustManager
  try {
    var ConscryptTM = Java.use('com.android.org.conscrypt.TrustManagerImpl');
    ConscryptTM.verifyChain.implementation = function (
        untrustedChain, trustAnchorChain, host, clientAuth, ocspData, tlsSctData) {
      console.log('[+] Conscrypt verifyChain bypassed pour : ' + host);
      return untrustedChain;
    };
    console.log('[+] Conscrypt bypass actif');
  } catch (e) {
    console.log('[-] Conscrypt non trouvé : ' + e);
  }

  // NetworkSecurityConfig (Android 7+)
  try {
    var NetworkSecurityTM = Java.use('android.security.net.config.RootTrustManager');
    NetworkSecurityTM.checkServerTrusted.implementation = function (chain, authType, hostname) {
      console.log('[+] NetworkSecurityConfig bypassed pour : ' + hostname);
      return;
    };
    console.log('[+] NetworkSecurityConfig bypass actif');
  } catch (e) {
    console.log('[-] NetworkSecurityConfig non trouvé : ' + e);
  }
});
```

---

### Module 4 — Bypass Natif (BoringSSL / OpenSSL)

Si le pinning est implémenté dans une librairie native `.so`, les hooks Java ne suffisent plus. Il faut travailler au niveau des fonctions C/C++.

#### 4a. Identifier la fonction cible dans Ghidra

Chercher dans le `.so` les fonctions :
- `SSL_CTX_set_verify`
- `SSL_get_verify_result`
- `X509_verify_cert`
- Fonctions custom contenant `pin`, `cert`, `fingerprint`

#### 4b. Hook natif avec Frida Interceptor

**Script : `scripts/bypass_native.js`**

```javascript
// Bypass SSL_get_verify_result (retourner X509_V_OK = 0)
var sslGetVerifyResult = Module.findExportByName('libssl.so', 'SSL_get_verify_result');
if (sslGetVerifyResult) {
  Interceptor.attach(sslGetVerifyResult, {
    onLeave: function (retval) {
      retval.replace(0); // X509_V_OK
      console.log('[+] SSL_get_verify_result → forcé à 0 (OK)');
    }
  });
}

// Bypass X509_verify_cert
var x509VerifyCert = Module.findExportByName('libssl.so', 'X509_verify_cert');
if (x509VerifyCert) {
  Interceptor.attach(x509VerifyCert, {
    onLeave: function (retval) {
      retval.replace(1); // 1 = succès
      console.log('[+] X509_verify_cert → forcé à 1 (succès)');
    }
  });
}

// Chercher dans les .so custom de l'app
Process.enumerateModules().forEach(function (module) {
  if (module.name.startsWith('lib') && module.name.endsWith('.so')) {
    console.log('[*] Module natif détecté : ' + module.name);
  }
});
```

---

### Module 5 — Script universel (point de départ recommandé)

Pour démarrer rapidement, utilise **objection** qui combine tous les bypasses :

```bash
# Installer objection
pip install objection

# Lancer le bypass SSL universel
objection -g com.target.app explore
# Puis dans la console objection :
android sslpinning disable
```

Ou directement avec le script communautaire `frida-multiple-unpinning` :

```bash
# Télécharger le script universel
wget https://raw.githubusercontent.com/hluwa/frida-dexdump/main/ssl_bypass.js -O scripts/universal_bypass.js

frida -U -f com.target.app -l scripts/universal_bypass.js --no-pause
```

---

### Module 6 — Validation : Capturer le trafic HTTPS déchiffré

Une fois le bypass actif, vérifier dans Burp Suite / mitmproxy :

```bash
# Avec mitmproxy en ligne de commande
mitmproxy --listen-host 0.0.0.0 --listen-port 8080

# Ou en mode web
mitmweb --listen-host 0.0.0.0 --listen-port 8080
# Ouvrir http://127.0.0.1:8081 dans le navigateur
```

**Checklist de validation :**

```
[ ] Le proxy affiche des requêtes HTTPS de l'app (pas d'erreur TLS)
[ ] Les headers et body des requêtes sont lisibles en clair
[ ] Les réponses serveur sont déchiffrées et visibles
[ ] Aucune erreur "certificate verify failed" côté app
[ ] Les tokens/cookies d'authentification sont visibles dans les requêtes
```

---

## 🔍 Diagnostic — Que faire si ça ne fonctionne pas ?

| Symptôme | Cause probable | Solution |
|---|---|---|
| App plante au lancement | Anti-Frida détecté | Ajouter bypass anti-Frida avant le bypass SSL |
| Erreur TLS côté app | Pinning natif actif | Utiliser `bypass_native.js` + Ghidra pour identifier la fonction |
| Proxy vide, aucune requête | Proxy mal configuré | Vérifier IP `10.0.2.2:8080` dans les paramètres Wi-Fi |
| Certificat rejeté par Android | API 24+ cert utilisateur | Pousser le cert dans `/system/etc/security/cacerts/` |
| `frida-ps -U` échoue | frida-server non lancé | Relancer `adb shell /data/local/tmp/frida-server &` |
| Hook ne se déclenche pas | Mauvais nom de classe | Utiliser `frida-trace` ou jadx pour retrouver le bon nom |

### Trouver le bon nom de classe avec frida-trace

```bash
# Tracer tous les appels TLS pour identifier les classes utilisées
frida-trace -U -f com.target.app \
  -j '*TrustManager*!*' \
  -j '*CertificatePinner*!*' \
  -j '*SSL*!*'
```

---

## ✅ Critères de réussite

| Étape | Validé quand... |
|---|---|
| Module 1 | `frida-ps -U` liste les processus Android |
| Module 2 | Le certificat CA Burp est installé, proxy accessible |
| Module 3 | Le hook TrustManager se déclenche (log Frida visible) |
| Module 4 | Les requêtes HTTPS apparaissent déchiffrées dans Burp |
| Module 5 | Un token ou credential est capturé en clair dans le proxy |

---

## 📚 Ressources complémentaires

- [OWASP MASTG — Testing Network Communication](https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/)
- [Frida documentation officielle](https://frida.re/docs/home/)
- [frida-multiple-unpinning (GitHub)](https://github.com/hluwa/frida-dexdump)
- [objection — Mobile Exploration Toolkit](https://github.com/sensepost/objection)
- [mitmproxy docs](https://docs.mitmproxy.org/)
- [PortSwigger — Configuring Burp with Android](https://portswigger.net/support/installing-burp-suites-ca-certificate-in-an-android-device)

---

## 👤 Auteur

**Hiba** — Étudiante en cybersécurité & développement logiciel  
*Lab réalisé dans le cadre d'un cours d'analyse de sécurité mobile*

---

*Dernière mise à jour : Juin 2026*
