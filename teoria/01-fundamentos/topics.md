# 📌 Topic

> Un Topic es una categoría o canal lógico nombrado donde los productores publican eventos y desde el cual los consumidores leen datos.

---

## 💡 La Analogía del Mundo Real

Piensa en un **Topic** como una **estación de radio** o una **sección de un periódico** (por ejemplo, la sección de *Deportes*).

Si te sintonizas a la estación de *Deportes*, esperas recibir únicamente noticias deportivas. La estación no crea las noticias ni decide quién las escucha; solo sirve como el canal organizado. Múltiples locutores (productores) pueden transmitir noticias a esa sección y millones de oyentes (consumidores) pueden escuchar la misma transmisión al mismo tiempo sin interferir entre sí.

---

## 🧩 Anatomía y Funcionamiento

Un Topic organiza el flujo de eventos en categorías claras dentro del clúster de Kafka/Redpanda.

```text
[ Producer A ] ──┐
                 ├───► [ TOPIC: ordenes-compra ] ───► [ Consumer Group 1 ]
[ Producer B ] ──┘     ├── Partición 0          ───► [ Consumer Group 2 ]
                       └── Partición 1
```

### 1. Categorización Lógica de Eventos
Un Topic es una entidad lógica con un nombre único (por ejemplo, `ordenes-compra`, `pagos-procesados` o `metricas-servidor`). Sirve para separar distintos dominios de datos y evitar que eventos disímiles se mezclen.

### 2. Abstracción sobre Particiones Físicas
Aunque lógicamente vemos el Topic como un solo canal continuo, físicamente se divide en una o más **particiones** repartidas a lo largo de los discos del clúster. Esto permite dividir el trabajo y escalar el procesamiento.

### 3. Modelo Publish/Subscribe Multiconsumidor
Los datos publicados en un Topic no se eliminan cuando un consumidor los lee. Múltiples aplicaciones independientes pueden suscribirse al mismo Topic y procesar la misma información a su propio ritmo sin afectar a los demás.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"El Topic borra los mensajes tan pronto como un consumidor los lee"* | Los mensajes permanecen en el Topic según la **política de retención** (por tiempo o tamaño), independientemente de cuántos consumidores los hayan leído. |
| *"Debo crear un Topic para cada mensaje o transacción individual"* | Los Topics son **categorías permanentes** de eventos del mismo tipo. Crear Topics efímeros sobrecarga innecesariamente los metadatos del clúster. |
| *"Un Topic solo permite un consumidor a la vez"* | Un Topic soporta **múltiples consumidores y grupos independientes** leyendo exactamente los mismos datos en paralelo. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Qué sucede con los eventos de un Topic después de que un consumidor los procesa con éxito?</b></summary>

<p><b>Respuesta:</b> Permanecen almacenados en el Topic hasta que se cumpla la política de retención configurada (por ejemplo, 7 días o límite de espacio), permitiendo que otros consumidores los lean si es necesario.</p>

</details>

<details>
<summary><b>2. ¿Cuál es la diferencia entre la visión lógica y la visión física de un Topic?</b></summary>

<p><b>Respuesta:</b> Lógicamente es un canal nombrado único (ej. `pagos`); físicamente es un conjunto de una o más particiones distribuidas en los discos de los Brokers del clúster.</p>

</details>

<details>
<summary><b>3. Si dos aplicaciones distintas necesitan leer exactamente los mismos eventos de un Topic, ¿deben duplicarse los mensajes?</b></summary>

<p><b>Respuesta:</b> No. Ambas aplicaciones pueden suscribirse al mismo Topic simultáneamente como consumidores independientes y leer los mismos datos sin interferir entre sí ni duplicar almacenamiento.</p>

</details>