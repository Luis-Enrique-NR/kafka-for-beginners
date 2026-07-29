# 🛡️ Laboratorio 03: Alta Disponibilidad, Resiliencia y Chaos Engineering en TechStore

## 📌 Contexto de la Práctica

El procesador de pagos de **TechStore** ha pasado a atender transacciones financieras críticas (`pagos-criticos`). En este escenario, depender de un solo broker como en las prácticas anteriores representa un punto único de falla (*Single Point of Failure*). Si el nodo cae, la pasarela de pagos colapsa.

En esta práctica evolucionaremos hacia un clúster **Multi-Broker nativo en modo KRaft (3 nodos)** y aprenderás a:
1. **Auditar la Topología KRaft:** Comprender la arquitectura sin ZooKeeper y verificar quórums de metadatos.
2. **Gestionar la Replicación:** Configurar el Factor de Replicación ($RF=3$), identificando Líderes, Followers e **ISR** (*In-Sync Replicas*).
3. **Ejecutar Chaos Engineering:** Simular caídas violentas de brokers (`docker stop`) y presenciar la reelección de líder en tiempo real sin interrupción de servicio.
4. **Implementar Resiliencia Extrema:** Evaluar el "Combo de Oro" de producción (`acks=all` + `min.insync.replicas=2`) y entender el compromiso entre disponibilidad y consistencia estricta.

---

## 🛠️ Requisitos Previos

1. Abre tu consola **CMD de Windows**, navega a la carpeta de este laboratorio y despliega el clúster:

   ```cmd
   cd laboratorios\03-alta-disponibilidad-y-resiliencia
   docker compose up -d
   ```

2. Verifica que los 3 brokers de Kafka y la consola web estén activos y saludables:

   ```cmd
   docker ps
   ```

---

## 🎯 Misión 1: Inspección de la Topología KRaft (Clúster Multi-Nodo)

**Objetivo:** Auditar la arquitectura moderna de Apache Kafka en modo KRaft, verificando los miembros activos del clúster y el estado del quórum de metadatos.

### 1. Consultar los Nodos Activos del Clúster
Ejecuta el script oficial de inspección de APIs de Kafka apuntando a `kafka-1` desde tu **CMD**:

```cmd
docker exec -it kafka-1 /opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```
* **Observación:** La salida listará los 3 brokers registrados (`kafka-1:9092`, `kafka-2:9092`, `kafka-3:9092`) confirmando que la red inter-broker se ha establecido correctamente.

### 2. Inspeccionar el Quórum de Controladores KRaft
En la arquitectura KRaft, la gestión de metadatos la realiza un quórum de nodos en lugar de un clúster ZooKeeper externo. Consulta el estado del quórum:

```cmd
docker exec -it kafka-1 /opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status
```
* **Observación:** Verás qué broker ostenta actualmente el rol de **Leader Controller** (encargado de orquestar el clúster) y cuáles actúan como **Followers** dentro del quórum.

### 3. Verificación Visual en Kafka UI
1. Abre tu navegador e ingresa a `http://localhost:8080`.
2. En el panel lateral, selecciona el clúster **techstore-kraft-cluster**.
3. Dirígete a la sección **Brokers** y confirma que se muestren los 3 nodos con sus IDs (1, 2 y 3) en estado *Online*.

---

## 🎯 Misión 2: Anatomía de la Replicación (Líderes, Followers e ISR)

**Objetivo:** Crear un topic con replicación completa en todos los nodos y analizar la distribución de responsabilidades entre los brokers.

### 1. Crear el Topic Replicado (`pagos-criticos`)
Crea un topic transaccional con **3 particiones** y un **Factor de Replicación de 3** ($RF=3$):

```cmd
docker exec -it kafka-1 /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic pagos-criticos --partitions 3 --replication-factor 3
```

### 2. Describir la Anatomía del Topic
Inspecciona la distribución física de las particiones en el clúster:

```cmd
docker exec -it kafka-1 /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic pagos-criticos
```

Analicemos un ejemplo de la salida resultante:

```text
Topic: pagos-criticos   PartitionCount: 3   ReplicationFactor: 3
    Topic: pagos-criticos   Partition: 0    Leader: X   Replicas: 1,2,3 Isr: 1,2,3
    Topic: pagos-criticos   Partition: 1    Leader: Y   Replicas: 2,3,1 Isr: 2,3,1
    Topic: pagos-criticos   Partition: 2    Leader: Z   Replicas: 3,1,2 Isr: 3,1,2
```

> ⚠️ **Nota Importante:** Kafka distribuye los líderes de las particiones de forma dinámica al crear el topic. Revisa la salida de **tu terminal** para identificar qué número de broker aparece en la columna `Leader` para la `Partition: 0` (puede ser el `1`, `2` o `3`).

