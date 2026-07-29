# 📌 Réplicas en Sincronía (In-Sync Replicas - ISR)

> Es el conjunto dinámico de réplicas de una partición que se encuentran completamente al día con la réplica Líder y listas para asumir el liderazgo en caso de fallo sin pérdida de datos.

---

## 💡 La Analogía del Mundo Real

Imagina un equipo de taquógrafos en un juicio rápido. El juez (Líder) dicta la sentencia a alta velocidad mientras 3 taquógrafos (Réplicas) escriben cada palabra en tiempo real. Dos de los taquógrafos escriben al mismo ritmo que el juez y no pierden ni una sola palabra. El tercer taquígrafo rompe la punta de su lápiz, deja de escribir por 2 minutos y se queda atrasado por varias páginas. Mientras ese tercer taquígrafo esté atrasado, no forma parte del "grupo oficial de actas al día" (ISR). Solo los dos taquógrafos que siguen el ritmo exacto del juez tienen la facultad de reemplazarlo si este debe ausentarse.

---

## 🧩 Anatomía y Funcionamiento

Dentro de Kafka, el Factor de Replicación ($RF$) define cuántas copias existen en total, pero el grupo **ISR** define cuántas de esas copias están **operativas y actualizadas en tiempo real**.

```text
Partición 0 (Líder: Broker 1 | Offset Máximo: 100)

[ Broker 1 (Líder) ]    --> Offset 100  --> [ DENTRO DEL ISR ]
[ Broker 2 (Follower) ] --> Offset 100  --> [ DENTRO DEL ISR ] (Al día)
[ Broker 3 (Follower) ] --> Offset 40   --> [ FUERA DEL ISR  ] (Lag / Desconectado)

Grupo ISR Actual = [Broker 1, Broker 2]
```

### 1. El Criterio de Sincronización (`replica.lag.time.max.ms`)
Un follower se considera "en sincronía" si envía peticiones continuas de extracción (*Fetch Requests*) al Líder y se mantiene al día con el registro. Si un follower deja de solicitar datos o se retrasa en ponerse al día durante un tiempo superior al configurado en el parámetro `replica.lag.time.max.ms` (por defecto 30 segundos), el Líder lo expulsa automáticamente del grupo ISR.

### 2. Dinamismo del Grupo (Contracción y Expansión)
El grupo ISR es completamente elástico:
* **Contracción:** Ocurre cuando un broker cae, sufre problemas de red o experimenta pausas largas de recolección de basura (*Garbage Collection*). El tamaño del ISR se reduce automáticamente.
* **Expansión (*Catch-up*):** Cuando el broker afectado se recupera, reanuda la copia de los eventos faltantes desde donde se quedó. Tan pronto alcanza el último offset del Líder, se reincorpora automáticamente al ISR.

### 3. Garantía de Elección de Líder Segura
Cuando la réplica Líder de una partición falla, Kafka únicamente elegirá como nuevo Líder a una réplica que pertenezca al grupo ISR. Esto garantiza de forma estricta que ningún mensaje confirmado previamente se pierda durante la transición.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"El grupo ISR siempre incluye a todos los brokers especificados en el Replication Factor"* | El grupo ISR **varía dinámicamente**. Si un broker de los 3 replicados se apaga o sufre latencia, el ISR se reduce temporalmente a 2 miembros mientras el Factor de Replicación sigue siendo 3. |
| *"Si una réplica es expulsada del ISR, pierde todos los datos guardados en su disco"* | La réplica conserva sus datos intactos. Al reconectarse, solo realiza un proceso de pases cortos (*Catch-up*) para descargar los mensajes emitidos durante su ausencia. |
| *"El parámetro `replica.lag.max.messages` sigue siendo la forma principal de expulsar réplicas lentas"* | Ese parámetro fue eliminado en versiones modernas de Kafka por generar falsos positivos en picos de tráfico. Hoy en día la expulsión se evalúa exclusivamente por tiempo (`replica.lag.time.max.ms`). |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué sucede con el grupo ISR de un Topic con RF = 3 si uno de sus tres brokers se apaga de forma repentina?</b></summary>

<p><b>Respuesta:</b> Tras expirar el tiempo de tolerancia (<code>replica.lag.time.max.ms</code>), el Líder expulsa al broker caído del grupo ISR, reduciendo la lista de réplicas en sincronía de 3 a 2 nodos activos.</p>

</details>

<details>
<summary><b>2. ¿Por qué Kafka restringe la reelección de Líder únicamente a las réplicas presentes en el grupo ISR?</b></summary>

<p><b>Respuesta:</b> Para prevenir la pérdida de datos (<i>Data Loss</i>), asegurando que el nuevo Líder designado posea exactamente la misma información confirmada que tenía el Líder anterior antes de fallar.</p>

</details>

<details>
<summary><b>3. ¿Qué proceso realiza una réplica cuando vuelve a iniciar tras haber estado fuera del ISR por varias horas?</b></summary>

<p><b>Respuesta:</b> Inicia un proceso de recuperación (<i>Catch-up</i>) solicitando únicamente los deltas u offsets faltantes desde su último registro guardado hasta igualar al Líder actual, momento en el cual reingresa automáticamente al ISR.</p>

</details>