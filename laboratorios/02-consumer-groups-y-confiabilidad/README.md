# 🚀 Laboratorio 02: Escalabilidad con Consumer Groups y Confiabilidad en TechStore

## 📌 Contexto de la Práctica

El evento de **Black Friday** ha llegado a **TechStore**. El volumen de transacciones se ha multiplicado exponencialmente y el sistema monoconsumidor que configuramos en la Misión 01 ya no es suficiente. 

En esta práctica aprenderás a:
1. **Escalar el procesamiento:** Distribuir la lectura de eventos entre múltiples instancias de consumo de forma automática mediante **Consumer Groups**.
2. **Probar la Confiabilidad del Productor:** Evaluar el impacto en velocidad y persistencia al cambiar los niveles de acuse de recibo (`acks`).
3. **Simular Fallos y Semánticas de Entrega:** Experimentar en la práctica la semántica **At-least-once** y cómo un fallo antes del *commit* de *Offsets* provoca relecturas y duplicados.

---

## 🛠️ Requisitos Previos

1. Asegúrate de tener la infraestructura levantada ingresando a la carpeta de este laboratorio desde tu **CMD**:

   ```cmd
   cd laboratorios\02-consumer-groups-y-confiabilidad
   docker compose up -d
   ```

2. Verifica que el contenedor de Redpanda esté saludable:

   ```cmd
   docker ps
   ```

---

## 🎯 Misión 1: Preparar el Topic y Generar Tráfico Automático

**Objetivo:** Aprovisionar un topic con **3 particiones** y ejecutar la generación automática de eventos para nuestras pruebas.

### 1. Crear el Topic
Ejecuta en tu terminal CMD el comando de administración de Redpanda (`rpk`):

```cmd
docker exec -it redpanda-0 rpk topic create ordenes-blackfriday -p 3
```

### 2. Generar eventos en masa

Elige la alternativa que prefieras desde tu **CMD de Windows**:

* **Opción A: Ráfaga rápida directamente desde CMD (Recomendado)**
  Copia y pega esta ráfaga de 30 eventos en tu consola CMD:

  ```cmd
  (for /L %i in (1,1,30) do @echo cliente-%i:{"id": %i, "monto": 50}) | docker exec -i redpanda-0 rpk topic produce ordenes-blackfriday --format "%k:%v\n"
  ```

* **Opción B: Entrando a la consola Bash interactiva de Redpanda**
  1. Entra a la consola del contenedor:
     ```cmd
     docker exec -it redpanda-0 bash
     ```
  2. Pega el bucle de envío Bash:
     ```bash
     for i in $(seq 1 30); do
       CLIENTE_ID=$(( (i % 3) + 1 ))
       KEY="cliente-00${CLIENTE_ID}"
       VALUE="{\"id\": $i, \"monto\": $(( (RANDOM % 100) + 10 )), \"timestamp\": $(date +%s)}"
       echo "$KEY:$VALUE" | rpk topic produce ordenes-blackfriday --format "%k:%v\n"
       sleep 0.1
     done
     ```
  3. Sal del contenedor para continuar con las siguientes misiones:
     ```bash
     exit
     ```

---

## 🎯 Misión 2: Escalamiento y Rebalanceo (Consumer Groups)

**Objetivo:** Observar en tiempo real cómo Kafka redistribuye las particiones a medida que agregamos o quitamos consumidores a un mismo grupo (`group.id`).

### 1. Iniciar el Primer Consumidor
Abre tu **Terminal 1** (CMD) y ejecuta:

```cmd
cd laboratorios\02-consumer-groups-y-confiabilidad
docker exec -it redpanda-0 rpk topic consume ordenes-blackfriday --group grupo-facturacion
```
* **Observación:** Al ser el único integrante del grupo `grupo-facturacion`, este consumidor tomará el control de **las 3 particiones** (0, 1 y 2).

### 2. Unir un Segundo Consumidor
Abre una **Terminal 2** (paralela en CMD) y ejecuta:

```cmd
cd laboratorios\02-consumer-groups-y-confiabilidad
docker exec -it redpanda-0 rpk topic consume ordenes-blackfriday --group grupo-facturacion
```
* **Observación:** Kafka ejecutará un **Rebalanceo**. Si ejecutas de nuevo el envío de eventos de la Misión 1, verás que los nuevos eventos se reparten entre ambas terminales.

