---
name: scroll-3d-sequence
description: Construye sitios web inmersivos estilo Apple / Rolex / Sean Kochel con reconstrucción 3D fotorrealista controlada por scroll usando keyframes multi-ángulo + video AI encadenado (10s-15s, órbita 3D y cámara macro) + extracción FFmpeg a secuencia WebP (240-360+ frames) + HTML5 Canvas a 60fps sin WebGL pesado. Úsala cuando el usuario pida una web inmersiva 3D con scroll, armado de producto pieza por pieza, scrollytelling cinematográfico o secuencias de fotogramas.
---

# Scroll 3D Sequence — Cinema & Multi-Angle Edition

> **Propósito:** Lograr experiencias web inmersivas de nivel **Awwwards / Apple / Rolex** con reconstrucción 3D fotorrealista, cambios de ángulo cinematográficos, iluminación dinámica y **240 a 360+ fotogramas** a 60fps sobre HTML5 `<canvas>`, sin depender de modelos 3D geométricos pesados ni WebGL lento.

---

## 1. Arquitectura del Pipeline (Multi-Ángulo & Alta Densidad)

Para que el usuario sienta una verdadera reconstrucción 3D inmersiva (y no una transición estática frontal), el flujo se divide en **5 fases**:

### Fase 1: Keyframes Multi-Ángulo (3 a 4 Estados Clave)
En lugar de solo 2 imágenes frontales, genera **3 o 4 keyframes de estudio** sobre el mismo fondo uniforme (ej. `#0b0e0d`):
1. **Keyframe A (Macro Extremo — Interior / Maquinaria):** Plano macro cerrado (45° o frontal) mostrando engranajes, circuitos o piezas internas en movimiento.
2. **Keyframe B (Vista Lateral / Explosión 3/4):** Ángulo orbital a ~35°–45° con componentes flotando o acoplándose en profundidad Z (caja, chasis, lentes o bisel).
3. **Keyframe C (Plano Hero Final — Producto Completo):** Ángulo 3/4 dramático o frontal de catálogo con iluminación de borde (*rim lighting*) y acabado final impecable.

### Fase 2: Generación de Video Multi-Tramo con Cámara 3D (`gemini-omni-1.2-flash`)
Cuando el presupuesto de API lo permite, prioriza **duración, resolución y movimiento de cámara real** sobre la brevedad:
- **Tramo 1 (`A → B`, 5s–8s, `1080p`):** Transición con **cámara macro que retrocede y orbita 35°** mientras las piezas internas se ensamblan dentro de la estructura.
- **Tramo 2 (`B → C`, 5s–8s, `1080p`):** Continuación (`--extend` o segundo tramo `first-frame=B`, `last-frame=C`) con **órbita suave hacia el plano hero** mientras se sellan cristales, tapas y accesorios externos.
- **Concatenación con FFmpeg** (si usas 2 tramos independientes):
  ```bash
  ffmpeg -i tramo1.mp4 -i tramo2.mp4 -filter_complex "[0:v][1:v]concat=n=2:v=1:a=0[outv]" -map "[outv]" assembly_full.mp4
  ```
- **Reglas de Prompting Cinematográfico:**
  - Especificar siempre movimiento de cámara explícito: *"smooth orbital camera sweep from macro close-up to 35-degree 3/4 hero angle"*.
  - Prohibir cortes: *"zero jump cuts, continuous single-take mechanical assembly, steady studio rim lighting, pure dark #0b0e0d seamless background"*.

### Fase 3: Extracción de Alta Densidad con FFmpeg (240 – 360+ Frames)
Para un scroll largo (`500vh`–`700vh`) sin saltos perceptibles:
```bash
mkdir -p public/frames
# 10-15 segundos a 24-30 fps = 240 a 450 fotogramas WebP HD
ffmpeg -y -i assembly_full.mp4 -vf "fps=24,scale=1600:-1" -c:v libwebp -quality 82 public/frames/frame_%04d.webp
```
- **Desktop HD:** `scale=1600:-1`, `-quality 82` (~45–75 KB/frame).
- **Optimización de carga:** En 300 frames (~15 MB total), precarga primero **1 de cada 6 frames** (los 50 frames clave) para que el canvas sea interactivo en **< 300ms**, y descarga los intermedios en segundo plano sin bloquear el hilo principal.

### Fase 4: Renderizador HTML5 `<canvas>` con Carga Progresiva y 60fps
- Usa `useScroll` (`h-[550vh]` a `h-[700vh]` para darle espacio real a los 240–360 frames).
- Aplica `lerp` suave (`0.12`–`0.16`) en `requestAnimationFrame`.
- Si el frame exacto aún está descargándose, busca automáticamente el frame cargado más cercano (`idx - offset` o `idx + offset`) para que **nunca parpadee ni quede en blanco**.
- Respeta `prefers-reduced-motion`: muestra directamente el último frame ensamblado.

### Fase 5: Narrativa Editorial Sincronizada
- Divide el scroll en **4 a 5 capítulos** con tipografía editorial limpia (sin cajas HUD genéricas ni cuadrículas ruidosas).
- Integra botones de salto rápido (`01` · `02` · `03` · `04`) que hagan `window.scrollTo({ behavior: 'smooth' })` al porcentaje exacto de cada hito.
- Funde el `<canvas>` con una viñeta radial CSS (`radial-gradient(circle at center, transparent 42%, #0b0e0d 88%)`) para que el producto flote en el espacio sin bordes rectangulares.

---

## 2. Patrón de Precarga Progresiva en React + TypeScript

```tsx
// 1. Precarga prioritaria: frame 0 y múltiplos de 6 primero para arranque instantáneo
const priorityIndices: number[] = [];
const secondaryIndices: number[] = [];
for (let i = 0; i < FRAME_COUNT; i++) {
  if (i === 0 || i % 6 === 0) priorityIndices.push(i);
  else secondaryIndices.push(i);
}

const imgs: HTMLImageElement[] = new Array(FRAME_COUNT);
const loadFrame = (i: number) => {
  const img = new Image();
  img.decoding = 'async';
  img.src = framePath(i);
  imgs[i] = img;
};

priorityIndices.forEach(loadFrame);
// Carga los frames intermedios en segundo plano tras montar los clave
setTimeout(() => secondaryIndices.forEach(loadFrame), 150);
```

---

## 3. Reglas de Oro

- **Más duración + cambio de ángulo = sensación 3D real:** Un plano fijo de 5s funciona como intro; **10s–15s con órbita de cámara (`35°–45°`) y zoom macro** convierte el scroll en una experiencia cinematográfica viral.
- **Nunca uses WebGL/Three.js primitivo para productos de lujo reales** cuando el objetivo es fotorrealismo.
- **Mantén el fondo de los keyframes 100% homogéneo** para que el recorte con viñeta radial sea invisible en cualquier pantalla (`16:9`, ultrawide o móvil).
