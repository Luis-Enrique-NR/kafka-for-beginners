# 📌 Líder y Seguidores (Leader and Followers)

> Es el modelo de roles jerárquicos asignados a las réplicas de una partición, donde un único nodo actúa como punto central de procesamiento de operaciones mientras los demás sincronizan sus datos en segundo plano para garantizar redundancia.

---

## 💡 La Analogía del Mundo Real

Imagina la dinámica de un Director Gerente y sus Asistentes en una firma de contratos. Cada cliente que llega a la oficina debe interactuar exclusivamente con el Director Gerente (Líder), quien revisa, firma e ingresa los contratos en el registro principal. Los Asistentes (Seguidores) no atienden a los clientes directamente; su único trabajo es copiar inmediatamente cada contrato firmado en sus propios libros de archivo. Si el Director Gerente se enferma repentinamente, uno de los Asistentes asume el rol de Director de inmediato sin interrumpir la atención al cliente ni perder ningún contrato previamente archivado.

---

## 🧩 Anatomía y Funcionamiento

En Apache Kafka, la responsabilidad de gestionar los datos de una partición no se divide en partes iguales entre sus réplicas, sino que se asignan roles claros para mantener la consistencia del registro (*Log*).

```text
  Productor / Consumidor
           |
    (Escritura/Lectura)
           v
  +-------------------+
  |    Broker 1       |  <-- Réplica Líder (Atiende I/O)
  |  Partición 0 (L)  |
  +-------------------+
    |               |
 (Fetch)         (Fetch)
    v               v
+------------+  +------------+
|  Broker 2  |  |  Broker 3  |  <-- Réplicas Followers (Pasivas)
| P-0 (F)    |  | P-0 (F)    |
+------------+  +------------+
```

### 1. El Rol de la Réplica Líder (Leader)
Para cada partición existe una única réplica designada como Líder. Es la responsable exclusiva de recibir las escrituras de los productores y asignarle a cada mensaje su número secuencial (*Offset*). Por defecto, también procesa todas las peticiones de lectura de los consumidores, garantizando una vista completamente consistente de los eventos.

### 2. El Rol de las Réplicas Seguidoras (Followers)
Las réplicas Followers no atienden peticiones de clientes de forma directa (salvo configuraciones avanzadas de lectura local). Funcionan como consumidores internos que envían peticiones continuas de extracción (*Fetch Requests*) a la réplica Líder para replicar de forma exacta la secuencia de mensajes en sus propios discos. Su objetivo principal es mantenerse actualizadas para asumir el liderazgo si la réplica Líder falla.

### 3. Balanceo de Líderes en el Clúster
Kafka distribuye los roles de Líder de las distintas particiones entre todos los brokers disponibles en el clúster. Un broker puede ser la réplica Líder para la Partición 0 de un Topic y, al mismo tiempo, actuar como réplica Follower para la Partición 1 del mismo Topic. Esta distribución evita cuellos de botella y maximiza el uso del hardware.

---

## ⚠️ Errores Comunes y Mitos

| Lo que la gente cree (Incorrecto) | Lo que realmente ocurre (Correcto) |
| :--- | :--- |
| *"Los Followers procesan las escrituras en paralelo con el Líder para repartir el trabajo"* | **Todas las escrituras son procesadas únicamente por el Líder.** Los Followers son réplicas pasivas que copian los datos en segundo plano mediante peticiones de extracción (*Fetch*). |
| *"Un broker es Líder o Follower de todo un Topic completo"* | El rol de Líder se asigna **de forma individual por cada partición**. Un broker puede ostentar el rol de Líder en unas particiones y de Follower en otras dentro del mismo Topic. |
| *"Las réplicas Followers nunca pueden procesar lecturas de consumidores"* | Aunque por defecto el Líder atiende todas las lecturas, en versiones modernas de Kafka es posible habilitar la lectura desde la réplica más cercana (*Fetch from Closest Replica*) para optimizar costos de red en la nube. |

---

## 🧠 Autoevaluación Rápida

<details>
<summary><b>1. ¿Cuál es la función principal de una réplica Follower bajo condiciones normales de operación?</b></summary>

<p><b>Respuesta:</b> Sincronizar continuamente los eventos desde la réplica Líder mediante peticiones <i>Fetch</i> para mantener una copia idéntica del registro y estar preparada para asumir el liderazgo en caso de falla.</p>

</details>

<details>
<summary><b>2. ¿Qué ocurre con el tráfico de clientes si el broker que aloja la réplica Líder de una partición se apaga repentinamente?</b></summary>

<p><b>Respuesta:</b> El clúster reelecto de forma automática a una de las réplicas Followers en sincronía como nuevo Líder, y los clientes redirigen su tráfico hacia esa nueva réplica sin pérdida de servicio.</p>

</details>

<details>
<summary><b>3. ¿Qué problema de arquitectura surge cuando un solo broker concentra la mayoría de las réplicas Líder del clúster?</b></summary>

<p><b>Respuesta:</b> Se produce un desbalanceo de carga (<i>Leader Skew</i>), donde ese broker absorbe casi la totalidad del tráfico de entrada/salida y procesamiento, mientras los demás brokers permanecen subutilizados actuando solo como Followers.</p>

</details>