### 3. Unir un Tercer Consumidor (Saturación Ideal)
Abre una **Terminal 3** (CMD) y ejecuta:

```cmd
cd laboratorios\02-consumer-groups-y-confiabilidad
docker exec -it redpanda-0 rpk topic consume ordenes-blackfriday --group grupo-facturacion
```
* **Observación:** El grupo alcanza su capacidad máxima balanceada: cada terminal atiende exactamente **1 partición**.

### 4. Probar un Consumidor en Reserva (Idle)
Abre una **Terminal 4** (CMD) y ejecuta:

```cmd
cd laboratorios\02-consumer-groups-y-confiabilidad
docker exec -it redpanda-0 rpk topic consume ordenes-blackfriday --group grupo-facturacion
```
* **Observación:** Al haber 3 particiones y 4 consumidores dentro del mismo grupo, este cuarto integrante quedará en estado de espera (*Idle*).

---

### 🔍 5. Herramientas de Diagnóstico e Inspección

En cualquier momento puedes abrir una terminal CMD libre para auditar el estado interno del grupo de consumo:

* **Describir la distribución del grupo:**
  ```cmd
  docker exec -it redpanda-0 rpk group describe grupo-facturacion
  ```
  *(Verás la columna `MEMBER-ID` asignada a cada partición y el conteo de `MEMBERS` activos).*

* **Listar todos los Consumer Groups activos:**
  ```cmd
  docker exec -it redpanda-0 rpk group list
  ```

> 💡 **Tip de Solución de Problemas:** Si cierras una terminal abruptamente en Windows y el clúster tarda en detectar la desconexión manteniendo el miembro como activo, puedes limpiar las conexiones de golpe ejecutando `docker restart redpanda-0`.

---

### 6. Simular Caída y Rebalanceo Automático

1. **Simular fallo:** Detén el consumidor activo de la **Terminal 1** presionando `Ctrl + C` o cerrando esa pestaña.
2. **Regenerar tráfico:** Abre tu terminal libre y **vuelve a enviar tráfico de la Misión 1 (Paso 2)** para transmitir un nuevo paquete de 30 mensajes.
3. **Observar la toma de relevo:** Ve inmediatamente a la **Terminal 4**, la cual anteriormente estaba congelada/en reposo (*Idle*).

* **✅ Criterio de Aceptación:** Verás que la **Terminal 4** empieza a recibir los mensajes de la partición huérfana en tiempo real. Si ejecutas el diagnóstico (`rpk group describe grupo-facturacion`), confirmarás que la partición fue reasignada oficialmente a un miembro activo.

---

## 🎯 Misión 3: Acuses de Recibo del Productor (`acks`) y Performance

**Objetivo:** Comprender y medir el balance entre **velocidad (latencia)** y **durabilidad (garantía de entrega)** ajustando el nivel de acuse de recibo (`acks`) en los productores.

---

### 💡 ¿Qué es `acks` y por qué es tan importante?

Cuando un productor envía datos a Kafka/Redpanda, debe decidir si esperará una confirmación del servidor antes de enviar el siguiente mensaje:

* **`acks=0` (Sin confirmación / Modo "Dispara y Olvida"):** El productor envía los datos y NO espera respuesta. Es ultra rápido y ofrece la **menor latencia**, pero si el broker falla, los datos se pierden. *(Ideal para: sensores IoT, métricas, logs).*
* **`acks=-1` o `acks=all` (Confirmación Total / Modo Seguro):** El productor espera a que el broker confirme que guardó el mensaje en disco y lo replicó. Ofrece **máxima durabilidad**, pero añade **mayor tiempo de espera (latencia)** por mensaje. *(Ideal para: transferencias bancarias, compras).*

---

### 1. Pruebas de Envío Individual en CMD (Sintaxis Windows)

Probemos primero el comportamiento enviando un solo evento a la vez. En CMD de Windows usamos cadenas simples sin escapar comillas dobles:

* **Enviar sin esperar confirmación (`acks=0`):**
  ```cmd
  echo cliente-001:{"id": 999, "monto": 500}| docker exec -i redpanda-0 rpk topic produce ordenes-blackfriday --format "%k:%v\n" --acks 0
  ```

