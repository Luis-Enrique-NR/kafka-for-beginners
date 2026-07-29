# 📌 Arquitectura KRaft (Kafka Raft Metadata Mode)

> Es el protocolo de consenso nativo basado en Raft que gestiona los metadatos del clúster de Kafka dentro de un registro interno, eliminando por completo la dependencia externa de Apache ZooKeeper.

---

## 💡 La Analogía del Mundo Real

Imagina un ayuntamiento municipal que antes contrataba a una empresa notarial externa (ZooKeeper) para validar cada registro de propiedad, cambio de directiva o libro contable. Cada trámite requería enviar mensajeros de un edificio a otro, lo que generaba demoras, costos de mantenimiento y cuellos de botella cuando la ciudad crecía. Con la arquitectura KRaft, el ayuntamiento elimina la empresa externa y nombra a un comité directivo interno electo entre los propios concejales (Controladores). Este comité mantiene la libreta de registros oficial dentro del mismo edificio utilizando un libro de actas compartido en tiempo real, haciendo la gestión instantánea, escalable y autosuficiente.

---

## 🧩 Anatomía y Funcionamiento

KRaft reemplaza la arquitectura tradicional de ZooKeeper almacenando la configuración y el estado global del clúster dentro de un topic interno especial de metadatos replicado mediante un protocolo de consenso adaptado.

```text
+-----------------------------------------------------------------------------------+
|                       Quorum KRaft                                                |
|                                                                                   |
|  [ Controller 1 ] <--- Active Leader (Escribe @metadata)                          |
|         |                                                                         |
|         +------------ (Replicación Raft) ------------------+                      |
|         v                                                  v                      |
|  [ Controller 2 ] (Follower)                        [ Controller 3 ] (Follower)   |
+-----------------------------------------------------------------------------------+
                               |
               (Propagación continua de metadatos)
                               v
+-------------------------------------------------------------+
|                        Data Brokers                         |
|   [ Broker 101 ]      [ Broker 102 ]      [ Broker 103 ]    |
+-------------------------------------------------------------+
```

### 1. El Quorum de Controladores (Controller Quorum)
En lugar de depender de un nodo controlador único coordinado por ZooKeeper, KRaft establece un conjunto dedicado de nodos llamados **Controllers**. Mediante votación basada en Raft, uno de ellos es elegido como el **Active Controller** (Líder del quorum), mientras que los demás actúan como seguidores (*Followers*) listos para asumir el control de inmediato si el activo falla.

### 2. El Registro de Metadatos (`@metadata`)
Toda la información sobre topics, particiones, asignaciones de réplicas, ACLs y configuraciones ya no reside en un árbol de datos jerárquico externo. Ahora se guarda como un registro de eventos inmutable dentro del particionamiento especial `@metadata`. El Active Controller escribe las modificaciones en este log y los demás controladores y Data Brokers lo consumen continuamente para actualizar su estado en memoria.

### 3. Recuperación Instantánea (*Snapshots* y Failover)
Dado que cada Data Broker mantiene una copia en memoria del estado de los metadatos leyendo el registro `@metadata`, no existe sobrecarga de red cuando un broker necesita consultar información del clúster. Si el Active Controller cae, la elección de un nuevo líder en el quorum toma milisegundos, permitiendo que el clúster escale a millones de particiones sin sufrir congelamientos.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"KRaft es un servicio o proceso independiente que debe instalarse por separado de Kafka"* | **KRaft está integrado de forma nativa en el binario de Kafka.** Un mismo nodo de Kafka se configura vía software para actuar como Controller, Data Broker o en modo combinado (*Combined Mode*). |
| *"KRaft guarda los metadatos en la memoria RAM y los pierde por completo si el clúster se apaga"* | Los metadatos se persisten en disco dentro del **Metadata Log (`@metadata`)** y se consolidan en archivos de captura (*Snapshots*) para permitir arranques e inicializaciones instantáneas. |
| *"ZooKeeper sigue siendo necesario para gestionar tareas administrativas avanzadas o seguridad"* | **ZooKeeper está totalmente obsoleto.** KRaft asume el 100% de las funciones operativas, la asignación de particiones, la seguridad (ACLs) y la gestión del ciclo de vida del clúster. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Cuál es la ventaja principal de almacenar los metadatos como un log de eventos en lugar de usarse una base de datos jerárquica externa?</b></summary>

<p><b>Respuesta:</b> Permite propagar los cambios de configuración mediante lecturas secuenciales ultrarrápidas a todos los nodos, eliminando las consultas remotas constantes y reduciendo el tiempo de conmutación por error (failover) a milisegundos.</p>

</details>

<details>
<summary><b>2. ¿Qué roles de nodo se pueden configurar en un clúster de Kafka bajo la arquitectura KRaft?</b></summary>

<p><b>Respuesta:</b> Se pueden configurar los roles <code>broker</code> (procesa eventos del usuario), <code>controller</code> (gestiona el quorum de metadatos) o <code>combined</code> (un solo nodo asume ambos roles, ideal para entornos de desarrollo o clústeres pequeños).</p>

</details>

<details>
<summary><b>3. ¿Qué ocurre dentro del Quorum KRaft cuando la réplica Active Controller sufre un fallo de hardware?</b></summary>

<p><b>Respuesta:</b> Los controladores seguidores (Followers) detectan la ausencia de respuesta e inician de forma automática una votación bajo el protocolo Raft para promocionar a un nuevo Active Controller en cuestión de milisegundos.</p>

</details>