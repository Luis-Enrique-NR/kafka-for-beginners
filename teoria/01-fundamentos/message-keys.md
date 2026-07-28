# 📌 Message Key

> La Message Key es un identificador opcional adjunto a un evento que utiliza el productor para determinar, mediante un algoritmo determinista de hashing, en qué partición específica debe almacenarse el mensaje.

---

## 💡 La Analogía del Mundo Real

Piensa en una **Message Key** como el **código postal en el sobre de una carta**.

Cuando envías correspondencia a un centro de clasificación de correo, el centro lee el código postal para colocar todas las cartas dirigidas a la misma zona en el mismo camión de reparto. Si no escribes un código postal (clave nula), el centro simplemente entrega las cartas al primer camión que tenga espacio disponible para balancear el trabajo, sin importar si cartas dirigidas al mismo destinatario terminan en vehículos diferentes.

---

## 🧩 Anatomía y Funcionamiento

La Message Key permite controlar de manera precisa el destino físico de cada evento dentro del clúster.

```text
[ Evento: Key="cliente-123" ] ──► [ Algoritmo de Hashing ] ──► (Hash % Total Particiones) ──► [ Partición 1 ]
[ Evento: Key="cliente-123" ] ──► [ Algoritmo de Hashing ] ──► (Hash % Total Particiones) ──► [ Partición 1 ]
[ Evento: Key=null          ] ──► [ Sticky Partitioner    ] ──► (Distribución de carga) ──► [ Partición 0 ]
```

### 1. Enrutamiento Determinista por Hashing
Cuando un evento incluye una clave, el productor aplica una función de hashing (por ejemplo, *Murmur2*) sobre los bytes de esa clave y realiza una operación de módulo con el número total de particiones: `particion = hash(key) % N`. Este cálculo es determinista: la misma clave siempre producirá exactamente el mismo resultado mientras el número de particiones no cambie.

### 2. Garantía de Orden por Entidad
Dado que Kafka garantiza el orden únicamente dentro de cada partición individual, el uso de una clave basada en la entidad de negocio (por ejemplo, `id_cliente` o `id_cuenta`) asegura que todas las transacciones de esa entidad caigan en la misma partición y, por ende, se procesen en estricto orden cronológico.

### 3. Comportamiento sin Clave (`Key = null`)
Si un evento se produce sin clave, el enrutador de Kafka utiliza estrategias de balanceo (como el *Sticky Partitioner*) para agrupar y repartir los eventos equitativamente entre todas las particiones disponibles, priorizando el rendimiento del clúster sobre el orden.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Si no especifico una Message Key, Kafka rechazará el evento"* | La clave es **completamente opcional**. Si no se envía, los datos se distribuyen entre las particiones para optimizar el rendimiento. |
| *"Claves distintas siempre caen en particiones distintas"* | Falso. Debido a la operación de módulo en el cálculo del hash (colisiones), dos claves diferentes pueden terminar en la misma partición. Lo único garantizado es que la **misma clave cae siempre en la misma partición**. |
| *"Puedo agregar más particiones a un Topic sin afectar el enrutamiento de mis claves existentes"* | Si el número total de particiones `N` cambia, la fórmula `hash(key) % N` producirá resultados diferentes, reasignando la misma clave a una partición distinta para nuevos eventos. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Por qué la Message Key es indispensable si requerimos procesar los eventos de un usuario en orden cronológico estricto?</b></summary>

<p><b>Respuesta:</b> Porque asegura que todos los eventos asociados a ese usuario sean dirigidos a la misma partición, que es la única estructura en Kafka que garantiza un ordenamiento secuencial e inmutable.</p>

</details>

<details>
<summary><b>2. ¿Es posible que los eventos con la clave `"cliente-A"` y los eventos con la clave `"cliente-B"` terminen en la misma partición?</b></summary>

<p><b>Respuesta:</b> Sí. El algoritmo de hashing puede generar colisiones o asignar el mismo número de partición al aplicar el operador módulo sobre la cantidad total de particiones disponibles.</p>

</details>

<details>
<summary><b>3. ¿Qué consecuencia tiene incrementar el número de particiones en un Topic donde ya se producían eventos con Message Keys?</b></summary>

<p><b>Respuesta:</b> Se altera la operación matemática de enrutamiento (`hash(key) % N`), lo que provocará que eventos futuros con claves previamente usadas terminen en particiones diferentes a las de sus eventos pasados, rompiendo la continuidad del orden por clave.</p>

</details>