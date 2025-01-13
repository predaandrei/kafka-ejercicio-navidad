## Dead Letter Topic
- En los casos en los que el fallo del procesado no se debe a un error temporal, debemos procesar los mensajes de manera diferente
- En este caso envíamos los mensajes a un dead letter topic
- Estos mensajes son conocidos como dead letter records
- Spring tiene una clase denomianda `DeadLetterPublishingRecoverer` con la que podemos lograr esta funcionalidad
- El `DeadLetterPublishingRecoverer` enviará los mensajes fallidos a un topic creado automáticamente denominado `nombre-del-topic.DLT`
- En este ejemplo crearemos nuestro propio dead letter topic

---

## 1. Crea el topic t-factura y t-factura-dead con dos particiones

- El dead letter topic debe contener al menos el mismo número de particiones que el topic original

```bash
docker compose -f docker-compose-core-yml
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-factura --partitions 2
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-factura-dead --partitions 2
```

---

## 2. Crea la clase Factura en el proyecto core-producer

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Factura {

    private String numeroFactura;
    private double cantidad;
    private String divisa;

}
```

---

## 3. En el proyecto producer crea la clase FacturaProducer

```java
@Service
public class FacturaProducer {

  @Autowired
  private KafkaTemplate<String, String> kafkaTemplate;

  @Autowired
  private ObjectMapper objectMapper;

  public void sendFactura(Factura factura) throws JsonProcessingException {
    var json = objectMapper.writeValueAsString(factura);
    kafkaTemplate.send("t-factura", (int) factura.getCantidad() % 2, factura.getNumeroFactura(), json);
  }
}
```

---

## 4. Crear la clase FacturaService en el paquete service del proyecto producer

```java
import java.util.concurrent.atomic.AtomicInteger;

@Service
public class FacturaService {

  private AtomicInteger counter = new AtomicInteger();

  public Factura generaFactura() {
    var numeroFactura = "FAC-" + counter.incrementAndGet();
    var cantidad = ThreadLocalRandom.current().nextInt(1, 1000);

    return new Factura(numeroFactura, cantidad, "EUR");
  }
}
```
---

## 5. Genera facturas en la clase principal

```java
@Autowired
private FacturaService facturaService;

@Autowired
private FacturaProducer facturaProducer;

@Override
public void run(String... args) throws Exception {
  for(int i = 0; i < 10; i++) {
      var factura = facturaService.generaFactura();
      
      for(i > 5) {
          factura.setCantidad(0d);
      }
      
      facturaProducer.sendFactura(factura);
  }
}

```
- Copia la clase `Factura` en el proyecto consumer
- Ejecuta el producer y páralo
- Comprueba el contenido de las particiones con el consumer de la consola

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-factura --offset earliest --partition 0
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-factura --offset earliest --partition 1
```

---

## 6. Crea la clase FacturaConsumer

```java
@Service
@Slf4j
public class FacturaConsumer {

  @Autowired
  private ObjectMapper objectMapper;

  @KafkaListener(topics = "t-factura", concurrency = "2", containerFactory = "facturaDltContainerFactory")
  public void consume(String message) throws JsonMappingException, JsonProcessingException {
    var factura = objectMapper.readValue(message, Factura.class);

    if (factura.getCantidad() < 1) {
      throw new IllegalArgumentException("Cantidad incorrecta para la factura " + factura);
    }

    log.info("Procesando la factura : {}", factura);
  }

}
```
- Crea el bean `FacturaDltContainerFactory` dentro de la clase KafkaConfig
```java
@Bean(name = "facturaDltContainerFactory")
public ConcurrentKafkaListenerContainerFactory<Object, Object> facturaDltContainerFactory(
        ConcurrentKafkaListenerContainerFactoryConfigurer configurer, KafkaTemplate<String, String> kafkaTemplate) {
  var factory = new ConcurrentKafkaListenerContainerFactory<Object, Object>();
  configurer.configure(factory, consumerFactory());

  var recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate, 
          (record, ex) -> new TopicPartition("t-factura-dead", record.partition()));

  factory.setCommonErrorHandler(new DefaultErrorHandler(recoverer, new FixedBackOff(1000, 5)));

  return factory;
}
```

- Comprueba utilizando el consumer de la consola que las 2 particiones `t-factura-dead` están vacías
```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-factura-dead --offset earliest --partition 0
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-factura-dead --offset earliest --partition 1
```
- Inicia ahora el consumer y a continuación vuelve a comprobar todas las particiones para ver su contenido
- El dead letter topic es un topic más por lo que podemos crear consumers específicos que consuman los mensajes de este topic y los procesen