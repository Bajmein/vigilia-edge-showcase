# Vigilia Edge — Arquitectura: El problema del tensor de video

Este documento explica las decisiones de diseño detrás del pipeline de video de Vigilia Edge
Desktop. El objetivo central es mover frames de cámara a través de múltiples etapas de
procesamiento con latencia mínima y sin copias de memoria innecesarias.

---

## El desafío

El procesamiento de video en tiempo real en Python enfrenta tres restricciones fundamentales:

1. **fps**: A 25 fps hay 40ms por frame. Cada copia de un tensor de video (típicamente
   1920×1080×3 bytes = ~6 MB) consume tiempo de bus y CPU que se acumula en el presupuesto
   de latencia.

2. **UI thread**: PySide6 exige que todo el renderizado ocurra en el hilo principal de Qt.
   Hacer inferencia en ese mismo hilo bloquearía la UI y degradaría la experiencia de usuario.

3. **GIL**: El Global Interpreter Lock de CPython impide el paralelismo real entre hilos Python.
   La inferencia en GPU con ONNX Runtime puede liberar el GIL, pero el decodificador de frames
   y el scheduler deben operar fuera de él para no introducir contención.

---

## La solución: Zero-Copy + Multiprocessing + Rust

### Capa 1 — Multiprocessing para romper el GIL

Los workers de inferencia corren como **procesos independientes** (`multiprocessing`), no como
hilos. Cada proceso tiene su propio intérprete Python y su propio GIL, lo que permite
paralelismo real entre el proceso de detección, el de tracking y el hilo UI de Qt.

### Capa 2 — Shared Memory para eliminar copias

Los frames de video se escriben **una sola vez** en un buffer de memoria compartida
(`multiprocessing.shared_memory`). Los procesos de inferencia y el hilo UI leen desde esa
misma región de memoria sin hacer copias. El resultado: el tensor de ~6 MB por frame
existe en exactamente un lugar en RAM durante toda su vida útil.

### Capa 3 — Iceoryx para metadatos de ultra-baja latencia

Los metadatos (bounding boxes, scores, timestamps) son pequeños pero se actualizan a alta
frecuencia. Iceoryx es un middleware IPC publicador/suscriptor con latencia sub-microsegundo
y semántica de zero-copy para mensajes pequeños. Permite que los workers de inferencia
publiquen resultados y que el hilo UI los consuma sin locks de Python.

### Capa 4 — Rust Core para el trabajo pesado sin GIL

La extensión Rust (compilada con Maturin y expuesta vía PyO3) gestiona:
- Decodificación de frames desde el stream de cámara
- Scheduling de workers (cuándo lanzar cada proceso, backpressure)
- Recolección de métricas del pipeline

Todo esto ocurre fuera del GIL de Python. El Rust Core se llama desde Python como una
biblioteca de extensión nativa (`.so`/`.pyd`), pero no tiene GIL que lo frene.

---

## Flujo completo

```
Cámara
  │  (stream de video)
  ▼
Rust Core — decodifica frame → escribe en Shared Memory
  │  (puntero + timestamp vía Iceoryx)
  ▼
Inference Workers (Proceso 1: Detector, Proceso 2: Tracker)
  │  leen frame desde Shared Memory sin copia
  │  ejecutan ONNX Runtime GPU
  │  publican bboxes + scores vía Iceoryx
  ▼
IPC Layer (Shared Memory + Iceoryx bus)
  │
  ▼
UI Thread Qt — consume metadatos vía Iceoryx → renderiza overlay sobre frame
```

---

## Resultado

El pipeline alcanza una latencia de extremo a extremo de **< 2 frames (≈80ms a 25fps)**
desde la captura de cámara hasta el renderizado del overlay en pantalla.

No hay copias de tensores entre procesos: el frame vive en Shared Memory desde que el
Rust Core lo decodifica hasta que el hilo Qt lo consume. Los metadatos (bounding boxes)
viajan por Iceoryx con latencia sub-milisegundo.

Esto hace posible un sistema de vigilancia que responde en tiempo real desde hardware
de escritorio sin depender de infraestructura cloud.