#### 📖 ¿Qué significa cada columna?
* **Partition:** El identificador numérico de la partición (0, 1 o 2).
* **Leader:** El único broker encargado de procesar **todas las lecturas y escrituras** de esa partición.
* **Replicas:** La lista completa de brokers que almacenan una copia del registro de eventos.
* **ISR (*In-Sync Replicas*):** El grupo de réplicas que están **100% al día** con el líder. Si una réplica se retrasa por red o disco, es removida del ISR.

---

## 🎯 Misión 3: Chaos Engineering I — Caída Forzada del Líder (`docker stop`)

**Objetivo:** Simular un fallo físico o caída de servidor en producción para observar la reelección automática de líder y la contracción del grupo ISR en tiempo real.

---

### 1. Iniciar un Consumidor de Eventos en Background (Terminal A)
Abre una **Terminal A** (CMD) para monitorear en tiempo real los mensajes que llegarán al topic:

```cmd
cd laboratorios\03-alta-disponibilidad-y-resiliencia
docker exec -it kafka-1 /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic pagos-criticos --from-beginning
```

### 2. Transmitir un Flujo de Eventos (Terminal B)
Abre una **Terminal B** (CMD) y envía un paquete inicial de 20 eventos con acuse de recibo total (`acks=all`):

```cmd
cd laboratorios\03-alta-disponibilidad-y-resiliencia
(for /L %i in (1,1,20) do @echo pago-%i:{"id": %i, "monto": 150}) | docker exec -i kafka-1 /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic pagos-criticos --property "parse.key=true" --property "key.separator=:" --producer-property acks=all
```

### 3. Identificar y Asesinar al Líder de la Partición 0
Revisa la salida del comando `--describe` ejecutado en la Misión 2 y determina qué broker es el **Líder de la Partición 0**:
* Si tu terminal indicó `Partition: 0 Leader: 1` $\rightarrow$ Tu objetivo es `kafka-1`.
* Si tu terminal indicó `Partition: 0 Leader: 2` $\rightarrow$ Tu objetivo es `kafka-2`.
* Si tu terminal indicó `Partition: 0 Leader: 3` $\rightarrow$ Tu objetivo es `kafka-3`.

Desde una terminal CMD libre, simula la caída violenta de ese servidor apagando su contenedor (reemplaza `kafka-X` por tu broker líder real, por ejemplo `docker stop kafka-2`):

```cmd
docker stop kafka-X
```

### 4. Observar la Elección de Nuevo Líder y Contracción de ISR
Inmediatamente tras detener el nodo, consulta el estado del topic ejecutando el comando **en cualquiera de los brokers supervivientes** (`kafka-Y`, donde $Y \neq X$):

```cmd
docker exec -it kafka-Y /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic pagos-criticos
```
*(Ejemplo: si apagaste `kafka-2`, ejecuta la consulta apuntando a `kafka-1` o `kafka-3`)*

* **✅ Criterio de Aceptación:** 
  Verás que la Partición 0 **sigue operativa**, pero su estado se ha actualizado automáticamente:
  * **Leader:** El liderazgo de la Partición 0 se reasignó exitosamente a uno de los nodos supervivientes.
  * **ISR:** El grupo de réplicas sincronizadas de las particiones se redujo automáticamente a **2 nodos** (el broker apagado fue removido del grupo de sincronización).

### 5. Verificar Tolerancia a Fallos enviando Nuevos Eventos
Vuelve a la **Terminal B** y envía 10 eventos adicionales conectándote al clúster a través de **cualquiera de los contenedores activos** (`kafka-Y`):

```cmd
(for /L %i in (21,1,30) do @echo pago-%i:{"id": %i, "monto": 200}) | docker exec -i kafka-Y /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic pagos-criticos --property "parse.key=true" --property "key.separator=:" --producer-property acks=all
```
*(Reemplaza `kafka-Y` por el nombre de un contenedor vivo, por ejemplo `kafka-3`)*

* **Observación:** El envío se completa con éxito. Los brokers supervivientes aceptan el tráfico y reenrutan las escrituras hacia los nuevos líderes. El sistema sobrevivió a la destrucción de un nodo.

### 6. Recuperación y Sincronización Automática (*Catch-up*)
Reanima el nodo caído que apagaste en el Paso 3 (`kafka-X`):

```cmd
docker start kafka-X
```

Espera 5 segundos y vuelve a describir el topic preguntando a cualquier broker (`kafka-topics.sh --describe`). Notarás que el broker resucitado recuperó los datos faltantes y se reincorporó automáticamente al grupo **ISR** (volviendo a tener los 3 nodos en sincronía).

---

## 🎯 Misión 4: Chaos Engineering II — El "Combo de Oro" (`acks=all` + `min.insync.replicas`)

**Objetivo:** Probar el límite de resiliencia del clúster cuando priorizamos la **consistencia absoluta sobre la disponibilidad**.

---

### 💡 ¿Qué es `min.insync.replicas`?

