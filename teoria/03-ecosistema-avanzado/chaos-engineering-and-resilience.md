# 📌 Ingeniería del Caos y Resiliencia (Chaos Engineering and Resilience)

> Es la disciplina de experimentar sobre un sistema distribuido inyectando fallos intencionados para verificar la capacidad del clúster y sus aplicaciones clientes de tolerar turbulencias sin interrupción de servicio ni pérdida de datos.

---

## 💡 La Analogía del Mundo Real

Imagina el sistema de extinción de incendios y alarmas de un rascacielos. Esperar a que ocurra un incendio real para descubrir si la presión de las mangueras es suficiente o si las salidas de emergencia están bloqueadas es un riesgo inaceptable. En su lugar, los inspectores realizan simulacros programados, cortan la electricidad intencionalmente y activan detectores de humo bajo condiciones controladas. Esto permite identificar fallas en las tuberías o en la señalización antes de una emergencia real, garantizando que el edificio sea genuinamente resiliente.

---

## 🧩 Anatomía y Funcionamiento

La ingeniería del caos en entornos de mensajería de eventos no evalúa el sistema en estado ideal, sino su comportamiento bajo condiciones de falla realistas (caída de brokers, latencia de red, saturación de disco o pausas de recolección de basura).

```text
[ Estado Estable (Steady State) ] ---> Inyección de Caos (ej. kill -9 a Broker Líder)
             |                                        |
             v                                        v
[ Métrica: Producción continua ]        [ Reacción: Reelección de Líder + ISR ]
             |                                        |
             +<--- (Verificación de Resiliencia) -----+
             v
[ Resultado: Cero pérdida de datos (Zero Data Loss) y recuperación automática ]
```

### 1. Definición del Estado Estable (*Steady State*)
Antes de inyectar cualquier fallo, se establece una línea base mediante métricas clave de rendimiento y disponibilidad (tasa de mensajes procesados por segundo, latencia de consumo, errores de cliente y tiempo de respuesta). El experimento busca demostrar que este estado estable se mantiene o se recupera de forma autónoma.

### 2. Inyección Controlada de Fallos (*Fault Injection*)
Se introducen perturbaciones simuladas en la infraestructura, tales como la detención abrupta de nodos (*broker crash*), particionamiento de red mediante reglas de cortafuegos, degradación en la velocidad del almacenamiento en disco o pausas largas del recolector de basura (*Garbage Collection*).

### 3. Validación de Resiliencia en Clientes y Clúster
Un clúster resiliente debe reajustar sus réplicas en sincronía (ISR) y elegir un nuevo Líder de forma transparente. Simultáneamente, los productores y consumidores deben absorber el evento mediante políticas de reintento (*retries*), backoff exponencial e idempotencia, sin lanzar excepciones fatales hacia la capa de negocio.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"La Ingeniería del Caos consiste en romper servidores al azar en producción sin control"* | Es un **método científico empírico y controlado**. Requiere definir hipótesis previas, métricas de estado estable y un radio de impacto (*blast radius*) estricto para mitigar riesgos. |
| *"Tener un clúster con Replication Factor = 3 garantiza la resiliencia total del sistema"* | La infraestructura puede auto-repararse, pero **si los productores/consumidores no tienen configurados reintentos y tiempos de espera adecuados**, la aplicación colapsará ante el cambio de Líder. |
| *"Los experimentos de caos solo deben ejecutarse en entornos de desarrollo local"* | Los entornos locales no reproducen la latencia real, los patrones de concurrencia ni el tráfico distribuido. Las pruebas deben avanzar progresivamente **hasta validar entornos productivos**. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Cuál es la función principal de definir el "estado estable" (Steady State) antes de inyectar caos en un clúster?</b></summary>

<p><b>Respuesta:</b> Establecer la línea base de métricas normales (rendimiento, latencia, tasa de errores) para comprobar si el sistema conserva o recupera su comportamiento esperado durante y después del fallo.</p>

</details>

<details>
<summary><b>2. Si se fuerza la caída del broker Líder de una partición, ¿qué configuración del lado del Productor evita que el evento se pierda o falle definitivamente?</b></summary>

<p><b>Respuesta:</b> La combinación de <code>acks=all</code>, la habilitación de reintentos automáticos (<code>retries</code>) con backoff exponencial y el parámetro de idempotencia (<code>enable.idempotence=true</code>).</p>

</details>

<details>
<summary><b>3. ¿A qué se refiere el término "radio de impacto" (Blast Radius) en un experimento de resiliencia?</b></summary>

<p><b>Respuesta:</b> Al límite de la extensión del experimento (un porcentaje de tráfico, una partición o un nodo específico) diseñado para minimizar las afectaciones a usuarios reales en caso de que la hipótesis de resiliencia falle.</p>

</details>