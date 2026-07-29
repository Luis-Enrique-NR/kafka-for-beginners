# 🚀 Apache Kafka for Beginners

> Repositorio didáctico enfocado en el aprendizaje práctico, conceptual y arquitectónico de Apache Kafka moderno (KRaft, Alta Disponibilidad y Resiliencia).

---

## 📌 Descripción General

Este proyecto es una guía paso a paso diseñada para dominar **Apache Kafka** desde sus cimientos hasta patrones avanzados de producción. Combina **fichas teóricas atómicas** con **laboratorios prácticos guiados en Docker**, permitiendo experimentar de primera mano con escenarios reales de fallas, replicación, particionamiento y consumo masivo de eventos.

A diferencia de los cursos tradicionales basados únicamente en ZooKeeper, este repositorio utiliza la **arquitectura KRaft nativa**, preparando al estudiante para los estándares actuales de la industria.

---

## 🎯 Objetivos de Aprendizaje

Al completar los módulos de este repositorio serás capaz de:

* **Comprender la Event-Driven Architecture (EDA):** Entender el rol de Kafka como log distribuido inmutable.
* **Dominar los Mecanismos Internos:** Configurar y auditar Particiones, Offsets, Réplicas, ISR (*In-Sync Replicas*) y el parámetro `min.insync.replicas`.
* **Garantizar la Durabilidad:** Implementar las semánticas de entrega (*At-least-once*, *At-most-once*, *Exactly-once*) y el "Combo de Oro" de producción (`acks=all`).
* **Operar Clústeres KRaft:** Administrar quórums de metadatos sin depender de Apache ZooKeeper.
* **Aplicar Chaos Engineering:** Inyectar fallos reales (`docker stop`), evaluar reelecciones de líder en tiempo real y diseñar sistemas verdaderamente tolerantes a fallos.

---

## 👤 ¿A quién va dirigido?

* **Desarrolladores de Software** que buscan integrar arquitectura de eventos en sus aplicaciones.
* **Ingenieros de Datos** que necesitan construir pipelines de streaming confiables y escalables.
* **Arquitectos de Software y DevOps / SREs** interesados en operar, escalar y asegurar clústeres de Kafka en entornos de alta disponibilidad.

---

## ⚙️ Prerrequisitos y Stack Tecnológico

Para ejecutar los laboratorios prácticos de este repositorio necesitarás:

* **Git:** Para clonar el repositorio.
* **Docker Desktop / Docker Engine:** (Versión 20.10 o superior) con **Docker Compose V2**.
* **Línea de Comandos (CLI):** Bash, CMD de Windows o PowerShell.
* **Memoria RAM:** Mínimo 8 GB (recomendado 16 GB para clústeres multi-broker).

---

## 🗺️ Estructura y Metodología del Repositorio

El proyecto utiliza un enfoque **doble desacoplado**: la teoría existe de forma independiente a la práctica para servir como fuente de consulta permanente.

```text
Kafka for Beginners/
├── teoria/                   # Fichas teóricas atómicas y agnósticas
│   ├── 01-fundamentos/       # Topics, Particiones, Offsets, Productores, Consumidores
│   ├── 02-mecanismos-internos/# Replicación, ISR, Elección de Líder, Rebalanceo
│   └── 03-ecosistema-avanzado/# Arquitectura KRaft, Métricas, Chaos Engineering
│
└── laboratorios/             # Misiones prácticas ejecutables con Docker
    ├── 01-.../               # Entornos mononodo y operaciones básicas
    ├── 02-.../               # Consumo avanzado y gestión de grupos
    └── 03-.../               # Clústeres multi-nodo, resiliencia y fallas simuladas
```

* **Fichas Teóricas (`teoria/`):** Documentos autocontenidos estructurados con definiciones cortas, analogías del mundo real, diagramas ASCII, tablas de mitos/errores y autoevaluaciones desplegables.
* **Laboratorios Prácticos (`laboratorios/`):** Escenarios paso a paso basados en casos de uso de la industria (ej. e-commerce *TechStore*), listos para desplegarse mediante `docker compose up`.

---

## 📚 Mapa de Módulos

El contenido se organiza de forma progresiva:

| Módulo | Enfoque Principal | Conceptos Clave |
| :--- | :--- | :--- |
| **01. Fundamentos** | Introducción a la mensajería de eventos | Brokers, Topics, Particiones, Offsets, Productores, Consumidores. |
| **02. Mecanismos Internos** | Replicación y consistencia de datos | Factor de Replicación, Líder/Followers, ISR, ACKs, Semánticas de Entrega. |
| **03. Ecosistema Avanzado** | Alta disponibilidad y operación en producción | Quórum KRaft, `min.insync.replicas`, Failover, Chaos Engineering. |
| **04. Casos de Uso** | Aplicación en la industria | Arquitecturas de microservicios, analítica en tiempo real y logs. |

---

## 🚀 Guía de Inicio Rápido

1. **Clona el repositorio:**
   ```bash
   git clone [https://github.com/tu-usuario/kafka-for-beginners.git](https://github.com/tu-usuario/kafka-for-beginners.git)
   cd kafka-for-beginners
   ```

2. **Explora la teoría:**
   Comienza leyendo los fundamentos en [`teoria/01-fundamentos/topics.md`](teoria/01-fundamentos/topics.md).

3. **Ejecuta tu primer laboratorio:**
   Navega a la carpeta del primer laboratorio y despliega el entorno con Docker:
   ```bash
   cd laboratorios/01-introduccion-conduktor
   docker compose up -d
   ```

---

## 🤝 Contribuciones

¡Las contribuciones son bienvenidas! Si encuentras un error en algún comando, quieres mejorar una analogía o proponer un nuevo escenario de laboratorio:

1. Haz un *Fork* del repositorio.
2. Crea una rama para tu mejora (`git checkout -b feature/nueva-ficha`).
3. Envía un *Pull Request* siguiendo la guía de estilo definida en las plantillas.
