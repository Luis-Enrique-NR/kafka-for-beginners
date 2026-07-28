# 📌 Producer ACKs

> Producer ACKs es el parámetro de configuración que define el nivel de confirmación que exige un productor al clúster antes de considerar un mensaje como enviado exitosamente, balanceando la durabilidad de los datos contra la latencia de red.

---

## 💡 La Analogía del Mundo Real

Imagina que debes enviar documentos importantes por correo postal:

* **`acks=0` (Carta ordinaria / "Dispara y olvida"):** Depositas la carta en un buzón público de la calle y te retiras inmediatamente. Es el método más rápido porque no haces filas ni esperas firmas, pero si la carta se pierde en el camino, jamás te enterarás.
* **`acks=1` (Correo certificado individual):** Acudes a la ventanilla de la oficina postal local. El empleado recibe tu paquete, lo guarda en el mostrador principal y te entrega un recibo. Te retiras seguro de que la oficina central lo tiene, pero si la oficina sufre un incendio antes de transferir el paquete a los camiones de reparto, tu envío podría perderse.
* **`acks=all` (Encomienda de alta seguridad con custodia compartida):** Entregas un documento confidencial en ventanilla y exiges esperar en la sala hasta que el empleado confirme que el documento no solo se guardó en la bóveda central, sino que sus sucursales de respaldo en otras ciudades ya recibieron y archivaron una fotocopia oficial. Toma más tiempo, pero garantiza durabilidad absoluta.

---

## 🧩 Anatomía y Funcionamiento

El nivel de *Acknowledgement* (ACK) determina en qué punto del flujo de replicación el broker líder le responde al cliente productor para liberar la petición.

```text
       acks=0 (Sin espera)
Producer ───────────► Broker Líder
  │ (Respuesta inmediata)
  ▼

       acks=1 (Confirmación del Líder)
Producer ───────────► Broker Líder ──[Escribe en disco]──► Confirma al Productor
                          │
                          └──(Replicación asíncrona)──► Réplicas (ISR)

       acks=all / acks=-1 (Líder + Réplicas en Sincronía)
Producer ───────────► Broker Líder ──[Escribe en disco]
                          │
                          ├──► Réplica 1 ──[OK]──┐
                          │                      ├──► Confirma al Productor
                          └──► Réplica 2 ──[OK]──┘
```

### 1. `acks=0` (Sin Confirmación)
El productor transmite el evento por la red y no espera ninguna respuesta por parte del broker. Ofrece la **máxima velocidad y menor latencia posible**, pero no tiene garantías de entrega: si la red falla, el broker está saturado o la partición no existe, el evento se perderá sin que el productor pueda detectarlo.

### 2. `acks=1` (Confirmación del Líder)
El productor espera a que únicamente la réplica líder de la partición escriba el registro en su log local. Una vez guardado en el líder, este envía una confirmación al productor. Proporciona un **equilibrio entre velocidad y seguridad**, aunque existe un riesgo residual de pérdida si el líder sufre un fallo eléctrico antes de que las réplicas hayan alcanzado a copiar el mensaje.

### 3. `acks=all` o `acks=-1` (Confirmación Total de Réplicas)
El productor espera a que el broker líder y todas sus réplicas alineadas o en sincronía (**In-Sync Replicas / ISR**) confirmen haber recibido y escrito el mensaje. Ofrece **máxima durabilidad y cero pérdida de datos**, a costa de introducir una mayor latencia en cada envío.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Usar `acks=all` garantiza que un mensaje nunca se enviará duplicado"* | `acks=all` garantiza **durabilidad y persistencia**, no deduplicación. Si el mensaje se guarda bien pero la respuesta de confirmación se pierde en la red, el productor reintentará el envío generando un duplicado a menos que esté activada la **idempotencia**. |
| *"Usar `acks=all` obliga a esperar a TODAS las réplicas físicas configuradas en el topic"* | Solo obliga a esperar a las réplicas que forman parte del conjunto activo **ISR (In-Sync Replicas)**, cuyo umbral mínimo lo determina la configuración `min.insync.replicas`. |
| *"Configurar `acks=0` es inútil porque no ofrece garantías de entrega"* | Es un patrón altamente eficiente para casos de uso de gran volumen donde perder un porcentaje mínimo de eventos no impacta el negocio (ej. telemetría IoT, métricas de rendimiento, registros de *logs* de navegación). |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué parámetro de configuración a nivel de broker debe combinarse con `acks=all` para definir cuántas réplicas deben confirmar la recepción como mínimo?</b></summary>

<p><b>Respuesta:</b> El parámetro <code>min.insync.replicas</code>. Este define la cantidad mínima de réplicas alineadas que deben confirmar la escritura exitosa para que el broker devuelva un acuse positivo al productor.</p>

</details>

<details>
<summary><b>2. Si una aplicación procesa transacciones bancarias o compras con tarjeta de crédito, ¿cuál es el nivel de `acks` obligatorio que se debe configurar?</b></summary>

<p><b>Respuesta:</b> Se debe utilizar <code>acks=all</code> (o <code>acks=-1</code>) combinado con idempotencia activada. En aplicaciones financieras, la durabilidad y la garantía de no pérdida de datos priorizan sobre la latencia de red.</p>

</details>

<details>
<summary><b>3. Si el broker líder recibe un mensaje enviado con `acks=1` y responde con éxito al productor, pero se cae un milisegundo después antes de ser replicado, ¿qué sucede con ese evento?</b></summary>

<p><b>Respuesta:</b> El evento se perderá. Al promover una réplica secundaria a nuevo líder, esta no contendrá el registro guardado por el líder anterior, provocando una inconsistencia conocida como pérdida de datos por conmutación no sincronizada.</p>

</details>