## Contexto
Tu empresa ha decidido implementar un sistema de procesamiento de pedidos en tiempo real utilizando Apache Kafka. La idea es que un productor publique información sobre los pedidos realizados por los clientes, y varios consumidores procesen esta información para realizar diferentes tareas: actualizar inventarios y notificar a los clientes.

## Objetivo
Implementar un sistema de Kafka con:

- Un productor que envíe información sobre pedidos a un topic llamado t-pedidos.
- Dos consumidores que procesen los pedidos desde distintos grupos de consumidores:
- Un grupo para actualizar inventarios (consumer-group-inventarios).
- Un grupo para notificar a los clientes (consumer-group-notificaciones).

## Requisitos
### 1. Crea el topic t-pedidos

- Usa la CLI de Kafka para crear un topic llamado t-pedidos con una sola partición.

### 2. Define la clase Pedido

Implementa una clase llamada Pedido con los siguientes atributos:
- pedidoId (String): identificador único del pedido.
- cliente (String): nombre del cliente.
- productos (Lista de Strings): lista de nombres de productos pedidos.
- total (double): monto total del pedido.
- fechaPedido (LocalDateTime): fecha y hora del pedido.

### 3. Implementa el Productor

- Crea un servicio Spring Boot para enviar pedidos al topic t-pedidos.
- Genera pedidos aleatorios con un intervalo de 3 segundos entre cada uno.
- Serializa los datos en formato JSON usando Jackson.

### 4. Implementa los Consumidores

Consumidor de Inventarios (consumer-group-inventarios):
- Lee los mensajes del topic y simula una reducción de los productos en stock.
- Loguea los productos procesados.

Consumidor de Notificaciones (consumer-group-notificaciones):
- Lee los mensajes del topic y simula el envío de una notificación al cliente.
- Loguea el mensaje enviado al cliente.

### 5. Configuración avanzada

Configura el application.yml de los consumidores con:
- `spring.kafka.consumer.auto-offset-reset=earliest` para asegurarte de procesar todos los mensajes desde el inicio.
- Usa el modo de reconocimiento (ack-mode) `batch` para el grupo de inventarios y `record` para el grupo de notificaciones.