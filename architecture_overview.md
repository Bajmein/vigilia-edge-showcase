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

## La capa de Cognición — más allá del scheduling

El núcleo Rust de Vigilia Edge no es simplemente un scheduler de procesos. Implementa la
capa cognitiva completa del sistema: el pipeline de análisis que transforma bounding boxes
crudas en eventos de seguridad accionables.

### Multi-tracker con selección de algoritmo

El núcleo expone tres algoritmos de tracking seleccionables desde `config.yaml`:
**BoT-SORT**, **ByteTrack** y **Hybrid-SORT**. La selección no requiere recompilación —
el algoritmo activo se carga al inicio del proceso. Esto permite ajustar el balance entre
precisión de re-identificación (BoT-SORT) y velocidad en escenas densas (ByteTrack) sin
tocar código.

### Filtro de Kalman para predicción de movimiento

La predicción de la posición de objetos entre frames se implementa con un filtro de Kalman
lineal usando la crate `nalgebra`. Esto permite mantener tracks de objetos durante oclusiones
breves y reduce los false-positive de intrusión cuando un objeto pasa brevemente fuera de
frame.

### Geometría computacional y validación de zonas

La validación de zonas de intrusión y cruce de líneas usa cálculos IoU vectorizados en Rust,
sin lock de intérprete Python en el hot path. Cada frame desencadena validación geométrica de
todos los tracks activos contra todas las zonas configuradas — a 25fps, esto es 25 evaluaciones
por segundo por zona, sin que Python intervenga en el ciclo interno.

### SAHI para objetos pequeños

Cuando el modo SAHI (Slicing Aided Hyper Inference) está activo, la capa de Cognición
coordina el slicing del frame en ventanas superpuestas y la fusión de las detecciones
parciales. Esto permite detectar objetos pequeños que el modelo no detectaría sobre el
frame completo, a costa de latencia adicional controlada por configuración.

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

---

## La capa de Infraestructura — el backbone invisible

Las cuatro capas de la solución (Multiprocessing, Shared Memory, Iceoryx, Rust Core) descansan
sobre una capa transversal de infraestructura que hace el sistema resiliente y observable en
producción.

### Ring Buffers como IPC complementario

Iceoryx es óptimo para metadatos de alta frecuencia y tamaño fijo (bounding boxes, timestamps).
Para eventos menos frecuentes con tamaño variable — logs de auditoría, cambios de estado del
motor de reglas, señales de respuesta hacia audio y notificaciones — el sistema usa Ring Buffers
como canal IPC complementario. Cada mecanismo opera en el dominio donde tiene ventaja: Iceoryx
para el hot path, Ring Buffers para el plano de control.

### Prometheus para observabilidad en producción

El pipeline expone métricas vía Prometheus: fps efectivos por proceso, latencia de inferencia
por frame, tasa de detecciones por zona, uso de memoria compartida. En entornos de producción,
estas métricas permiten detectar degradación de rendimiento (un proceso de inferencia que cae
por debajo del fps objetivo) antes de que impacte la experiencia del operador.

### Defensive programming y ciclo de vida de recursos

Los procesos de larga duración en un pipeline de video enfrentan escenarios de fallo que no
ocurren en un script de uso único: cámaras que se desconectan, GPU que se quedan sin memoria,
procesos hijo que terminan inesperadamente. La capa de infraestructura aplica patrones de
defensive programming — reintentos con backoff, liberación determinista de recursos al estilo
RAII — para que el pipeline se recupere automáticamente de fallos transitorios sin intervención
del operador.
