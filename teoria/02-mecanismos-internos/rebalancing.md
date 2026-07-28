# 📌 Rebalancing

> El Rebalancing es el proceso mediante el cual un clúster redistribuye dinámicamente las particiones de un topic entre los miembros activos de un Consumer Group para mantener el balance de carga y garantizar la tolerancia a fallos.

---

## 💡 La Analogía del Mundo Real

Imagina un centro de atención telefónica con 3 líneas de entrada (particiones) y 3 operadores (consumidores). Cada operador atiende una línea fija.

Si el Operador 1 sufre una indisposición y debe retirarse repentinamente, el supervisor (Coordinador del Grupo) interviene para **reorganizar la asignación de llamadas**: transfiere temporalmente la Línea 1 al Operador 2 o al Operador 3 para evitar que las llamadas queden desatendidas. Posteriormente, si se contrata a un nuevo operador o el Operador 1 regresa, el supervisor redistribuye nuevamente las líneas para que cada trabajador vuelva a tener una carga de trabajo equitativa.

---

## 🧩 Anatomía y Funcionamiento

El rebalanceo es un mecanismo automático coordinado entre el broker de Kafka/Redpanda y los clientes consumidores para garantizar que cada partición activa sea procesada por un único consumidor del grupo a la vez.

```text
Estado Inicial: (3 Particiones, 2 Consumidores)
Partición 0 ──► [ Consumidor A ]
Partición 1 ──► [ Consumidor A ]
Partición 2 ──► [ Consumidor B ]

           ▼ (Evento: Se une el Consumidor C)
───► [ REBALANCING EN PROCESO DE REASIGNACIÓN ] ◄───

Estado Final: (3 Particiones, 3 Consumidores)
Partición 0 ──► [ Consumidor A ]
Partición 1 ──► [ Consumidor B ]
Partición 2 ──► [ Consumidor C ]
```

### 1. Desencadenadores del Rebalanceo (*Triggers*)
Un proceso de rebalanceo se inicia automáticamente ante cualquiera de los siguientes eventos:
* Un nuevo consumidor se une a un Consumer Group existente.
* Un consumidor activo se desconecta voluntariamente o sufre una caída (*crash*).
* Un consumidor es expulsado del grupo por exceder los tiempos límite de procesamiento o señal de vida (*heartbeat*).
* Se añaden nuevas particiones a un topic existente.

### 2. Detección de Fallos (*Heartbeats* y *Timeouts*)
El clúster monitorea la salud de cada consumidor mediante dos parámetros principales:
* **`session.timeout.ms`:** Tiempo máximo que el clúster espera recibir una señal periódica (*heartbeat*) desde la instancia. Si no la recibe en este lapso, asume que la instancia cayó.
* **`max.poll.interval.ms`:** Tiempo máximo permitido entre llamadas consecutivas a la función de lectura (`poll()`). Si la lógica de negocio tarda demasiado en procesar un lote y supera este intervalo, el cliente se declara colgado o bloqueado y es expulsado del grupo.

### 3. Estrategias de Asignación (*Assignors*)
Existen dos enfoques principales para ejecutar la reasignación de particiones:
* **Eager Rebalance (*Stop-the-World*):** Se revocan temporalmente todas las particiones de todos los consumidores del grupo, pausando el procesamiento global hasta que se completa la nueva asignación.
* **Cooperative / Incremental Sticky Rebalance:** Solo se revocan y reasignan las particiones afectadas directamente por la alta o baja. Los consumidores no afectados continúan procesando sus particiones sin pausar el grupo.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Un rebalanceo siempre detiene por completo el procesamiento de todos los consumidores del grupo"* | Con las estrategias modernas (*Cooperative Sticky Assignor*), solo se pausan las particiones que cambian de dueño, permitiendo que el resto del grupo continúe operando sin interrupción total. |
| *"Si el hilo de heartbeat sigue enviando señales, el clúster nunca expulsará al consumidor"* | Si el código de negocio se bloquea o tarda demasiado en procesar registros superando el valor de `max.poll.interval.ms`, el cliente será expulsado del grupo aunque el hilo secundario de *heartbeat* siga respondiendo. |
| *"El broker asigna directamente las particiones a cada consumidor durante un rebalanceo"* | El clúster delega el cálculo de asignación a uno de los consumidores del grupo designado como **Consumer Leader**, quien calcula la distribución y devuelve el plan al broker coordinador para propagarlo. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué diferencia existe entre los parámetros `session.timeout.ms` y `max.poll.interval.ms`?</b></summary>

<p><b>Respuesta:</b> <code>session.timeout.ms</code> evalúa la conectividad de red y la presencia física del proceso mediante el envío de <i>heartbeats</i> periódicos, mientras que <code>max.poll.interval.ms</code> evalúa la salud de la aplicación monitoreando que el hilo principal siga solicitando y procesando lotes de datos activamente.</p>

</details>

<details>
<summary><b>2. ¿Qué ventaja ofrece la estrategia de rebalanceo Cooperativo sobre la estrategia tradicional Eager?</b></summary>

<p><b>Respuesta:</b> La estrategia Cooperativa minimiza el tiempo de inactividad, ya que evita detener la lectura en todo el grupo (<i>Stop-the-World</i>). Solo suspende y reasigna las particiones estrictamente necesarias que cambian de consumidor.</p>

</details>

<details>
<summary><b>3. ¿Qué ocurre si un consumidor tarda en procesar un lote de mensajes más tiempo del configurado en `max.poll.interval.ms`?</b></summary>

<p><b>Respuesta:</b> El cliente considerará que está bloqueado o degradado. Abandonará el grupo intencionalmente y el Coordinador del clúster iniciará un rebalanceo para reasignar sus particiones a otros miembros activos.</p>

</details>