# 📌 Batching and Compression

> Batching y Compression son mecanismos de optimización del lado del productor que agrupan múltiples mensajes en lotes y reducen su tamaño en bytes antes de enviarlos por la red, maximizando el rendimiento (*throughput*) y reduciendo el consumo de ancho de banda.

---

## 💡 La Analogía del Mundo Real

Imagina una tienda de comercio electrónico que despacha cientos de pedidos al día:

* **Sin Batching ni Compresión (Envíos individuales):** Cada vez que un cliente compra un artículo pequeño, un repartidor en motocicleta realiza un viaje exclusivo hasta la casa del cliente. La carretera se satura, los costos de transporte se disparan y el servicio se vuelve ineficiente.
* **Con Batching y Compresión:** La tienda acumula los paquetes en el almacén hasta llenar una caja grande (**Batching**). Además, utiliza una máquina de empacado al vacío para extraer todo el aire de los paquetes y reducir el volumen del lote (**Compresión**). Finalmente, una sola camioneta transporta esa única caja comprimida con docenas de pedidos. El ahorro en combustible, tráfico y tiempo por paquete es drástico.

---

## 🧩 Anatomía y Funcionamiento

En lugar de realizar una petición de red por cada mensaje individual, el cliente productor mantiene *buffers* de memoria por cada partición activa donde acumula registros para empaquetarlos en lotes.

```text
Mensajes individuales (RAM)
┌───┐ ┌───┐ ┌───┐ ┌───┐
│M1 │ │M2 │ │M3 │ │M4 │
└───┘ └───┘ └───┘ └───┘
  │     │     │     │
  ▼     ▼     ▼     ▼
┌──────────────────────────────┐
│  Acumulador de Lotes         │ ──► [ Algoritmo de Compresión ] ──► Envío por Red (1 solo Socket)
│ (`batch.size` / `linger.ms`) │     (gzip, snappy, lz4, zstd)          Broker
└──────────────────────────────┘
```

### 1. Acumulación en Lotes (`batch.size` y `linger.ms`)
El empaquetamiento en el productor está controlado por dos parámetros clave que actúan bajo una regla de "el primero que se cumpla":
* **`batch.size`:** Límite máximo en bytes para un lote (por defecto 16 KB). Si el buffer de una partición se llena hasta este límite, el lote se envía inmediatamente.
* **`linger.ms`:** Tiempo máximo en milisegundos que el productor esperará para acumular más mensajes antes de despachar el lote. Si el volumen es bajo y el lote no se ha llenado, al expirar este tiempo se enviará con los mensajes acumulados hasta el momento.

### 2. Algoritmos de Compresión (`compression.type`)
La compresión no se aplica mensaje por mensaje, sino **a nivel de lote completo**. Esto permite que el algoritmo aproveche la repetición de estructuras (como esquemas JSON o encabezados repetidos) logrando tasas de compresión significativamente altas. Los algoritmos soportados incluyen:
* **`snappy` / `lz4`:** Diseñados para máxima velocidad de CPU con un nivel de compresión moderado. (Ideal para latencia ultra baja).
* **`gzip` / `zstd`:** Diseñados para máxima tasa de compresión, reduciendo drásticamente el espacio en red y disco a costa de un mayor uso de CPU.

### 3. Procesamiento de Extremo a Extremo (*End-to-End*)
Cuando la compresión está activada, el broker **no descomprime el lote** al recibirlo; lo escribe en disco exactamente tal como llegó. De igual forma, cuando un consumidor solicita datos, el broker envía el bloque comprimido directamente desde el disco a la tarjeta de red mediante la técnica **Zero-Copy**. La descompresión ocurre únicamente cuando el mensaje llega al proceso final del consumidor.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Configurar `linger.ms` en un valor mayor siempre añade latencia a todas las peticiones"* | Si el flujo de datos es alto y llena el `batch.size` al instante, el lote se despacha inmediatamente sin esperar a que expire el tiempo de `linger.ms`. |
| *"El broker descomprime cada lote para validar los mensajes antes de escribirlos en disco"* | El broker almacena los lotes comprimidos en estado puro para preservar la eficiencia de entrada/salida (*Zero-Copy*). Solo el consumidor final descomprime los datos. |
| *"Comprimir datos siempre empeora el rendimiento por la carga adicional en la CPU"* | En la mayoría de los entornos de red, el cuello de botella es el ancho de banda. Reducir el tamaño en bytes suele acelerar la transferencia global, compensando ampliamente el uso marginal de CPU. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué parámetro de configuración del productor controla el tiempo que esperará un lote incompleto antes de enviarse al broker?</b></summary>

<p><b>Respuesta:</b> El parámetro <code>linger.ms</code>. Define la espera deliberada en milisegundos para permitir que se acumulen más registros en el mismo lote antes de realizar la transmisión por red.</p>

</details>

<details>
<summary><b>2. ¿Por qué la compresión a nivel de lote (*batch-level*) es más eficiente que comprimir cada mensaje individualmente?</b></summary>

<p><b>Respuesta:</b> Porque los algoritmos de compresión detectan patrones y cadenas repetidas a lo largo de todo el lote (como claves JSON o nombres de atributos), lo que genera una tasa de reducción de espacio mucho mayor que al comprimir un único registro pequeño aislado.</p>

</details>

<details>
<summary><b>3. ¿Qué componente de la arquitectura es el encargado de realizar la descompresión de los datos cuando se utiliza compresión de extremo a extremo?</b></summary>

<p><b>Respuesta:</b> El <b>consumidor</b>. El broker almacena y transmite el lote comprimido en su estado original; la descompresión la ejecuta la aplicación consumidora al recibir el paquete de datos.</p>

</details>