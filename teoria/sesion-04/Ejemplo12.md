## Global Exception Handler
- En lugar de crear un handler para cada consumer de la aplicación, podemos crear un exception handler global para todos los consumers de la aplicación
- La gestión de errores se hará a nivel de contenedor de Spring
- Ejemplo
  - Publicamos número aleatorios
  - Si los números son impares, lanzamos una excepción
  - La manejaremos usando un error handler global

---

## 1. Crea el topic t-numeros-simples

```bash
docker compose -f docker-compose-core-yml
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-numero-simple --partitions 1
```

---

## 2. Crea la clase NumeroSimple en el proyecto core-producer

```java
public class NumeroSimple {

  private int numero;
    
}
```

---

## 3. En el proyecto producer crea la clase NumeroSimpleProducer

```java
@Service
public class NumeroSimpleProducer {

  @Autowired
  private KafkaTemplate<String, String> kafkaTemplate;

  @Autowired
  private ObjectMapper objectMapper;

  public void send(NumeroSimple numeroSimple) throws JsonProcessingException {
    var json = objectMapper.writeValueAsString(numeroSimple);
    kafkaTemplate.send("t-numero-simple", json);
  }

}
```

---

## 4. Crear Numeros Simples en la clase principal

```java
import java.util.UUID;

@Autowired
private OrdenProductoProducer ordenProductoProducer;

@Auwired
private NumeroSimpleProducer numeroSimpleProducer;

@Override
public void run(String... args) throws Exception {
  OrdenProducto zapatillas = new OrdenProducto(3, "zapatillas");
  OrdenProducto colonia = new OrdenProducto(10, "colonia");
  OrdenProducto camisetas = new OrdenProducto(5, "camisetas");

  ordenProductoProducer.send(zapatillas);
  ordenProductoProducer.send(colonia);
  ordenProductoProducer.send(camisetas);
  
  for(int i = 100; i < 103; i++) {
      var numeroSimple = new NumeroSimple(i);
      
      numeroSimpleProducer.send(numeroSimple);
  }
}
```

- Ejecuta el productor.
- Copia la clase `NumeroSimple` en el proyecto consumer.

---

## 5. Crea la clase NumeroSimpleConsumer en el proyecto consumer

```java
@Service
@Slf4j
public class NumeroSimpleConsumer {

  @Autowired
  private ObjectMapper objectMapper;

  @KafkaListener(topics = "t-numero-simple")
  public void consume(String message) throws JsonMappingException, JsonProcessingException {
    var numeroSimple = objectMapper.readValue(message, NumeroSimple.class);

    if (numeroSimple.getNumero() % 2 != 0) {
      throw new IllegalArgumentException("Numero impar : " + numeroSimple.getNumero());
    }

    LOG.info("Procesando el numero simple : {}", numeroSimple);
  }
}

```

- Si ahora ejecutamos el consumer y el producer podremos ver como se lanza una excepción al procesar el segundo número
- Spring es el encargado de logear esta excepción ya que no hemos creado ningún exception handler de momento

---

## 6. Crea la clase GlobalErrorHandler en el paquete error del proyecto consumer

```java
@Service
@Slf4j
public class GlobalErrorHandler implements CommonErrorHandler {

  @Override
  public boolean handleOne(Exception thrownException, ConsumerRecord<?, ?> record, Consumer<?, ?> consumer, MessageListenerContainer container) {
    log.error("handleOne() for : {}", record.value().toString());
    return true;
  }

  @Override
  public void handleOtherException(Exception thrownException, Consumer<?, ?> consumer, MessageListenerContainer container, boolean batchListener) {
    log.error("handleOtherException() for : {}", thrownException.getMessage());
  }
}
```

- Ahora si volvemos a ejecutar el consumer deberíamos ver como el global exception handler gestiona las excepciones
- Podemos ver como el global exception handler gestiona la excepción para los número impares pero nó para las ordenes de productos
- Si queremos que la excepción para las ordenes de productos sea gestionada también por el common error handler, deberemos relanzarla


```java
@Service(value = "ordenProductoErrorHandler")
@Slf4j
public class OrdenProductoErrorHandler implements ConsumerAwareListenerErrorHandler {

  @Override
  public Object handleError(Message<?> message, ListenerExecutionFailedException exception, Consumer<?, ?> consumer) {
    LOG.warn("Error en el procesado de una orden de producto, enviando el error a ElasticSearch : {}, causa : {}", message.getPayload(),
            exception.getMessage());

    if (exception.getCause() instanceof RuntimeException) {
      throw exception;
    }
    return null;
  }

}
```

- Si ahora volvemos a relanzar los dos proyectos, veremos como las excepciones para las ordenes de productos son tratadas dos veces

