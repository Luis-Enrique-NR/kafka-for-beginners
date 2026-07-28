# 📌 Offset

> El Offset es un número entero secuencial, inmutable e incremental asignado automáticamente a cada evento dentro de una partición para identificar de forma única su posición exacta.

---

## 💡 La Analogía del Mundo Real

Piensa en el **Offset** como el **número de página de un libro** o el **marcapáginas** que usas al leer.

Cada página del libro tiene un número incremental único que jamás cambia (0, 1, 2, 3...). Si estás leyendo y te quedas en la página 45, no necesitas volver a empezar desde la portada al día siguiente; simplemente abres el libro en la página 46. El número de página (Offset) le indica al lector (Consumidor) exactamente en qué punto exacto del relato se encuentra.

---

## 🧩 Anatomía y Funcionamiento

El Offset actúa como la dirección primaria e inalterable de cada evento dentro de la estructura de la partición.

```text
[ PARTICIÓ N 0 ]
┌───────────┬───────────┬───────────┬───────────┐
│ Offset 0  │ Offset 1  │ Offset 2  │ Offset 3  │ ... (Próximo: Offset 4)
└───────────┴───────────┴───────────┴───────────┘
      ▲                               ▲
      │                               │
(Mensaje más antiguo)          (Mensaje más reciente)
```

### 1. Ámbito Local por Partición
Los Offsets son independientes por cada partición. Esto significa que la partición 0 tiene su propio Offset 0, y la partición 1 también tiene su propio Offset 0. Para identificar unívocamente un mensaje en todo el clúster, se requiere la combinación triple de: **Topic + Partición + Offset**.

### 2. Secuencia Inmutable y Monotónicamente Incremental
Cuando un nuevo evento se escribe en una partición, se le asigna el siguiente número entero disponible (`Offset N+1`). Una vez asignado, ese Offset nunca se modifica, no se reutiliza y tampoco se reenumera aunque los mensajes más antiguos sean eliminados por caducidad.

### 3. Marcador de Progreso del Consumidor (Committed Offset)
Los consumidores informan periódicamente al clúster cuál fue el último Offset que procesaron con éxito (operación conocida como *Offset Commit*). Si una aplicación consumidora se cae o se reinicia, consulta este registro en el clúster para reanudar la lectura exactamente a partir del siguiente evento no procesado.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"El Offset es un identificador global único a nivel de todo el Topic"* | El Offset es **exclusivo de cada partición**. La partición 0 y la partición 1 del mismo Topic tendrán ambas un mensaje registrado con Offset 0. |
| *"Si se elimina el mensaje con Offset 0 por caducidad, todos los demás mensajes se reenumeran"* | Los Offsets son **inmutables**. Borrar mensajes antiguos genera un hueco inicial, pero la numeración de los eventos existentes y futuros jamás se altera. |
| *"El Broker decide qué Offset le corresponde leer al consumidor en cada petición"* | El **Consumidor es quien controla su lectura** indicando al Broker el Offset exacto a partir del cual desea recibir eventos. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Es posible que existan dos mensajes distintos dentro del mismo Topic que tengan exactamente el mismo número de Offset?</b></summary>

<p><b>Respuesta:</b> Sí, siempre y cuando pertenezcan a particiones distintas dentro de ese mismo Topic, ya que el contador de Offset es independiente en cada partición.</p>

</details>

<details>
<summary><b>2. Si una política de retención elimina los primeros 100 mensajes de una partición, ¿cuál será el Offset del primer mensaje disponible?</b></summary>

<p><b>Respuesta:</b> El primer mensaje disponible será el Offset 100. La eliminación de datos no altera ni disminuye los números de Offset de los mensajes restantes.</p>

</details>

<details>
<summary><b>3. ¿Cómo utiliza un consumidor el Offset para garantizar la tolerancia a fallos ante una caída del sistema?</b></summary>

<p><b>Respuesta:</b> Registra (commit) en el clúster el último Offset procesado. Al reiniciar, consulta ese valor guardado y solicita al Broker únicamente los eventos a partir de `Offset + 1`, evitando reprocesar la historia completa.</p>

</details>