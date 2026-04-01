# FingerScroll — Documentation technique complète

## Architecture choisie : PWA (Progressive Web App)

### Pourquoi une PWA plutôt que Flutter ou React Native ?

| Critère | Flutter/React Native | PWA (solution retenue) |
|---|---|---|
| Accès caméra | Via plugin natif | `getUserMedia()` natif |
| Latence traitement | ~80-120ms | ~30-60ms |
| Distribution | Google Play + App Store | URL directe |
| Installation | Requise | Optionnelle |
| Mise à jour | Soumission store | Immédiate |
| Taille | ~15-80 MB | ~50 KB |
| Compilation | Xcode + Android Studio | Aucune |

---

## Algorithme de détection : Frame Differencing

### Principe
L'algorithme compare chaque frame vidéo avec la précédente :

```
Frame N  ─┐
           ├──► Différence absolue pixel par pixel ──► Masque binaire
Frame N-1 ─┘         (si diff > seuil)

Masque binaire ──► Centroïde (barycentre des pixels actifs)

Centroïde(N) - Centroïde(N-1) ──► Vecteur delta (deltaX, deltaY)

Si |deltaY| > dead_zone ET |deltaY| > |deltaX| × 1.5
   ──► scrollBy(deltaY × vitesse)
```

### Paramètres internes

```javascript
// Seuil de différence pixel (0-255) — calculé depuis CONFIG.sensitivity
const threshold = 18 - sensitivity * 1.4;
// sensitivity=1  → threshold=16.6  (très sensible)
// sensitivity=5  → threshold=11    (équilibré)
// sensitivity=10 → threshold=4     (très sensible)

// Surface minimale de mouvement (anti-bruit)
const minPixels = W * H * 0.005; // 0.5% de l'image
// Ex. 320×240 = 76800 px → minimum 384 pixels en mouvement

// Frames consécutives requises avant de scroller
const MIN_CONSECUTIVE = 2; // évite les faux positifs isolés

// Multiplicateur de défilement
const scrollMult = speed * 1.8;
// speed=5 → mult=9px par pixel de déplacement caméra
```

---

## Déploiement

### Option A — Test local immédiat (Chrome DevTools)

1. Ouvrez `index.html` dans Chrome
2. Ouvrez DevTools → onglet "Sensors" → activez la caméra simulée
3. Ou ouvrez directement via `file://` (certains navigateurs bloquent `getUserMedia` en `file://`)

**Solution recommandée pour test local :**
```bash
# Python 3
python3 -m http.server 8080

# Node.js
npx serve .

# Puis ouvrir : http://localhost:8080
```

### Option B — Déploiement HTTPS (requis pour mobile)

`getUserMedia()` exige HTTPS sur mobile. Options gratuites :

**Vercel (recommandé) :**
```bash
npm i -g vercel
cd finger-scroll-app
vercel --prod
# URL HTTPS générée automatiquement
```

**Netlify :**
```bash
# Glissez le dossier sur netlify.com/drop
# URL HTTPS immédiate
```

**GitHub Pages :**
```bash
git init
git add .
git commit -m "FingerScroll v1"
git remote add origin https://github.com/VOTRE_NOM/fingerscroll.git
git push -u origin main
# Activer Pages dans Settings → Pages → main branch
```

### Option C — Installation comme app native (PWA)

**Android (Chrome) :**
1. Ouvrir l'URL HTTPS dans Chrome
2. Menu ⋮ → "Ajouter à l'écran d'accueil"
3. L'app s'installe comme une vraie application native
4. Accès hors ligne partiel (SW possible)

**iOS (Safari) :**
1. Ouvrir l'URL HTTPS dans Safari (obligatoire sur iOS)
2. Bouton Partager → "Sur l'écran d'accueil"
3. L'app s'ouvre en mode plein écran sans barre Safari

---

## Intégration dans un projet existant

### Extraction du moteur de détection