* **Enviar exigiendo confirmación total (`acks=-1`):**
  ```cmd
  echo cliente-001:{"id": 1000, "monto": 750}| docker exec -i redpanda-0 rpk topic produce ordenes-blackfriday --format "%k:%v\n" --acks -1
  ```

---

### 📊 2. Tangibilizar el Impacto en Métricas y Rendimiento

Elige la alternativa que prefieras para medir y comparar el rendimiento real:

#### ⚡ Opción A: Prueba Rápida con `rpk` en CMD (Sin descargas)
Copia y pega estos comandos directamente en tu **CMD**. Transmitirán 1,000 eventos en *streaming*. Nota el tiempo en que la consola tarda en responder:

* **Ráfaga con `acks=0` (Respuesta inmediata):**
  ```cmd
  (for /L %i in (1,1,1000) do @echo cliente-%i:{"id": %i, "monto": 100}) | docker exec -i redpanda-0 rpk topic produce ordenes-blackfriday --format "%k:%v\n" --acks 0
  ```

* **Ráfaga con `acks=-1` (Espera confirmación de persistencia):**
  ```cmd
  (for /L %i in (1,1,1000) do @echo cliente-%i:{"id": %i, "monto": 100}) | docker exec -i redpanda-0 rpk topic produce ordenes-blackfriday --format "%k:%v\n" --acks -1
  ```

---

#### 🛠️ Opción B: Benchmark Oficial de Kafka (`kafka-producer-perf-test`)

Para obtener métricas profesionales ($p_{50}$, $p_{99}$, latencia promedio y `records/sec`), utilizaremos la herramienta oficial de Apache Kafka corriendo dentro de Docker en la red de Redpanda (`--network container:redpanda-0`).

> ⚠️ **Fase 0: Calentamiento de la JVM (Warm-up Package)**  
> La Máquina Virtual de Java (JVM) y los buffers de red tardan unos milisegundos en inicializarse la primera vez que se ejecutan. Para **evitar métricas distorsionadas** en nuestra prueba oficial, ejecutaremos primero un paquete pequeño de prueba para "calentar" el entorno:

```cmd
:: PAQUETE DE CALENTAMIENTO (Ejecútalo una vez para preparar el entorno)
docker run --rm --network container:redpanda-0 apache/kafka:latest /opt/kafka/bin/kafka-producer-perf-test.sh --topic ordenes-blackfriday --num-records 5000 --record-size 100 --throughput -1 --producer-props bootstrap.servers=localhost:9092 acks=0
```

---

#### 🚀 Pruebas Oficiales de Carga Masiva (100,000 Eventos)

Una vez realizado el calentamiento, ejecuta las pruebas oficiales de 100,000 registros para observar la diferencia estadística real:

* **Prueba 1: Benchmark con `acks=0` (Máxima velocidad de respuesta):**
  ```cmd
  docker run --rm --network container:redpanda-0 apache/kafka:latest /opt/kafka/bin/kafka-producer-perf-test.sh --topic ordenes-blackfriday --num-records 100000 --record-size 100 --throughput -1 --producer-props bootstrap.servers=localhost:9092 acks=0
  ```

* **Prueba 2: Benchmark con `acks=all` (Garantía de durabilidad total):**
  ```cmd
  docker run --rm --network container:redpanda-0 apache/kafka:latest /opt/kafka/bin/kafka-producer-perf-test.sh --topic ordenes-blackfriday --num-records 100000 --record-size 100 --throughput -1 --producer-props bootstrap.servers=localhost:9092 acks=all
  ```

---

### 📖 3. Guía de Lectura e Interpretación de Resultados

Al finalizar el benchmark, la terminal imprimirá una línea de resumen como esta:

```text
100000 records sent, 161550.888530 records/sec (15.41 MB/sec), 8.89 ms avg latency, 312.00 ms max latency, 8 ms 50th, 17 ms 95th, 20 ms 99th, 21 ms 99.9th.
```

#### ¿Qué significa cada métrica?

