## Handling Exceptions
- En el caso del producer, envolver las excepciones en un bloque try/catch y no enviarlas a Kafka si se produce un error
- En el caso del consumer, por defecto, Spring boot loggea la excepción
- Podemos implementar nuestro propio error handler
- Ejemplo
  - Publicamos un pedido de productos
  - Se lanza una excepción si la cantidad de productos es no válido

---

## 1. Crea el topic t-orden-producto

```bash
docker compose -f docker-compose-core-yml
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-orden-producto --partitions 1
```

---

## 2. Crea la clase OrdenProducto en el proyecto core-producer

```java
public class OrdenProducto {

  private int cantidad;
  private String producto;
    
}
```

---

## 3. En el proyecto producer crea la clase OrdenProductoProducer

```java
@Service
public class OrdenProductoProducer {

  @Autowired
  private KafkaTemplate<String, String> kafkaTemplate;

  @Autowired
  private ObjectMapper objectMapper;

  public void send(OrdenProducto ordenProducto) throws JsonProcessingException {
    var json = objectMapper.writeValueAsString(ordenProducto);

    kafkaTemplate.send("t-orden-producto", json);
  }

}
```

---

## 4. Crear Ordenes de producto en la clase principal

```java
import java.util.UUID;

@Autowired
private OrdenProductoProducer ordenProductoProducer;

@Override
public void run(String... args) throws Exception {
  OrdenProducto zapatillas = new OrdenProducto(3, "zapatillas");
  OrdenProducto colonia = new OrdenProducto(10, "colonia");
  OrdenProducto camisetas = new OrdenProducto(5, "camisetas");

  ordenProductoProducer.send(zapatillas);
  ordenProductoProducer.send(colonia);
  ordenProductoProducer.send(camisetas);
  
}
```

- Ejecuta el productor.
- Copia la clase `OrdenProducto` en el proyecto consumer.

---

## 5. Crea la clase OrdenProductoConsumer en el proyecto consumer

```java
@Service
@Slf4j
public class OrdenProductoConsumer {

  private static final int MAX_CANTIDAD_PRODUCTO = 7;

  @Autowired
  private ObjectMapper objectMapper;

  @KafkaListener(topics = "t-orden-producto")
  public void consume(String message) throws JsonMappingException, JsonProcessingException {
    var ordenProducto = objectMapper.readValue(message, OrdenProducto.class);
    if (ordenProducto.getAmount() > MAX_CANTIDAD_PRODUCTO) {
      throw new IllegalArgumentException("Demasiados productos : " + ordenProducto.getAmount());
    }

    log.info("Procesando una orden de producto : {}", ordenProducto);
  }
}

```

- Cuando consumamos la orden para colonia, se lanzará una excepción.
- Podemos crear una lógica personalizada para gestionar la excepción.
- Para ello necesitamos un error handler.
- Podemos usar el error handler dentro de la anotación `@KafkaListener`

---

## 6. Crea la clase OrdenProductoErrorHandler para gestionar la excepción

```java
@Service(value = "ordenProductoErrorHandler")
@Slf4j
public class OrdenProductoErrorHandler implements ConsumerAwareListenerErrorHandler {

  @Override
  public Object handleError(Message<?> message, ListenerExecutionFailedException exception, Consumer<?, ?> consumer) {
    LOG.warn("Error en el procesado de una orden de producto, enviando el error a ElasticSearch : {}, causa : {}", message.getPayload(),
            exception.getMessage());
    
    return null;
  }

}

```

- Añade el errorHandler al código del Consumer

```java
@KafkaListener(topics = "t-orden-producto", errorHandler = "ordenProductoErrorHandler")
```
