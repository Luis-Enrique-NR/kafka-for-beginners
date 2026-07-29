# 📌 Factor de Replicación (Replication Factor)

> Es la propiedad que define cuántas copias idénticas de cada partición se almacenan a lo largo de distintos brokers en un clúster para garantizar la alta disponibilidad y la prevención de pérdida de datos.

---

## 💡 La Analogía del Mundo Real

Imagina que redactas un contrato legal sumamente crítico. Si conservas únicamente el documento original en un archivador, cualquier incendio en esa oficina destruirá la información para siempre. Para evitarlo, sacas 3 fotocopias exactas del contrato original y guardas cada una en cajas fuertes ubicadas en 3 sucursales bancarias distintas de la ciudad. Si una sucursal sufre un siniestro, la información sigue estando intacta y accesible en las otras dos.

---

## 🧩 Anatomía y Funcionamiento

El Factor de Replicación ($RF$) se establece a nivel de Topic al momento de su creación y determina el número total de copias físicas que existirá para cada una de sus particiones en el clúster.

```text
Topic: "pagos" (Particiones = 1, Replication Factor = 3)

[ Broker 1 ] -----------> [ Broker 2 ] -----------> [ Broker 3 ]
+-------------------+     +-------------------+     +-------------------+
| Partición 0       |     | Partición 0       |     | Partición 0       |
| (Réplica Líder)   |     | (Réplica Follower)|     | (Réplica Follower)|
+-------------------+     +-------------------+     +-------------------+
       |                         ^                         ^
       +--- Replicación de red --+-------------------------+
```

### 1. Tolerancia a Fallos ($N - 1$)
La regla básica de resiliencia en Kafka establece que, para tolerar la caída simultánea de $N$ brokers sin perder el acceso a una partición, se requiere un Factor de Replicación de $N + 1$. Por lo tanto, un Topic con $RF = 3$ puede resistir la falla catastrófica de hasta 2 brokers ($3 - 1 = 2$) de forma consecutiva o simultánea sin caer en estado *Offline*.

### 2. Distribución de Réplicas en los Brokers
Kafka asigna las réplicas de manera uniforme asegurando que dos copias de la misma partición nunca residan en el mismo nodo físico. De las $N$ réplicas creadas, una asume el rol de **Líder** (encargada de procesar las operaciones de entrada y salida) y las $RF - 1$ restantes asumen el rol de **Followers** (copias pasivas que replican los eventos del líder).

### 3. Impacto en Almacenamiento y Red
Aumentar el factor de replicación mejora la durabilidad, pero introduce compromisos de infraestructura:
* **Espacio en disco:** Un $RF = 3$ triplica los requerimientos de almacenamiento ($3 \times \text{tamaño de datos}$).
* **Ancho de banda de red:** Genera tráfico inter-broker adicional constante para mantener los datos sincronizados entre el Líder y los seguidores.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Si defino un Replication Factor de 3, la velocidad de lectura del Topic se triplica"* | El factor de replicación otorga **redundancia y alta disponibilidad**, no mayor paralelismo. Por defecto, todas las lecturas y escrituras son atendidas únicamente por la réplica Líder. |
| *"Puedo configurar un Replication Factor de 5 en un clúster que solo tiene 3 Brokers activos"* | Kafka **rechazará la creación del Topic** lanzando un error. El Factor de Replicación no puede superar el número total de brokers físicos disponibles en el clúster. |
| *"Aumentar el Factor de Replicación impacta el almacenamiento en disco pero no consume red"* | Cada réplica requiere transmitir de forma continua los eventos a través de la red interna del clúster hacia los brokers secundarios, incrementando el tráfico inter-nodo. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Cuál es el número máximo de brokers que pueden fallar en un Topic con RF = 3 sin perder la disponibilidad de la partición?</b></summary>

<p><b>Respuesta:</b> Pueden fallar hasta 2 brokers (3 - 1 = 2). Mientras al menos 1 réplica permanezca activa y en sincronía, la partición continuará operando.</p>

</details>

<details>
<summary><b>2. ¿Qué ocurre si ejecutas un comando para crear un Topic con RF = 4 en un clúster de 3 nodos?</b></summary>

<p><b>Respuesta:</b> La operación falla con un `InvalidReplicationFactorException`, ya que Kafka no puede asignar dos réplicas de la misma partición a un mismo broker.</p>

</details>

<details>
<summary><b>3. Si un Topic recibe 50 GB de datos crudos y tiene un RF = 3, ¿cuánto espacio total en disco consumirá el clúster?</b></summary>

<p><b>Respuesta:</b> Consumirá 150 GB de almacenamiento total en el clúster, ya que los 50 GB iniciales se almacenarán en el nodo Líder y se duplicarán en los 2 nodos Followers (50 GB × 3 = 150 GB).</p>

</details>