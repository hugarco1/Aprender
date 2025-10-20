# 🧱 Proyecto: Sistema de Sincronización Descentralizada para Mundos de Minecraft

## 📝 Descripción General

Este proyecto propone el desarrollo de un sistema que permita a múltiples jugadores compartir y modificar un mismo mundo de Minecraft de forma **descentralizada y asíncrona**, **sin necesidad de un servidor Minecraft tradicional**.

El enfoque se basa en que cada jugador utilice su propia copia del mundo, y los cambios realizados se sincronicen entre los distintos clientes a través de un **servidor intermedio ligero**, que actúa únicamente como punto de sincronización de cambios, **sin ejecutar Minecraft ni hostear el mundo**.

---

## 🎯 Objetivos del Proyecto

- Permitir a los jugadores jugar en el mismo mundo de Minecraft sin estar conectados simultáneamente.
- Eliminar la necesidad de un servidor Minecraft dedicado.
- Facilitar la sincronización de los cambios realizados por cada jugador en su copia del mundo.
- Habilitar la sincronización en tiempo real entre jugadores si están conectados al mismo tiempo.
- Ofrecer una arquitectura más flexible para el juego cooperativo, permitiendo tanto sesiones sincrónicas como asincrónicas.
- Reducir los requisitos de infraestructura, recursos y configuración de red frente al multijugador tradicional.

---

## 📦 Arquitectura Propuesta

### Componentes Principales

1. **Cliente Minecraft con Mod de Sincronización**
   - Mod instalado en cada cliente.
   - Detecta y registra cambios en el mundo local (bloques, entidades, inventario, etc.).
   - Envia los cambios al servidor de sincronización cuando el jugador está conectado.
   - Recibe y aplica los cambios realizados por otros jugadores.

2. **Servidor de Sincronización**
   - Servicio externo ligero (no es un servidor Minecraft).
   - Recibe y almacena los cambios enviados por los jugadores.
   - Permite a los clientes consultar y recuperar los cambios que aún no han aplicado.
   - Gestiona posibles conflictos y realiza seguimiento del estado del mundo para cada cliente.
   - Actúa como intermediario para la sincronización en tiempo real entre clientes si están activos simultáneamente.

3. **Base de Datos de Cambios**
   - Almacena todos los cambios reportados por los jugadores.
   - Cada cambio se almacena con metadatos (jugador, marca de tiempo, zona del mundo, tipo de cambio, etc.).
   - Puede guardar historial completo o solo el estado actual más reciente, según configuración.

---

## 🔧 Funcionamiento General

### Juego Individual

Cada jugador puede jugar con su copia local del mundo, sin necesidad de conectarse a Internet ni al servidor. Durante el juego, los cambios se registran localmente mediante el mod instalado.

### Sincronización Posterior

Cuando el jugador vuelve a tener conexión, el mod:

- Envía al servidor todos los cambios locales aún no sincronizados.
- Consulta al servidor si hay cambios realizados por otros jugadores que aún no están aplicados localmente.
- Descarga y aplica esos cambios en el mundo local del jugador.

Este proceso puede ejecutarse de forma automática al iniciar el juego, o manual bajo decisión del usuario.

### Sincronización en Tiempo Real (Opcional)

Si varios jugadores están activos al mismo tiempo y conectados, el mod puede:

- Consultar el servidor de forma periódica o mantener una conexión abierta.
- Enviar cambios en tiempo real conforme ocurren.
- Aplicar los cambios recibidos de otros jugadores en vivo, permitiendo una experiencia similar al multijugador tradicional, pero sin conexión directa entre clientes ni servidor de juego.

---

## 📡 Justificación del Uso de un Servidor Intermedio

Aunque el sistema se aleja del modelo cliente-servidor tradicional de Minecraft, se utiliza un **servidor intermedio** por las siguientes razones:

- Permite desacoplar la ejecución del juego de la lógica de sincronización.
- Evita los problemas asociados al networking P2P puro (NAT traversal, descubrimiento de nodos, firewalls, etc.).
- Ofrece un punto de coordinación que permite gestionar conflictos y mantener la integridad del mundo compartido.
- Requiere menos recursos que hostear un servidor Minecraft y puede ejecutarse en hardware modesto (incluso Raspberry Pi o VPS baratos).
- Permite almacenar historial de cambios o versiones del mundo para recuperación o auditoría.

