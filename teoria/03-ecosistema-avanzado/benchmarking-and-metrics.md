# 📌 Benchmarking and Metrics

> Benchmarking and Metrics es el conjunto de técnicas, herramientas y mediciones cuantitativas utilizadas para evaluar el rendimiento, la capacidad de procesamiento y los tiempos de respuesta de una arquitectura de eventos bajo diferentes escenarios de carga.

---

## 💡 La Analogía del Mundo Real

Imagina un banco de pruebas para autos de alta velocidad:

Para evaluar un vehículo no basta con observar su aspecto estético; se le somete a una pista de pruebas especial. Allí se mide la velocidad máxima sostenida en recta (**Rendimiento / Throughput**), el tiempo que tarda el motor en responder al pisar el acelerador (**Latencia**), y cómo reacciona el auto al enfrentar pendientes o curvas extremas (**Percentiles de latencia**).

Además, antes de tomar el cronómetro oficial, el piloto da dos vueltas de prueba a velocidad moderada para calentar las llantas y el motor (**Warm-up Phase**). Si se midiera el auto con el motor congelado, los tiempos registrados serían pésimos e irrestrictos a la realidad del vehículo en su temperatura óptima de operación.

---

## 🧩 Anatomía y Funcionamiento

Las métricas de rendimiento en un clúster evalúan la capacidad del sistema tanto en términos de volumen masivo como de velocidad de respuesta individual por evento.

```text
Productor / Herramienta de Carga
            │
            ├──► [ Rendimiento / Throughput ] ──► Records/sec & MB/sec
            │
            └──► [ Distribución de Latencia ]
                      ├──► Mediana (p50) ──► Comportamiento Habitual
                      └──► Colas / Outliers (p95, p99) ──► Picos de Latencia (GC, Red)
```

### 1. Rendimiento frente a Latencia (Throughput vs. Latency)
Existen dos grandes indicadores complementarios:
* **Throughput (`records/sec` / `MB/sec`):** Mide la cantidad total de mensajes o volumen de datos transferido y procesado por unidad de tiempo.
* **Latencia (`avg latency` / `max latency`):** Mide el tiempo (en milisegundos) que le toma a un mensaje individual viajar desde el cliente hacia el broker y recibir su confirmación.

### 2. Análisis por Percentiles ($p_{50}$, $p_{95}$, $p_{99}$)
Confiar únicamente en la latencia promedio (*avg latency*) es un error común que oculta comportamientos anómalos. La distribución por percentiles refleja el comportamiento real:
* **$p_{50}$ (Mediana):** El 50% de los mensajes se procesó por debajo de este tiempo. Refleja la experiencia del usuario promedio.
* **$p_{95}$ y $p_{99}$ (Latencias de Cola):** El 95% y 99% de los mensajes se procesaron por debajo de este tiempo. Revela el impacto de eventos esporádicos como pausas de la máquina virtual (GC), saturación momentánea de red o retransmisiones.

### 3. Fase de Calentamiento (*Warm-up Phase*)
En entornos basados en Java o máquinas virtuales, la primera ejecución de carga suele arrojar métricas distorsionadas. La fase de calentamiento (*warm-up*) es una corrida previa de baja escala diseñada para cargar clases en memoria, compilar código *Just-In-Time* (JIT) y estabilizar los *buffers* de red antes de tomar las mediciones definitivas.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"La latencia promedio es suficiente para evaluar la estabilidad en producción"* | El promedio enmascara picos severos. Deben monitorearse los percentiles $p_{95}$ y $p_{99}$ para detectar demoras esporádicas por pausas de recolección de basura o congestión. |
| *"Ejecutar la prueba de rendimiento de inmediato arroja los datos más precisos"* | La primera corrida sufre penalizaciones de rendimiento por arranque en frío de la máquina virtual. Se requiere un calentamiento previo (*warm-up*) para obtener cifras fiables. |
| *"Un mayor número de mensajes por segundo siempre indica una mejor configuración"* | Un alto número de `records/sec` puede ser el resultado de un empaquetamiento masivo de lotes (*batching*) que aumenta la latencia individual de cada mensaje. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Por qué es engañoso confiar únicamente en la latencia promedio al medir una arquitectura de eventos?</b></summary>

<p><b>Respuesta:</b> Porque el promedio diluye los valores extremos (*outliers*). Un promedio aparentemente bajo puede ocultar picos críticos de retraso que afectan al percentil 99 ($p_{99}$) causados por saturación de disco, pausas de GC o congestión en la red.</p>

</details>

<details>
<summary><b>2. ¿En qué consiste la fase de calentamiento (*warm-up*) durante una prueba de carga?</b></summary>

<p><b>Respuesta:</b> Es una prueba preliminar de corta duración que inicializa la máquina virtual (JVM), compila código en tiempo de ejecución (JIT) y llena los buffers de memoria para garantizar que las métricas de la prueba oficial reflejen el rendimiento del clúster y no los costos de arranque.</p>

</details>

<details>
<summary><b>3. ¿Qué indica exactamente la métrica del percentil $p_{99}$ en un reporte de desempeño?</b></summary>

<p><b>Respuesta:</b> Indica que el 99% de todas las solicitudes enviadas respondieron en un tiempo igual o inferior a ese valor, mientras que solo el 1% restante experimentó una latencia mayor.</p>

</details>