```javascript
// fingerscroll-engine.js — module autonome

export class FingerScrollEngine {
  constructor(options = {}) {
    this.config = {
      sensitivity: 5,
      speed: 5,
      ecoMode: false,
      useFront: false,
      haptic: true,
      minPixelRatio: 0.005,
      minConsecutiveFrames: 2,
      verticalDominanceRatio: 1.5,
      ...options
    };
    
    this.stream = null;
    this.animFrame = null;
    this.prevData = null;
    this.prevCentroid = null;
    this.consecutiveFrames = 0;
    this.onScrollDelta = null; // callback(deltaY: number)
    this.onDetectionChange = null; // callback(active: boolean)
  }

  get threshold() {
    return Math.round(18 - this.config.sensitivity * 1.4);
  }

  get speedMult() {
    return this.config.speed * 1.8;
  }

  async start(videoEl, canvasEl) {
    const fps = this.config.ecoMode ? 15 : 30;
    const res = this.config.ecoMode
      ? { width: 160, height: 120 }
      : { width: 320, height: 240 };

    this.stream = await navigator.mediaDevices.getUserMedia({
      video: {
        facingMode: this.config.useFront ? 'user' : { ideal: 'environment' },
        width: { ideal: res.width },
        height: { ideal: res.height },
        frameRate: { ideal: fps }
      },
      audio: false
    });

    videoEl.srcObject = this.stream;
    await videoEl.play();

    canvasEl.width = res.width;
    canvasEl.height = res.height;

    this._ctx = canvasEl.getContext('2d', { willReadFrequently: true });
    this._video = videoEl;
    this._canvas = canvasEl;
    this._detect();
  }

  stop() {
    this.stream?.getTracks().forEach(t => t.stop());
    this.stream = null;
    cancelAnimationFrame(this.animFrame);
    this.prevData = null;
    this.prevCentroid = null;
    this.consecutiveFrames = 0;
  }

  _detect() {
    this.animFrame = requestAnimationFrame(() => this._detect());
    if (!this.stream || this._video.readyState < 2) return;

    const W = this._canvas.width;
    const H = this._canvas.height;

    this._ctx.drawImage(this._video, 0, 0, W, H);
    const frameData = this._ctx.getImageData(0, 0, W, H).data;

    if (!this.prevData) { this.prevData = frameData.slice(); return; }

    let sumX = 0, sumY = 0, count = 0;
    for (let i = 0; i < frameData.length; i += 4) {
      const diff = (
        Math.abs(frameData[i]   - this.prevData[i]) +
        Math.abs(frameData[i+1] - this.prevData[i+1]) +
        Math.abs(frameData[i+2] - this.prevData[i+2])
      ) / 3;
      if (diff > this.threshold) {
        const pi = i / 4;
        sumX += pi % W;
        sumY += Math.floor(pi / W);
        count++;
      }
    }

    this.prevData = frameData.slice();

    const minPx = Math.round(W * H * this.config.minPixelRatio);
    if (count < minPx) {
      this.consecutiveFrames = 0;
      this.prevCentroid = null;
      this.onDetectionChange?.(false);
      return;
    }

    const cx = sumX / count;
    const cy = sumY / count;

    if (!this.prevCentroid) { this.prevCentroid = { x: cx, y: cy }; return; }

    const dy = cy - this.prevCentroid.y;
    const dx = cx - this.prevCentroid.x;
    this.prevCentroid = { x: cx, y: cy };

    if (Math.abs(dy) < 2 || Math.abs(dx) > Math.abs(dy) * this.config.verticalDominanceRatio) {
      this.consecutiveFrames = 0;
      return;
    }

    this.consecutiveFrames++;
    if (this.consecutiveFrames < this.config.minConsecutiveFrames) return;

    this.onDetectionChange?.(true);
    this.onScrollDelta?.(dy * this.speedMult);
  }
}

// Utilisation :
// const engine = new FingerScrollEngine({ sensitivity: 7, speed: 6 });
// engine.onScrollDelta = (dy) => window.scrollBy(0, dy);
// engine.onDetectionChange = (active) => updateUI(active);
// await engine.start(videoElement, canvasElement);
```

---

## Personnalisation avancée

### Ajuster la détection dans des conditions de faible lumière

```javascript
// Dans la fonction de détection, abaisser le seuil et augmenter minPixels
const threshold = 8; // plus bas = détecte des différences subtiles
const minPixels = W * H * 0.002; // 0.2% — zone de mouvement réduite
```

### Ajouter un filtre temporel (lissage)

```javascript
// Moyenne mobile du centroïde sur 3 frames pour un défilement plus doux
const SMOOTH = 3;
let centroidHistory = [];

centroidHistory.push({ x: cx, y: cy });
if (centroidHistory.length > SMOOTH) centroidHistory.shift();

const smoothCx = centroidHistory.reduce((s, p) => s + p.x, 0) / centroidHistory.length;
const smoothCy = centroidHistory.reduce((s, p) => s + p.y, 0) / centroidHistory.length;
```

### Activer le Service Worker (hors ligne)

```javascript
// sw.js
const CACHE = 'fingerscroll-v1';
const ASSETS = ['/', '/index.html'];

self.addEventListener('install', e =>
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(ASSETS)))
);

self.addEventListener('fetch', e =>
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)))
);

// Dans index.html :
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('./sw.js');
}
```

---

## Sécurité et conformité RGPD

- ✅ Aucune image stockée (traitement en RAM, frame discardée après analyse)
- ✅ Aucune requête réseau émise depuis le moteur de détection
- ✅ `getUserMedia()` requiert un geste utilisateur explicite
- ✅ Arrêt automatique via `document.visibilitychange`
- ✅ Toutes les pistes vidéo stoppées via `track.stop()` à la désactivation
- ✅ Compatible HTTPS uniquement (exigence navigateur)

---

## Compatibilité

| Plateforme | Navigateur | Support |
|---|---|---|
| Android | Chrome 88+ | ✅ Complet |
| Android | Firefox 85+ | ✅ Complet |
| iOS | Safari 14.3+ | ✅ Complet |
| iOS | Chrome | ⚠️ Limité (WebKit) |
| Desktop | Chrome/Edge/Firefox | ✅ Complet |

---

## Roadmap — Extensions possibles

1. **Détection de gestes** : main ouverte vs fermée via MediaPipe Hands (WASM)
2. **Défilement horizontal** : activation optionnelle de l'axe X
3. **Zoom** : deux doigts devant la caméra → pinch-to-zoom
4. **Mode présentation** : navigation slide à slide avec seuil configurable
5. **React Native / Flutter** : wrapper natif du même algorithme via `react-native-camera` + Canvas API
