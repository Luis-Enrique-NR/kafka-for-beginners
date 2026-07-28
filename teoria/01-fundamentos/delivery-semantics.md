# 📌 Delivery Semantics

> Las Delivery Semantics son las garantías formales que ofrece una arquitectura de eventos sobre la cantidad de veces que un mensaje será entregado y procesado por un consumidor en presencia de fallos.

---

## 💡 La Analogía del Mundo Real

Imagina que realizas una compra mediante un servicio de entregas a domicilio:

* **At-most-once (Como máximo una vez):** El repartidor deja el paquete en la puerta de la calle sin pedir firma y se retira. Si alguien roba el paquete antes de que abras la puerta, la tienda no enviará un reemplazo. Nivel de riesgo: puedes recibir el paquete **0 o 1 vez**, pero jamás recibirás duplicados.
* **At-least-once (Al menos una vez):** El repartidor intenta entregarte el paquete. Si al llegar a la central de envíos la señal telefónica falla y el sistema no registra la entrega exitosa, la tienda asumirá que el paquete se perdió y despachará un segundo paquete de repuesto. Nivel de riesgo: recibirás el paquete **1 o más veces** (posible duplicado), pero garantizas que nunca te quedarás sin el producto.
* **Exactly-once (Exactamente una vez):** El repartidor escanea un código QR único pegado en tu puerta en un sistema transaccional integrado en tiempo real. Si intenta escanear el mismo paquete dos veces o la red se interrumpe, el sistema identifica el código duplicado y cancela el segundo despacho de forma transparente. Nivel de riesgo: recibirás el paquete **exactamente 1 vez**.

---

## 🧩 Anatomía y Funcionamiento

El modelo de semántica depende directamente del momento exacto en que el consumidor efectúa el **commit del offset** con respecto al procesamiento real del mensaje.

```text
Entrada de evento ──────► Consumidor ──────► Procesamiento / Base de datos

[At-most-once]   1. Guarda Offset ──► 2. Procesar evento  (Si hay crash en 2 ──► Mensaje Perdido)

[At-least-once]  1. Procesar evento ──► 2. Guarda Offset  (Si hay crash en 1 ──► Mensaje Duplicado)

[Exactly-once]   ┌─────────────────────────────────────────────────┐
                 │ Procesamiento + Offset en 1 Transacción Atomica │
                 └─────────────────────────────────────────────────┘
```

### 1. At-least-once (Al menos una vez)
Es la semántica por defecto en sistemas distribuidos. El consumidor lee el mensaje, ejecuta la lógica de negocio (guardar en base de datos, enviar un correo) y **finalmente confirma el offset** en el broker. Si la aplicación sufre un fallo (*crash*) durante el procesamiento antes de hacer el *commit*, al reiniciarse volverá a leer los mismos eventos, procesándolos por segunda vez. **Garantiza cero pérdida de datos, pero requiere tolerancia a duplicados.**

### 2. At-most-once (Como máximo una vez)
El consumidor lee el lote de eventos y **guarda el offset inmediatamente**, antes de procesar el contenido. Si la aplicación se interrumpe o falla durante la ejecución del procesamiento, al reconectarse comenzará desde el nuevo offset guardado, omitiendo los mensajes que no alcanzó a procesar. **Garantiza cero duplicados, pero permite pérdida de datos.**

### 3. Exactly-once (Exactamente una vez)
Garantiza que cada evento sea procesado exactamente una sola vez dentro del flujo completo (*Read-Process-Write*). Requiere la combinación de productores idempotentes (`enable.idempotence=true`), la gestión de transacciones de offsets y el aislamiento de lectura en consumidores (`isolation.level=read_committed`). Es la opción más segura pero la de mayor sobrecosto computacional.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"La semántica At-least-once evita que los datos se procesen más de una vez"* | Garantiza únicamente que **nunca se perderán mensajes**. Un rebalanceo o *crash* antes de hacer el *commit* del *offset* provocará que los mensajes se procesen múltiples veces. |
| *"Configurar Exactly-once en Kafka resuelve la duplicación en cualquier sistema externo"* | La garantía *Exactly-once* nativa de Kafka aplica al flujo interno del clúster (*Kafka-to-Kafka*). Si el consumidor escribe en una base de datos externa o API de terceros, debe implementar la **idempotencia** en el destino. |
| *"At-most-once debe evitarse siempre en arquitecturas de software"* | Es el patrón óptimo para telemetría de alto volumen o métricas de sensores en tiempo real donde perder un dato aislado no afecta el sistema y procesar duplicados distorsionaría las estadísticas. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué sucede en la semántica At-least-once si un consumidor lee un evento, lo guarda en la base de datos y sufre un crash antes de hacer el commit del offset?</b></summary>

<p><b>Respuesta:</b> Al reiniciar, el consumidor solicitará de nuevo los registros a partir del último offset confirmado. Como el offset del evento procesado no se guardó, volverá a recibirlo y lo procesará una segunda vez, generando un duplicado en la base de datos a menos que exista un control de idempotencia.</p>

</details>

<details>
<summary><b>2. ¿Qué propiedad debe implementar una aplicación consumidora para operar de forma segura bajo la semántica At-least-once?</b></summary>

<p><b>Respuesta:</b> Debe ser <b>Idempotente</b>. Esto significa que procesar el mismo mensaje múltiples veces produce exactamente el mismo resultado en el sistema que procesarlo una sola vez (por ejemplo, usando claves primarias únicas o restricciones <i>ON CONFLICT DO NOTHING</i> en base de datos).</p>

</details>

<details>
<summary><b>3. ¿En qué momento guarda el offset un consumidor configurado bajo la semántica At-most-once?</b></summary>

<p><b>Respuesta:</b> Confirma el offset inmediatamente después de recibir el mensaje desde el broker, antes de ejecutar la lógica de procesamiento interna del negocio.</p>

</details>