* 🚀 **`records/sec` (Rendimiento / Throughput):** Cantidad de mensajes procesados por segundo. A mayor número, más capacidad de procesamiento.
* ⏱️ **`avg latency` (Latencia Promedio):** Tiempo medio (en milisegundos) que le tomó al cliente enviar el mensaje y recibir la respuesta del servidor.
* 🎯 **`50th` ($p_{50}$ - Mediana de latencia):** El **50% de tus mensajes** tardó menos de este valor en procesarse. Es la métrica más realista del comportamiento cotidiano.
* ⚠️ **`99th` ($p_{99}$ - Percentil 99):** El **99% de tus mensajes** tardó menos de este valor. Muestra cómo se comporta el sistema bajo picos de tensión o lentitud de red.
* 🔴 **`max latency` (Latencia Máxima):** El caso más lento registrado en toda la prueba (por ejemplo, durante la creación inicial del buffer).

---

### 🔍 ¿Qué conclusiones debemos extraer de los resultados?

Al comparar los reportes de ambas pruebas, notarás la regla de oro de la arquitectura de eventos:

| Métrica clave | `acks=0` | `acks=all` | ¿Qué nos enseña esto? |
| :--- | :--- | :--- | :--- |
| **Latencia Promedio** | **~8.8 ms** (Baja) | **~18.9 ms** (Alta) | Con `acks=0` la respuesta es **más del doble de rápida** porque el cliente no espera a que el broker guarde o confirme nada. |
| **Percentil 50 ($p_{50}$)** | **~8 ms** | **~19 ms** | La mitad de las peticiones en `acks=0` se liberaron en tiempo récord. |
| **Durabilidad** | ❌ **Riesgo** | ✅ **Garantizada** | En `acks=0` ganas velocidad, pero arriesgas datos si hay un fallo de red o caída del servidor. |

> 💡 **Nota para el desarrollador:**  
> A veces verás que `acks=all` procesa muchos `records/sec` en Redpanda. Esto ocurre por el **Batching (Agrupamiento)**: como el productor debe esperar la confirmación del broker, aprovecha ese tiempo de espera para agrupar más mensajes en un solo paquete (*batch*). Sin embargo, nota cómo la **latencia por mensaje individual** en `acks=all` siempre será mayor que en `acks=0`.

---

### 🌐 4. Verificación Visual en Conduktor

1. Abre **Conduktor Console** en tu navegador (`http://localhost:8080`) y dirígete al topic `ordenes-blackfriday`.
2. Activa los gráficos en la esquina superior derecha (**Graphs: ON**).
3. Vuelve a lanzar cualquiera de las pruebas masivas y observa en el panel **Produce rate (msg/s)** los picos de entrada en tiempo real.

---

## 🎯 Misión 4: Semánticas de Entrega (Simular Duplicados en Crashes)

**Objetivo:** Comprender la semántica por defecto en sistemas de eventos (**At-least-once** / Al menos una vez) y observar cómo la falta de confirmación (*commit*) provoca la re-entrega de mensajes duplicados tras un fallo (*crash*).

---

### 💡 ¿Qué es el *Auto-Commit* y qué pasa cuando falla una aplicación?

Cuando un consumidor procesa un mensaje, debe notificar al clúster que ya terminó mediante un **Offset Commit**.
* **Auto-commit activado (`true`):** El cliente confirma automáticamente su avance cada cierto tiempo.
* **Sin auto-commit (`false`):** Si la aplicación lee eventos y sufre un *crash* antes de hacer el *commit*, el clúster asumirá que **nadie los procesó exitosamente** y se los volverá a entregar al siguiente consumidor que se conecte.

> ⚠️ **Nota técnica:** Usaremos el cliente oficial de Apache Kafka (`kafka-console-consumer.sh`) corriendo en Docker para este laboratorio, ya que `rpk` fuerza el *auto-commit* automático al usar grupos de consumo y no permite simular el fallo de forma limpia.

---

### 🧪 Paso a Paso para Simular la Re-entrega por Crash

#### 0. Limpiar el Topic (Paso Previo)
Para evitar leer los miles de eventos acumulados durante las pruebas de rendimiento (Misión 3), recrea el topic para iniciar con un estado limpio:

```cmd
docker exec -it redpanda-0 rpk topic delete ordenes-blackfriday
docker exec -it redpanda-0 rpk topic create ordenes-blackfriday -p 3
```

#### 1. Iniciar el Consumidor (Terminal A)
Abre una terminal CMD y ejecuta el consumidor oficial desactivando la confirmación automática (`enable.auto.commit=false`):

