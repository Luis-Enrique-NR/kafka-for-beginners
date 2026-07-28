# 📌 Consumer Groups

> Un Consumer Group es un conjunto de instancias consumidoras coordinadas que comparten la lectura de un topic para escalar el procesamiento horizontalmente y proveer tolerancia a fallos.

---

## 💡 La Analogía del Mundo Real

Imagina la zona de cajas de cobro en un supermercado concurrido. Los clientes que llegan con sus carritos llenos representan el flujo ininterrumpido de mensajes en un topic. 

Si solo existiera **un cajero** (un consumidor individual), se formaría un cuello de botella crítico y los clientes tardarían horas en ser atendidos. Para resolverlo, la administración abre un **"Grupo de Cajas"** (Consumer Group). Ahora, la fila total de clientes se distribuye entre las distintas cajas abiertas. Si una caja se satura, el trabajo se reparte entre las demás; si un cajero debe retirarse repentinamente, otro cajero del grupo asume su turno para continuar atendiendo sin perder el control de la fila.

---

## 🧩 Anatomía y Funcionamiento

Dentro de un clúster, los consumidores se organizan mediante un identificador único. El clúster coordina automáticamente la asignación de particiones para asegurar que el trabajo se distribuya de forma equitativa y sin duplicaciones dentro del mismo grupo.

```text
Topic: ordenes-compra (3 Particiones)

┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Partición 0  │   │ Partición 1  │   │ Partición 2  │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Consumidor A │   │ Consumidor B │   │ Consumidor C │   [Consumidor D]
└──────────────┴───┴──────────────┴───┴──────────────┴─────────┬────────┘
                             Consumer Group: "servicio-facturacion"    │
                                                                       └─── (Estado Idle / Espera)
```

### 1. Identificador de Grupo (`group.id`)
El parámetro `group.id` define la pertenencia de una instancia de consumo a un grupo lógico. Cuando múltiples instancias comparten el mismo `group.id`, operan en modo de **balanceo de carga** (cola de trabajo). Si las instancias tienen identificadores de `group.id` diferentes, cada grupo recibirá una copia completa e independiente de todos los eventos del topic (patrón **Publish-Subscribe**).

### 2. Escalabilidad y el Límite de Particiones
La capacidad máxima de escalamiento horizontal paralelo de un Consumer Group está delimitada estrictamente por el número de particiones del topic. 
* Si $N_{\text{consumidores}} = N_{\text{particiones}}$, cada consumidor procesa exactamente una partición (escenario ideal).
* Si $N_{\text{consumidores}} > N_{\text{particiones}}$, las instancias excedentes quedarán en estado inactivo o en reserva (**Idle**). No recibirán datos a menos que una instancia activa falle o se desconecte.

### 3. Rebalanceo Automático
Cuando una nueva instancia se une al grupo o una existente se desconecta (por fallo, mantenimiento o latencia excesiva), el clúster ejecuta un proceso de **Rebalanceo** (*Rebalance*). Durante este evento, las particiones se redistribuyen dinámicamente entre las instancias activas disponibles para garantizar la continuidad del servicio.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Agregar más consumidores a un grupo siempre aumenta la velocidad de procesamiento"* | La capacidad de lectura paralela está limitada por el número de particiones. Las instancias que superen dicho número quedarán inactivas en estado *Idle*. |
| *"Dos grupos de consumo distintos se dividen las particiones de un mismo topic"* | Cada Consumer Group mantiene su propio seguimiento de lectura independiente. Múltiples grupos leen la totalidad de los datos del topic sin interferir entre sí. |
| *"Una partición puede ser procesada en paralelo por varios consumidores del mismo grupo"* | Una partición solo puede estar asignada a **un único consumidor** dentro del mismo grupo para garantizar el orden estricto de los eventos registrados. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué ocurre con una instancia si se une a un grupo que ya tiene un consumidor asignado a cada partición del topic?</b></summary>

<p><b>Respuesta:</b> La instancia quedará en estado de reposo o espera (<i>Idle</i>). No recibirá mensajes hasta que ocurra un rebalanceo provocado por la caída de una instancia activa o por el incremento en el número de particiones del topic.</p>

</details>

<details>
<summary><b>2. Si dos microservicios independientes (ej. Facturación y Notificaciones) necesitan consumir el mismo topic, ¿deben usar el mismo `group.id`?</b></summary>

<p><b>Respuesta:</b> No. Deben utilizar valores de <code>group.id</code> distintos. De esta manera, cada microservicio actuará como un Consumer Group independiente y recibirá su propia copia completa de todos los mensajes del topic.</p>

</details>

<details>
<summary><b>3. ¿Por qué se prohíbe que dos consumidores del mismo grupo lean en paralelo de una misma partición?</b></summary>

<p><b>Respuesta:</b> Para preservar el orden cronológico de los eventos. Al asignar la partición a un único consumidor dentro del grupo, se garantiza que los registros se procesen en la misma secuencia exacta en la que fueron escritos.</p>

</details>