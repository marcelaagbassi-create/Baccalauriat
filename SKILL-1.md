---
name: baccalaureat-game-dev
description: >
  Skill pour développer, débugger et améliorer l'application web Baccalauréat
  (marcelaagbassi-create.github.io) — jeu de mots multijoueur hébergé sur GitHub Pages
  avec Firebase Realtime Database. Utilise ce skill dès que l'utilisateur mentionne
  le jeu Baccalauréat, un bug dans index.html, Firebase, les salons multijoueurs,
  la messagerie, les modes de jeu, le chat en direct, ou toute fonctionnalité
  de cette application. Couvre aussi les erreurs de syntaxe JS, les problèmes
  de chargement Firebase, les règles de sécurité, et le déploiement GitHub Pages.
---

# Baccalauréat — Jeu Web Multijoueur

Application web de jeu de mots style "Baccalauréat" (chaque joueur remplit des
catégories avec une lettre tirée au sort). Hébergée sur GitHub Pages, backend
Firebase Realtime Database.

## Architecture technique

- **Fichier unique** : `index.html` (~6000 lignes, tout-en-un : HTML + CSS + JS)
- **Hébergement** : GitHub Pages (`marcelaagbassi-create.github.io`)
- **Base de données** : Firebase Realtime Database (REST API, pas de SDK)
- **Auth** : Anonyme via Identity Toolkit REST + refresh token en localStorage
- **Firebase bridge** : `window._fb` — pont entre l'API REST Firebase et le code JS
- **Script JS** : Script plain (pas `type="module"`) pour compatibilité GitHub Pages

## Structure Firebase Realtime Database

```
salons/{code}/          — Salons de jeu multijoueur
joueurs/{uid}/          — Profils joueurs
msgs/{uid}/{convId}/    — Index conversations messagerie
msgconv/{convId}/msgs/  — Messages de conversations
presence/{uid}/         — Présence en ligne
appels/{convId}/        — Signaling WebRTC pour appels
chatGlobal/{canal}/     — Chat global (général/stratégie/off-topic)
```

## Règles Firebase recommandées

```json
{
  "rules": {
    "salons": { ".read": "auth != null", ".write": "auth != null" },
    "joueurs": { ".read": "auth != null", ".write": "auth != null" },
    "msgs": { "$uid": { ".read": "auth != null", ".write": "auth != null" } },
    "msgconv": { ".read": "auth != null", ".write": "auth != null" },
    "presence": { ".read": "auth != null", ".write": "auth != null" },
    "appels": { ".read": "auth != null", ".write": "auth != null" },
    "chatGlobal": { ".read": "auth != null", ".write": "auth != null" }
  }
}
```

## Objet joueur `J`

```js
let J = {
  uid, pseudo, emoji, avatarImg, biid, tel,
  pieces: {or, argent, bronze},
  debloques: [], indices: 3, titreEquipe: null
}
```

Sauvegardé dans `localStorage` clé `'bac3'` via `sauvegarderLocal()`.

## Firebase REST Bridge (`window._fb`)

Le SDK Firebase ne peut pas être chargé via CDN sur certains navigateurs mobiles.
À la place, toutes les opérations Firebase passent par `window._fb` :

```js
window._fb = {
  ref(db, path)              // → makeRef(path)
  set(ref, val)              // → PUT REST
  get(ref)                   // → GET REST (once)
  push(ref)                  // → POST REST (génère clé Firebase)
  update(ref, val)           // → PATCH REST (supporte chemins imbriqués)
  remove(ref)                // → DELETE REST
  onValue(ref, cb, errCb)    // → SSE EventSource
  onChildAdded(ref, cb)      // → SSE EventSource (child_added)
  off(ref)                   // → ferme SSE
  serverTimestamp()          // → {".sv": "timestamp"}
  signInAnonymously()        // → Identity Toolkit REST
  onAuthStateChanged(a, cb)  // → callback sur auth
}
```

### Initialisation auth

```js
// Token stocké dans localStorage clé '_fb_auth'
// { refreshToken, uid }
// Refresh automatique via securetoken.googleapis.com
// Retry sur 401 (token expiré)
```

## Fonctions globales exposées sur `window`

Le script est plain (pas module). Toutes les fonctions sont exposées via
`Object.assign(window, {...})` **à la fin du script**, après toutes les
définitions. C'est critique — les mettre avant les définitions donne `undefined`.