Este servidor **no ejecuta Minecraft**, ni procesa la lógica del juego, ni almacena una copia completa del mundo. Solo maneja los "deltas" o cambios incrementales.

---

## 🧠 Detalles Técnicos del Sistema

### Mod de Cliente

El mod instalado en el cliente debe ser capaz de:

- Detectar eventos relevantes del juego:
  - Colocación y destrucción de bloques.
  - Movimiento y creación/eliminación de entidades.
  - Cambios en el inventario de jugadores y contenedores.
- Registrar esos cambios en un formato estructurado (por ejemplo, JSON o binario).
- Almacenar los cambios localmente cuando no hay conexión.
- Enviar los cambios pendientes al servidor de sincronización.
- Consultar y aplicar los cambios ajenos obtenidos del servidor.
- Resolver automáticamente los conflictos simples (por ejemplo, prioridad por marca de tiempo).
- Notificar o solicitar intervención del jugador en caso de conflictos más complejos.

### Servidor de Sincronización

El servidor debe exponer una API que permita a los clientes:

- Subir cambios realizados localmente (`POST /pushChanges`)
- Descargar cambios ajenos aún no aplicados (`GET /pullChanges`)
- Consultar estado de sincronización (`GET /status`)
- Resolver conflictos de sincronización (`POST /resolveConflict`)

El servidor también debe:

- Autenticar a los jugadores (si se requiere seguridad).
- Verificar la integridad de los datos recibidos.
- Validar que los cambios sean consistentes con el mundo.
- Aplicar reglas de prioridad en caso de conflictos (ej. último cambio, jugador autorizado, merge manual).
- Registrar logs o historial para permitir reversión si es necesario.

---

## 📌 Consideraciones Importantes

### Consistencia del Mundo

- Es fundamental evitar que los mundos de los distintos jugadores diverjan permanentemente.
- Cada cliente debe aplicar correctamente todos los cambios realizados por los demás jugadores, en el mismo orden lógico.
- La integridad del mundo debe validarse periódicamente (ej. checksum de regiones o chunks).

### Conflictos

- Conflictos ocurren cuando dos jugadores modifican la misma zona del mundo (mismo bloque, misma entidad, etc.) sin sincronizarse entre sí.
- El sistema debe definir reglas claras para su resolución automática o solicitar la intervención del jugador en caso de conflicto grave.

### Seguridad

- Aunque el sistema está pensado para grupos reducidos y entornos de confianza, debe contemplar posibles ataques o corrupción involuntaria:
  - Validación de cambios antes de aplicarlos.
  - Autenticación básica de clientes.
  - Firma de paquetes (opcional).
  - Limitación del acceso al servidor.

### Escalabilidad

- El sistema está diseñado para entornos con pocos jugadores (2–5), no como solución de reemplazo de servidores masivos.
- Sin embargo, la arquitectura puede ampliarse con ciertas optimizaciones:
  - Almacenamiento por regiones.
  - Indexación eficiente de cambios.
  - Mecanismos de compresión y delta-pack.

---

## 📌 Beneficios de Esta Arquitectura

- **Independencia:** los jugadores no dependen de una conexión permanente.
- **Flexibilidad:** pueden jugar cuando quieran, incluso offline.
- **Simplicidad técnica:** evita los problemas de configurar y mantener servidores Minecraft.
- **Privacidad:** los mundos se almacenan localmente, sin exposición pública.
- **Innovación:** permite una nueva forma de juego cooperativo en Minecraft, combinando asincronía y descentralización.

---

## 📣 Evaluación y Retroalimentación

Este documento tiene como fin presentar la idea con claridad para evaluación por parte de desarrolladores, diseñadores o jugadores técnicos.

Se agradecen comentarios sobre:

- Factibilidad técnica y experiencia de usuario
- Riesgos o limitaciones no contempladas
- Sugerencias de mejora en la arquitectura o implementación
- Ideas adicionales para enriquecer la experiencia

---

## 🏁 Conclusión

Esta propuesta busca introducir una forma alternativa de compartir mundos de Minecraft que no depende de un servidor tradicional, permitiendo sesiones de juego más libres y adaptadas a estilos cooperativos asincrónicos.

Se trata de un sistema técnicamente viable, modular, seguro, y aplicable a escenarios reales de juego entre amigos, comunidades pequeñas o entornos educativos.

