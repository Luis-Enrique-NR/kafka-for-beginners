# 🚀 Misión 01: El Primer Flujo de Eventos en TechStore

## 📌 Nota Técnica Inicial: Entorno Community y Enfoque Híbrido

En versiones recientes de **Conduktor Console (Community Edition)**, la plataforma funciona como una herramienta de **observabilidad y monitoreo**, pero restringe la creación directa de recursos desde la interfaz web.

Por esta razón, utilizaremos un **enfoque híbrido**:
* **CLI (Terminal):** Para la creación y administración de recursos (Topics, configuraciones).
* **Conduktor Console (GUI):** Para la inspección visual de datos, auditoría de mensajes y monitoreo del clúster.

---

## 📜 Escenario de Negocio

Como Ingeniero de Datos en **TechStore**, debes habilitar el canal de comunicación en streaming para procesar **órdenes de compra**. El sistema debe soportar procesamiento en paralelo mediante múltiples particiones, garantizando que todos los eventos asociados a un mismo cliente se procesen de forma estrictamente secuencial.

---

## 🎯 Misión 1: Verificación de Infraestructura (Broker)

**Objetivo:** Inicializar los servicios con Docker Compose y confirmar que el Broker está activo antes de enviar tráfico.

* **Acción:**
  1. Abre tu terminal en la raíz del proyecto e ingresa a la carpeta de esta práctica:
     ```bash
     cd laboratorios/01-introduccion-conduktor
     ```
  2. Levanta los contenedores en segundo plano:
     ```bash
     docker compose up -d
     ```
  3. Abre Conduktor Console en tu navegador: `http://localhost:8080`.
  4. En el menú lateral izquierdo, selecciona **Brokers**.

* **✅ Criterio de Aceptación:**
  * Visualizar **1 Broker** listado con ID `0`.
  * Dirección asignada: `redpanda-0:9092` (Rol: **Controller**).
  * Estado global del clúster: `Healthy`.

---

## 🎯 Misión 2: Crear el Topic (CLI + GUI)

**Objetivo:** Aprovisionar el canal `ordenes-compra` con **2 particiones** utilizando la herramienta de administración de Redpanda (`rpk`).

* **Acción (Terminal):**
  Ejecuta el siguiente comando para crear el topic:

  ```bash
  docker exec -it redpanda-0 rpk topic create ordenes-compra -p 2
  ```

* **Verificación (Conduktor GUI):**
  1. Ve a la sección **Topics** en el menú izquierdo.
  2. Actualiza la vista (`F5`).

* **✅ Criterio de Aceptación:**
  * El topic `ordenes-compra` aparece listado con `Partitions: 2`.

---

## 🎯 Misión 3: Producir Eventos (Producer & Keys)

**Objetivo:** Simular el envío de eventos para validar el enrutamiento por clave (*Key*).

* **Acción:**
  1. En Conduktor, ingresa al topic `ordenes-compra` y selecciona la pestaña **Produce**.
  2. Envía los siguientes 4 eventos (en la caja **Key** asegúrate de no incluir espacios al inicio ni al final):

  * **Evento 1 (Sin Key):**
    * **Key:** *(vacío)*
    * **Value:** 
      ```json
      {"id": 101, "cliente": "Cliente_A", "producto": "Mouse", "monto": 25.00}
      ```

  * **Evento 2 (Sin Key):**
    * **Key:** *(vacío)*
    * **Value:** 
      ```json
      {"id": 102, "cliente": "Cliente_B", "producto": "Teclado", "monto": 45.00}
      ```

  * **Evento 3 (Con Key):**
    * **Key:** `cliente-001`
    * **Value:** 
      ```json
      {"id": 103, "cliente": "Cliente_C", "producto": "Monitor", "monto": 300.00}
      ```

  * **Evento 4 (Con Key idéntica):**
    * **Key:** `cliente-001`
    * **Value:** 
      ```json
      {"id": 104, "cliente": "Cliente_C", "producto": "Audífonos", "monto": 80.00}
      ```

---

## 🎯 Misión 4: Auditoría de Eventos (Consume, Partitions & Offsets)

**Objetivo:** Inspeccionar la distribución física de los mensajes y la asignación de *Offsets*.

* **Acción:**
  1. Dentro del topic `ordenes-compra`, cambia a la pestaña **Consume**.
  2. En la esquina superior derecha de la tabla de datos, haz clic en el botón **Display**.
  3. Activa las casillas **Partition** y **Offset** para hacer visibles estas columnas.

* **✅ Criterio de Aceptación:**
  1. **Determinismo por Key:** Los eventos 103 y 104 (que comparten la clave `cliente-001`) deben estar ubicados en la **misma partición**.
  2. **Secuencia de Offsets:** En la partición donde cayeron los eventos con la clave `cliente-001`, sus números de *Offset* deben ser consecutivos e incrementales.
  3. **Balanceo sin Key:** Los eventos 101 y 102 deben haber sido asignados por el enrutador del clúster según la política de asignación activa.

---

## 💡 Resumen de Validación Práctica

| Concepto | Comprobación en UI |
| :--- | :--- |
| **Broker** | Nodo disponible `redpanda-0:9092` en la pestaña *Brokers*. |
| **Topic** | Recipiente lógico `ordenes-compra`. |
| **Partition** | Canales independientes (`0` y `1`) mostrados en la columna *Partition*. |
| **Message Key** | Identificador (`cliente-001`) que garantiza el envío a una misma partición. |
| **Offset** | Identificador numérico único por posición dentro de cada partición. |

---

## 🧹 Limpieza

Para detener la infraestructura y eliminar volúmenes temporales al finalizar la práctica:

```bash
docker compose down -v
```

---

## 📚 Anexos: Fundamentos Teóricos Relacionados

Para comprender en detalle la teoría detrás de los componentes utilizados en esta práctica, consulta los siguientes documentos:

* 🏢 **Broker:** [brokers.md](../../teoria/01-fundamentos/brokers.md)
* 📂 **Topic:** [topics.md](../../teoria/01-fundamentos/topics.md)
* 🧩 **Partición:** [partitions.md](../../teoria/01-fundamentos/partitions.md)
* 🔑 **Message Key:** [message-keys.md](../../teoria/01-fundamentos/message-keys.md)
* 💬 **Message Value:** [message-value.md](../../teoria/01-fundamentos/message-value.md)
* 🔢 **Offset:** [offsets.md](../../teoria/01-fundamentos/offsets.md)
* 📤 **Producer:** [producers.md](../../teoria/01-fundamentos/producers.md)
* 📥 **Consumer:** [consumers.md](../../teoria/01-fundamentos/consumers.md)
* ⚡ **Event-Driven Architecture:** [event-driven-architecture.md](../../teoria/01-fundamentos/event-driven-architecture.md)