Fonctions principales :
- `allerEcran(id)` — navigation entre écrans
- `allerCreerSalon()` — créer un salon multijoueur
- `lancerPartie()` — lancer la partie depuis le salon
- `allerChatGlobal()` — ouvrir la messagerie
- `msgEnvoyer()` — envoyer un message dans une conversation
- `sauvegarderProfil()` — sauvegarder profil (pseudo + téléphone obligatoire)

## Bugs récurrents et solutions

### `firebase is not defined`
- **Cause** : SDK Firebase CDN bloqué par le navigateur
- **Solution** : Utiliser `window._fb` REST bridge (pas de dépendance externe)

### `[function] is not defined` sur boutons
- **Cause** : `Object.assign(window, {...})` placé avant les définitions de fonctions
- **Solution** : Toujours placer l'`Object.assign` à la **fin** du script

### Double message envoyé
- **Cause** : `onChildAdded` Firebase v9 déclenche pour les messages existants
- **Solution** : Stocker les clés déjà chargées dans `seenKeys`, ignorer les doublons

### `Database lives in a different region`
- **Cause** : URL de la DB incorrecte
- **URL correcte** : `https://baccalaureats-default-rtdb.europe-west1.firebasedatabase.app`

### `Connexion en cours, réessaie...` sur Créer un salon
- **Cause** : `authReady = false` car auth Firebase REST échoue
- **Solution** : Mettre `authReady = true` dans `initApp()` dès qu'un `uid` local existe

### `Missing catch or finally after try` / `Unexpected token 'else'`
- **Cause** : Accolades mal imbriquées dans le JS
- **Solution** : Vérifier avec `node --check` ou `node --input-type=module --check`

### `MediaRecorder TypeError` sur mobile
- **Cause** : `mimeType` non supporté sur Android
- **Solution** :
```js
const mimes = ['audio/webm;codecs=opus','audio/webm','audio/ogg;codecs=opus','audio/mp4',''];
const mimeType = mimes.find(m => m==='' || MediaRecorder.isTypeSupported(m)) || '';
new MediaRecorder(stream, mimeType ? {mimeType} : {})
```

### `getVoices` crash
- **Cause** : `window.speechSynthesis` undefined sur certains navigateurs
- **Solution** : Toujours vérifier `if (!window.speechSynthesis) return;`

## Modes de jeu

- **Solo** : classique, chrono, survie, entraînement, catégorie, difficile
- **Multijoueur** : salon avec code (hôte + invités), chat en direct, voix
- **PlayBot** : adversaire IA (solo contre bot)
- **Tournoi** : bracket automatique

## Messagerie (style WhatsApp)

- Conversations **privées** (par numéro de téléphone) et **groupes**
- Messages vocaux (MediaRecorder → base64 → Firebase)
- Appels WebRTC (STUN : `stun:stun.l.google.com:19302`)
- 3 canaux chat global : général, stratégie, off-topic

## Workflow de debug

1. Vérifier syntaxe JS : `python3 verify.py` (node --check sur le gros script)
2. Chercher l'erreur exacte avec `grep -n "pattern"` ou `view` autour de la ligne
3. Utiliser `str_replace` pour corriger (jamais réécrire le fichier entier)
4. Re-vérifier syntaxe après chaque correction
5. Tester avec `present_files`

## Vérificateur syntaxe (réutilisable)

```python
# /tmp/verify_plain.py
import re, subprocess
with open('/mnt/user-data/outputs/index.html', 'r', encoding='utf-8') as f:
    content = f.read()
scripts = re.findall(r'<script(?![^>]*src)[^>]*>(.*?)</script>', content, re.DOTALL)
big = max(scripts, key=len)
proc = subprocess.run(['node', '--check'], input=big, capture_output=True, text=True)
print("✅ Syntax OK" if proc.returncode == 0 else "❌ " + proc.stderr[:500])
```

## Sécurité

- La clé API Firebase (`AIzaSy...`) est **normale à exposer** dans le code client web
  Firebase — la sécurité est assurée par les règles Realtime Database, pas par la clé
- GitHub envoie des alertes "secret detected" pour les clés API Google — c'est un
  faux positif pour Firebase web apps, mais restreindre la clé dans Google Cloud
  Console (domaines autorisés) est recommandé
- Les emails Firebase "règles non sécurisées" s'arrêtent avec des règles par nœud
  ou en désactivant les alertes dans Firebase Console → Paramètres des alertes

## Déploiement

1. Modifier `index.html` localement
2. Push sur GitHub → déploiement automatique GitHub Pages
3. URL : `https://marcelaagbassi-create.github.io/Baccalaur-at/`
