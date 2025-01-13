## Contexto

Imagina que trabajas en el desarrollo de un sistema de gestión de pedidos para una tienda en línea. 
Los eventos del sistema incluyen la creación de pedidos, actualizaciones de stock y notificaciones para los clientes. 
El objetivo es diseñar e implementar una arquitectura con Apache Kafka que asegure el procesamiento eficiente y 
confiable de estos eventos.

## Requisitos
### Creación de Topics

Crea los siguientes topics con el comando kafka-topics.sh:
- t-pedido con 3 particiones.
- t-stock con 2 particiones.
- t-notificaciones con 1 partición.

### Productores

Implementa tres productores

#### PedidoProducer

- Produce mensajes JSON con los datos de pedidos (ID, cliente, total, y lista de productos).
- Asegúrate de que los mensajes tengan una clave basada en el ID del cliente para garantizar que todos los pedidos de un cliente vayan a la misma partición.

#### StockProducer

- Produce mensajes JSON para actualizar el stock de productos.
- Envía actualizaciones cada 5 segundos con un Scheduled interval.

#### NotificacionesProducer

- Produce mensajes para enviar notificaciones a los clientes cuando se crea un pedido.

### Consumers

Implementa consumidores en grupos para cada topic

#### PedidoConsumer

- Procesa los pedidos de manera secuencial.
- Si el importe total es superior a 1000€, lanza una excepción simulando un error. Maneja la excepción utilizando un handler global y registra los errores en un log.
- Cada vez que un pedido se procese con éxito:
  - Envía un mensaje al topic t-notificaciones con los datos relevantes (ID del pedido, ID del cliente y mensaje de confirmación).
  - Este mensaje será consumido posteriormente por el NotificacionesConsumer.

#### StockConsumer

- Consume los mensajes de stock y los procesa en tiempo real.
- Utiliza un filtro para descartar actualizaciones de stock con cantidades menores a 10.

#### NotificacionesConsumer

- Consume notificaciones y las procesa con un retardo aleatorio para simular latencia.

## Semánticas de Consumo

- Configura la aplicación para usar auto-offset-reset en modo earliest para los consumidores.
- Establece ack-mode a record para hacer commit después de procesar cada mensaje en PedidoConsumer.
- Establece ack-mode a batch 5 para NotificationConsumer.

## Dead Letter Topic

- Configura un Dead Letter Topic para t-pedido llamado t-pedido-dlt.
- Implementa una estrategia para enviar los mensajes fallidos al Dead Letter Topic después de 3 intentos.


## Modelos

### Pedido

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Pedido {
    private String id;
    private String clienteId;
    private double total;
}
```

### Notificación

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Notificacion {
    private String pedidoId;
    private String clienteId;
    private String mensaje;
}
```

### Stock

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Stock {
    private String productoId;
    private int cantidadDisponible;
    private long timestamp;
}
```

## Dependencias

```xml
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-core</artifactId>
  <version>2.18.2</version>
</dependency>

<dependency>
   <groupId>com.fasterxml.jackson.datatype</groupId>
   <artifactId>jackson-datatype-jsr310</artifactId>
   <version>2.18.2</version>
</dependency>

<dependency>
   <groupId>org.springframework.kafka</groupId>
   <artifactId>spring-kafka</artifactId>
</dependency>

<dependency>
   <groupId>org.projectlombok</groupId>
   <artifactId>lombok</artifactId>
   <optional>true</optional>
</dependency>
```