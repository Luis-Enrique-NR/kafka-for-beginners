# 📌 Mínimo de Réplicas en Sincronía (min.insync.replicas)

> Es la configuración a nivel de topic o clúster que establece la cantidad mínima de réplicas en sincronía que deben confirmar una escritura cuando el productor exige garantías máximas, asegurando la durabilidad frente a pérdidas de nodos.

---

## 💡 La Analogía del Mundo Real

Imagina un proceso de transferencia bancaria corporativa de alto valor. Para aprobar la salida de dinero, la política interna exige la firma obligatoria de al menos 2 gerentes de un comité de 3. Si 2 gerentes sufren una emergencia y están ilocalizables, quedando solo 1 disponible, el banco rechaza inmediatamente cualquier nueva solicitud de transferencia. En lugar de procesar operaciones con una sola firma (lo cual pondría en riesgo los fondos si ese único gerente comete un error), el sistema se bloquea preventivamente para garantizar la integridad financiera hasta que reaparezca el segundo gerente.

---

## 🧩 Anatomía y Funcionamiento

La propiedad `min.insync.replicas` actúa como un freno de seguridad que valida el tamaño actual del grupo **ISR** antes de dar por completada una transacción solicitada por un productor.

```text
Configuración: RF = 3, min.insync.replicas = 2, Producer acks = all

CASO A: ISR = [Broker 1, Broker 2] (Tamaño = 2)
Productor ---> (Escribe evento) ---> Líder (B1) ---> Copia a Follower (B2)
                                                         |
                                 <--- OK (ACK 2/2) ------+ (ESCRITURA EXITOSA)

CASO B: ISR = [Broker 1] (Tamaño = 1, Broker 2 y 3 cayeron)
Productor ---> (Escribe evento) ---> Líder (B1)
                                         |
                                 <--- ERROR -------------+ (NotEnoughReplicasException)
```

### 1. El "Combo de Oro" de la Durabilidad
`min.insync.replicas` no opera de forma aislada; funciona en pareja con la configuración del productor `acks=all` (o `acks=-1`). Cuando un productor envía un evento con `acks=all`, Kafka le exige al Líder que espere la confirmación de **todas** las réplicas presentes actualmente en el grupo ISR. La regla `min.insync.replicas` le impone una cota inferior a ese grupo ISR para evitar que un ISR degradado (por ejemplo, reducido a solo 1 nodo) acepte escrituras de forma insegura.

### 2. Comportamiento ante Caídas de Nodos (`NotEnoughReplicasException`)
Si múltiples brokers caen y el número de réplicas activas dentro del ISR disminuye por debajo del umbral de `min.insync.replicas`, el Líder rechazará inmediatamente las nuevas peticiones de escritura lanzando una excepción de tipo `NotEnoughReplicasException` (o `NotEnoughReplicasAfterAppendException`).

### 3. Asimetría entre Escritura y Lectura
Es fundamental destacar que este parámetro **solo afecta la disponibilidad de escrituras**. Cuando el grupo ISR cae por debajo del mínimo configurado:
* **Escrituras:** Quedan bloqueadas y son rechazadas para mantener la consistencia estricta.
* **Lecturas:** Los consumidores pueden seguir leyendo sin interrupciones los datos ya existentes desde las réplicas que permanezcan con vida.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Configurar min.insync.replicas = 2 garantiza que el evento siempre se escriba en los Brokers 1 y 2"* | **No especifica qué nodos físicos deben responder**, sino cuántas réplicas activas del grupo ISR actual deben confirmar la recepción, independientemente de cuáles brokers sean. |
| *"En un Topic con Replication Factor = 3, lo ideal es configurar min.insync.replicas = 3 para máxima seguridad"* | Asignar `min.insync.replicas = RF` **destruye la tolerancia a fallos del clúster**. Si un solo broker cae, el ISR baja a 2 y automáticamente se bloquean todas las escrituras. El estándar recomendado para $RF = 3$ es `min.insync.replicas = 2`. |
| *"El parámetro min.insync.replicas bloquea escrituras aunque el productor use acks=1"* | `min.insync.replicas` **es ignorado** si el productor envía peticiones con `acks=1` o `acks=0`. Solo surte efecto en el broker cuando la petición exige confirmación completa (`acks=all`). |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué excepción devuelve Kafka al productor cuando intenta escribir con acks=all en una partición cuyo ISR es menor al parámetro min.insync.replicas?</b></summary>

<p><b>Respuesta:</b> Devuelve la excepción <code>NotEnoughReplicasException</code> (o <code>NotEnoughReplicasAfterAppendException</code>), informando que la partición no cuenta con el número de réplicas en sincronía necesario para garantizar la confirmación solicitada.</p>

</details>

<details>
<summary><b>2. Si un Topic tiene RF = 3 y min.insync.replicas = 2, ¿cuántos brokers pueden caer simultáneamente manteniendo aún la capacidad de escribir y leer?</b></summary>

<p><b>Respuesta:</b> Puede caer exactamente 1 broker. Con 2 brokers con vida el grupo ISR se mantendrá en 2, cumpliendo el umbral mínimo para aceptar escrituras con <code>acks=all</code>.</p>

</details>

<details>
<summary><b>3. ¿Por qué un consumidor puede seguir procesando eventos en una partición cuya capacidad de escritura ha sido bloqueada por min.insync.replicas?</b></summary>

<p><b>Respuesta:</b> Porque <code>min.insync.replicas</code> restringe únicamente el registro de nuevos eventos para evitar inconsistencias de durabilidad; la consulta de eventos ya confirmados en el registro histórico sigue estando disponible en los nodos supervivientes.</p>

</details>