Por defecto, si un productor envía datos con `acks=all`, el broker líder aceptará el mensaje siempre que al menos **el propio líder** lo guarde. Si 2 réplicas están caídas, el mensaje se acepta pero **pierde redundancia**.

Para garantizar durabilidad en sistemas bancarios, se combina:
* **`acks=all` (Productor):** *"No me des OK hasta que todas las réplicas del ISR hayan confirmado"*.
* **`min.insync.replicas=2` (Topic):** *"Rechaza cualquier escritura si el número de réplicas vivas y sincronizadas (ISR) es menor a 2"*.

---

### 1. Configurar la Regla Estricta en el Topic
Aplica la regla de producción `min.insync.replicas=2` al topic `pagos-criticos`:

```cmd
docker exec -it kafka-2 /opt/kafka/bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name pagos-criticos --alter --add-config min.insync.replicas=2
```

### 2. Prueba con 1 Broker Caído (Operación Permitida)
Simula la caída de un broker (`kafka-3`):

```cmd
docker stop kafka-3
```

Con `kafka-3` apagado, el grupo ISR del topic cuenta con **2 nodos vivos** (`kafka-1` y `kafka-2`). Intenta enviar un mensaje exigiendo confirmación total (`acks=all`):

```cmd
echo pago-seguro:{"monto": 500} | docker exec -i kafka-1 /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic pagos-criticos --producer-property acks=all
```
* **Resultado:** el mensaje se procesa con **éxito** porque $ISR (2) \ge min.insync.replicas (2)$.

---

### 3. Simular Falla Catastrófica (Doble Caída)
Apaga un **segundo broker** (`kafka-2`), dejando con vida únicamente a `kafka-1`:

```cmd
docker stop kafka-2
```

Ahora el topic tiene solo **1 nodo vivo** en su ISR ($ISR = 1$).

---

### 4. Intentar Escribir Bajo Falla Catastrófica (Rechazo Garantizado)
Intenta nuevamente enviar un evento exigiendo `acks=all`:

```cmd
echo pago-critico:{"monto": 999} | docker exec -i kafka-1 /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic pagos-criticos --producer-property acks=all
```

* **✅ Criterio de Aceptación:**  
  La consola arrojará una excepción explícita de Kafka:
  ```text
  org.apache.kafka.common.errors.NotEnoughReplicasException: Messages are rejected since there are fewer in-sync replicas than required.
  ```

> 🧠 **Lección de Arquitectura:** Kafka prefirió **rechazar la transacción** antes que aceptar un pago financiero sin la garantía de que estuviera replicado en al menos 2 servidores.

---

### 5. Restaurar el Clúster
Levanta los nodos apagados para devolver la salud al clúster:

```cmd
docker start kafka-2
docker start kafka-3
```

Si reintentas el comando de envío, verás que las escrituras se desbloquean inmediatamente tan pronto los nodos se reincorporan al ISR.

---

## 💡 Resumen de Validación Práctica

| Escenario | Nodos Vivos | ISR | `min.insync.replicas` | Resultado con `acks=all` |
| :--- | :---: | :---: | :---: | :--- |
| **Estado Normal** | 3 / 3 | `[1, 2, 3]` | 2 | ✅ **Escritura Exitosa** (Redundancia total) |
| **Caída de 1 Broker** | 2 / 3 | `[1, 2]` | 2 | ✅ **Escritura Exitosa** (Cumple quórum mínimo) |
| **Caída de 2 Brokers** | 1 / 3 | `[1]` | 2 | ❌ **Rechazo (`NotEnoughReplicasException`)** |

---

## 🧹 Limpieza

Para detener los consumidores, eliminar los datos persistidos y apagar el entorno desde CMD:

```cmd
docker compose down -v
```

---

## 📚 Anexos: Fundamentos Teóricos Relacionados

Para profundizar en los conceptos de alta disponibilidad, replicación y resiliencia demostrados en esta práctica:

* 🏛️ **Arquitectura KRaft:** [kraft-architecture.md](../../teoria/03-ecosistema-avanzado/kraft-architecture.md)
* 🔁 **Factor de Replicación:** [replication-factor.md](../../teoria/02-mecanismos-internos/replication-factor.md)
* 👑 **Líder y Seguidores:** [leader-and-followers.md](../../teoria/02-mecanismos-internos/leader-and-followers.md)
* 🛡️ **Réplicas en Sincronía (ISR):** [in-sync-replicas-isr.md](../../teoria/02-mecanismos-internos/in-sync-replicas-isr.md)
* 🔒 **Mínimo de Réplicas en Sincronía:** [min-insync-replicas.md](../../teoria/02-mecanismos-internos/min-insync-replicas.md)
* ⚡ **Elección de Líder y Failover:** [leader-election-and-failover.md](../../teoria/02-mecanismos-internos/leader-election-and-failover.md)
* 🧪 **Ingeniería del Caos y Resiliencia:** [chaos-engineering-and-resilience.md](../../teoria/03-ecosistema-avanzado/chaos-engineering-and-resilience.md)