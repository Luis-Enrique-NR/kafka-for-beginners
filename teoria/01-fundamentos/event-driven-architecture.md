# 📌 Event-Driven Architecture

> Event-Driven Architecture (EDA) es un patrón de diseño de software en el que la producción, detección y consumo de eventos constituyen el mecanismo principal para acoplar sistemas de forma débil y reaccionar a cambios de estado en tiempo real.

---

## 💡 La Analogía del Mundo Real

Piensa en **Event-Driven Architecture** como el **sistema de comandas de un restaurante concurrido**.

El mesero anota un pedido y pega el ticket en la barra de la cocina (emite un evento). El mesero no se queda parado frente al cocinero esperando a que prepare el platillo; inmediatamente se retira a atender otras mesas. Por su parte, el cocinero toma el ticket cuando tiene espacio libre en la parrilla, el barman toma la sección de bebidas y el sistema de facturación registra la venta. Cada miembro trabaja a su propio ritmo reaccionando al evento generado por el ticket.

---

## 🧩 Anatomía y Funcionamiento

EDA transforma la comunicación entre servicios cambiando peticiones directas por la publicación de hechos sucedidos en el sistema.

```text
                                            ┌──► [ Servicio de Inventario ]
[ Servicio de Órdenes ] ──► ( Publica Evento )  │
     (Productor)           "OrdenCreada"    ├──► [ Servicio de Envíos ]
                                │           │
                                ▼           └──► [ Servicio de Notificaciones ]
                         [ Event Broker ]                  (Consumidores)
```

### 1. Comunicación Basada en Hechos Inmutables
En EDA, un evento representa un hecho del pasado que ya ocurrió (por ejemplo, `OrdenCreada`, `PagoRechazado` o `TemperaturaElevada`). Al ser eventos pasados, son inmutables y sirven como la fuente única de verdad sobre lo que sucedió en el negocio.

### 2. Desacoplamiento Total (Publish / Subscribe)
El emisor (Productor) no sabe quién leerá su evento, cuántos servicios lo consumirían ni en qué servidor están alojados. Solo emite el hecho al intermediario (*Event Broker*). Esto permite añadir nuevos consumidores en el futuro sin modificar ni una sola línea de código del emisor.

### 3. Procesamiento Asíncrono e Independiente
A diferencia del modelo tradicional *Request/Response* (donde el cliente se bloquea esperando una respuesta inmediata), en EDA los componentes trabajan de forma asíncrona. Si un consumidor se apaga o se retrasa, el emisor continúa funcionando sin interrupción.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"EDA es simplemente hacer llamadas API REST asíncronas entre microservicios"* | REST sigue siendo una petición dirigida punto a punto; EDA se basa en **publicar hechos al aire** sin destino ni receptor explícito. |
| *"El servicio emisor indica qué acción debe ejecutar el servicio receptor"* | El emisor solo **informa que algo ocurrió**; es el receptor quien decide de forma autónoma qué acción tomar según su propia lógica. |
| *"Si adopto EDA ya no necesito bases de datos relacionales ni persistencia"* | EDA gestiona el **flujo de comunicación y eventos en tránsito**; las bases de datos siguen siendo necesarias para almacenar estados persistentes a largo plazo. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Cuál es la diferencia fundamental entre una arquitectura basada en Request/Response (como REST) y una basada en Eventos (EDA)?</b></summary>

<p><b>Respuesta:</b> En Request/Response el emisor le pide explícitamente a un servicio destino que haga algo y espera su respuesta; en EDA el emisor solo anuncia un hecho que ya ocurrió sin importar quién o cuándo lo procesen.</p>

</details>

<details>
<summary><b>2. ¿Cómo beneficia el desacoplamiento de EDA al momento de agregar un nuevo requerimiento de negocio?</b></summary>

<p><b>Respuesta:</b> Permite conectar un nuevo servicio consumidor para escuchar eventos existentes sin necesidad de modificar, redeplegar ni interrumpir a los servicios emisores originales.</p>

</details>

<details>
<summary><b>3. ¿Por qué se nombran los eventos en tiempo pasado (ej. `OrdenProcesada` en lugar de `ProcesarOrden`)?</b></summary>

<p><b>Respuesta:</b> Porque representan hechos históricos inmutables que ya sucedieron en el sistema, en lugar de comandos u órdenes dirigidas a un componente en particular.</p>

</details>