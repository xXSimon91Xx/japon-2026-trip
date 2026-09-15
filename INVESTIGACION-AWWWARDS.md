# Investigación: cómo se construyen las webs 3D de Awwwards

> Recopilado por subagente (14 fuentes: Codrops, three.js discourse, GSAP docs,
> web.dev, GitHub topics). **Nota honesta**: sin acceso a la telemetría interna
> de los SOTD privados, las cifras son rangos de referencia, no medidas suyas.

## 1. LA TÉCNICA MÁS IMPACTANTE: una sola curva de cámara

**El patrón ganador** es **UNA sola `CatmullRomCurve3`** para todo el viaje de la
cámara, más una **curva gemela para el `lookAt`**, gobernadas por **un único
ScrollTrigger** con `scrub` ~1.2 sobre el track completo.

Convierte N secciones en **un solo plano-secuencia** y **elimina los cortes**
(el salto que tengo entre `#hero` y `#cine`).

```js
import * as THREE from 'three';
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
import Lenis from 'lenis';
gsap.registerPlugin(ScrollTrigger);

const lenis = new Lenis({ lerp: 0.09 });
function raf(t){ lenis.raf(t); requestAnimationFrame(raf); }
requestAnimationFrame(raf);
lenis.on('scroll', ScrollTrigger.update);

// 1. UNA sola curva para TODO el viaje
const camCurve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(0, 2.2, 14),
  new THREE.Vector3(6, 3.0, 6),
  new THREE.Vector3(-5, 2.0, -2),
  new THREE.Vector3(0, 4.5, -12),
  new THREE.Vector3(8, 1.5, -22),
], false, 'centripetal', 0.5);

const targetCurve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(0, 1.5, 0),
  new THREE.Vector3(0, 1.5, -4),
  new THREE.Vector3(0, 2.0, -10),
  new THREE.Vector3(0, 2.5, -18),
  new THREE.Vector3(0, 1.0, -26),
], false, 'centripetal', 0.5);

const state = { t: 0 };
const lookTarget = new THREE.Vector3();

function applyCam(t){
  camCurve.getPointAt(THREE.MathUtils.clamp(t,0,1), camera.position);
  targetCurve.getPointAt(THREE.MathUtils.clamp(t,0,1), lookTarget);
  camera.lookAt(lookTarget);
}

ScrollTrigger.create({
  trigger: '#scroll-track', start: 'top top',
  end: 'bottom bottom', scrub: 1.2,
  onUpdate: (self) => { state.t = self.progress; }
});
```

## 2. HTML sobre 3D continuo
- **Canvas `fixed` z-index 0**; secciones HTML con `min-height: 100-200vh` por parada de cámara.
- Sincronizar por `progress = index/(n-1)`, con `data-progress` en cada sección.
- Texto con `mix-blend-mode` y `pointer-events: none` excepto los CTAs.

## 3. Transiciones sin corte
- **Niebla** (`THREE.FogExp2`) + fade a color en `onUpdate` según rango de `t`.
- Crossfade entre render targets con `shader mix()`.
- Túneles/partículas como *wipes* orgánicos entre "biomas".

## 4. Premium feel
- **Lenis `lerp` 0.08-0.1** + **scrub 1.2**.
- Reveals con **SplitText** y **clip-path**.
- **Cursor custom** con lerp 0.15 y estados `data-cursor`.
- **Preloader con % real** (`THREE.LoadingManager`).

## 5. Rendimiento (presupuesto)
- Instancing + geometría mergeada.
- Texturas **KTX2/Basis** + **DRACO**.
- **`pixelRatio` cap 1.5-2**.
- **Postprocesado a media resolución**.
- Frustum culling y LOD.
- **Budget SOTD típico: <3 MB inicial, LCP <2.5 s, 60 fps desktop / 30+ móvil.**
- Objetivo: **60 fps con <100 draw calls y <500k triángulos visibles**.
- Si cae: desactivar bloom → bajar `pixelRatio` a 1 → reducir partículas.

## 6. Checklist para implementar
- [ ] Sustituir las cámaras por sección por **UNA curva continua**
- [ ] `scrub` 1.2 en un único ScrollTrigger
- [ ] Preloader con progreso real
- [ ] Postprocesado a media resolución
- [ ] `pixelRatio` limitado a 1.5
- [ ] Medir draw calls y triángulos en la escena real
