# 📌 Consumer

> Un Consumer es la aplicación o cliente que se suscribe a uno o más topics para leer, deserializar y procesar los eventos almacenados en las particiones del clúster.

---

## 💡 La Analogía del Mundo Real

Piensa en un **Consumer** como un **lector en una biblioteca pública**.

El lector entra a la biblioteca, busca la sección que le interesa (Topic), toma un libro y utiliza su propio separador de páginas (Offset) para saber exactamente dónde se quedó. El lector lee a su propio ritmo sin romper las hojas ni alterar el texto. Si cierra el libro para ir a almorzar y regresa más tarde, reanuda la lectura a partir del separador de páginas. Además, múltiples lectores pueden leer copias del mismo libro al mismo tiempo sin molestarse entre sí.

---

## 🧩 Anatomía y Funcionamiento

El Consumer extrae los datos del clúster mediante un esquema de consumo controlado por la propia aplicación.

```text
[ BROKER ] ◄─── (Petición Poll) ─── [ CONSUMER ] ───► [ Deserializador ] ──► [ Lógica de Negocio ]
  (Disco)  ───► (Arreglo byte[]) ───► (Buffer)   ───► (Byte[] -> Object) ──► (Procesamiento)
```

### 1. Modelo de Extracción Basado en Pull
A diferencia de los sistemas tradicionales de mensajería que empujan datos (*Push*), en Kafka el Consumer solicita activamente los mensajes (*Pull*) mediante ciclos continuos (`poll()`). Esto garantiza que la aplicación consumidora controle su propio ritmo y nunca sea saturada por un volumen de tráfico superior a su capacidad de procesamiento.

### 2. Deserialización de Payload
Dado que los datos viajan por el clúster como arreglos de bytes neutros (`byte[]`), el Consumer debe utilizar el deserializador adecuado (JSON, Avro, Protobuf, String) para transformar esos bytes de nuevo en objetos utilizables dentro de la lógica de código.

### 3. Avance y Marcado de Posición (Offset Commit)
El Consumer es responsable de reportar periódicamente al clúster cuál fue el último Offset que procesó con éxito. Esta confirmación (*commit*) permite que, ante una falla o reinicio de la aplicación, el Consumer retome el procesamiento exactamente en el evento siguiente sin perder datos.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"El Broker empuja (push) los mensajes al consumidor tan pronto como llegan"* | Los consumidores utilizan un **modelo Pull**, solicitando eventos activamente al Broker a través del método `poll()` para evitar saturarse. |
| *"Cuando un consumidor lee un evento, este se destruye para evitar lecturas duplicadas"* | La lectura **no altera ni elimina los datos**. Múltiples consumidores independientes pueden leer los mismos mensajes tantas veces como lo necesiten. |
| *"Un consumidor puede editar directamente el contenido de un evento almacenado en el Broker"* | Los logs de Kafka son **estrictamente inmutables**. Si un consumidor detecta un dato erróneo, debe procesarlo y, si aplica, emitir un nuevo evento corregido a otro Topic. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué ventaja técnica ofrece el modelo "Pull" para los consumidores en comparación con un modelo "Push"?</b></summary>

<p><b>Respuesta:</b> Evita que el consumidor se sature durante picos altos de tráfico, ya que es la propia aplicación consumidora la que decide cuándo y cuántos eventos solicitar según su capacidad de procesamiento disponible.</p>

</details>

<details>
<summary><b>2. ¿Qué ocurre si un Consumer se reinicia y no realizó el commit de sus últimos Offsets procesados?</b></summary>

<p><b>Respuesta:</b> Al volver a conectarse, leerá nuevamente los eventos desde el último Offset confirmado guardado en el clúster, provocando un procesamiento duplicado de esos mensajes en la aplicación.</p>

</details>

<details>
<summary><b>3. ¿Por qué el Consumer debe especificar un Deserializador de clave y valor al conectarse?</b></summary>

<p><b>Respuesta:</b> Porque el Broker le entregará los datos como una secuencia pura de bytes (<code>byte[]</code>) y el Consumer necesita la instrucción exacta para convertir esa secuencia en objetos de lenguaje de programación válidos.</p>

</details>