```cmd
docker run --rm -it --network container:redpanda-0 apache/kafka:latest /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ordenes-blackfriday --group grupo-auditoria --consumer-property enable.auto.commit=false --from-beginning
```

#### 2. Enviar eventos desde el Productor (Terminal B)
Abre una **segunda terminal CMD** y transmite un conjunto de 30 eventos al topic variando las claves (`cliente-%i`):

```cmd
(for /L %i in (1,1,30) do @echo cliente-%i:{"id": %i, "monto": 50}) | docker exec -i redpanda-0 rpk topic produce ordenes-blackfriday --format "%k:%v\n"
```

#### 3. Observar la pantalla y simular el Crash
Vuelve a la **Terminal A**. Verás cómo se imprimen los 30 mensajes en tiempo real. Una vez que terminen de llegar, presiona **`Ctrl + C`** para cerrar la consola abruptamente. 

*(Esto simula que tu microservicio sufrió una caída o fallo eléctrico ANTES de haber guardado su avance).*

#### 4. Reconectar el Consumidor tras el Crash
En la **Terminal A**, vuelve a lanzar **exactamente el mismo comando** del paso 1:

```cmd
docker run --rm -it --network container:redpanda-0 apache/kafka:latest /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ordenes-blackfriday --group grupo-auditoria --consumer-property enable.auto.commit=false --from-beginning
```

---

### ✅ Criterio de Aceptación y Análisis

* **Re-entrega de eventos (At-least-once):** El consumidor **volverá a leer los 30 mensajes desde el inicio**, demostrando que Redpanda re-entrega los eventos para evitar la pérdida de datos cuando no existe un *commit* confirmado en el broker.
* **Ordenamiento por partición:** Notarás que en la re-entrega los mensajes no necesariamente aparecen en orden strictly secuencial (`1, 2, 3...`), sino agrupados por lotes (por ejemplo: `2, 5, 8...`). Esto ocurre porque los mensajes se reparten entre las **3 particiones** del topic según su clave (`cliente-id`), recordando una regla fundamental: **el orden solo está garantizado dentro de la misma partición, no a nivel global del topic**.

---

## 💡 Resumen de Validación Práctica

| Concepto | Comprobación en la Práctica |
| :--- | :--- |
| **Consumer Group** | Varias terminales compartiendo el mismo `group.id` repartiéndose las 3 particiones. |
| **Rebalanceo** | Reasignación automática de particiones al abrir o cerrar terminales consumidoras. |
| **Consumidor Idle** | El 4to consumidor queda en reserva porque `Particiones (3) < Consumidores (4)`. |
| **Acks (`0` vs `-1`)** | Diferencia entre enviar "a ciegas" o esperar confirmación de persistencia. |
| **At-least-once** | Relectura de eventos duplicados tras fallar un consumidor sin hacer *commit* de *Offset*. |

---

## 🧹 Limpieza

Para detener los consumidores, eliminar el topic de prueba y apagar el entorno desde CMD:

```cmd
docker exec -it redpanda-0 rpk topic delete ordenes-blackfriday
docker compose down -v
```

---

## 📚 Anexos: Fundamentos Teóricos Relacionados

Para profundizar en la arquitectura, los mecanismos internos y las métricas evaluadas durante esta práctica:

* 👥 **Consumer Groups:** [consumer-groups.md](../../teoria/01-fundamentos/consumer-groups.md)
* 🔄 **Rebalancing:** [rebalancing.md](../../teoria/02-mecanismos-internos/rebalancing.md)
* 🔢 **Offsets:** [offsets.md](../../teoria/01-fundamentos/offsets.md)
* 🤝 **Producer ACKs:** [producer-acks.md](../../teoria/01-fundamentos/producer-acks.md)
* 📦 **Batching and Compression:** [batching-and-compression.md](../../teoria/02-mecanismos-internos/batching-and-compression.md)
* 📊 **Benchmarking and Metrics:** [benchmarking-and-metrics.md](../../teoria/03-ecosistema-avanzado/benchmarking-and-metrics.md)
* 🚚 **Delivery Semantics:** [delivery-semantics.md](../../teoria/01-fundamentos/delivery-semantics.md)