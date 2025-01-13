## Retry
- Vamos a crear un handler con un mecanismo de reintento

---

## 1. Crea el topic t-file

```bash
docker compose -f docker-compose-core-yml
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-file --partitions 2
```

---

## 2. Crea la clase FileKafka en el proyecto core-producer

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class FileKafka {

    private String name;
    private long size;
    private String extension;

}
```

---

## 3. En el proyecto producer crea la clase FileKafkaProducer

```java
@Service
public class FileKafkaProducer {

  @Autowired
  private KafkaTemplate<String, String> kafkaTemplate;

  @Autowired
  private ObjectMapper objectMapper;

  public void sendFileToPartition(FileKafka fileKafka, int partition) throws JsonProcessingException {
    var json = objectMapper.writeValueAsString(fileKafka);
    kafkaTemplate.send("t-file", partition, fileKafka.getExtension(), json);
  }

}
```

---

## 4. Crear la clase FileKafkaService en el paquete service del proyecto producer

```java
import java.util.concurrent.atomic.AtomicInteger;

@Service
public class FileKafkaService {

  private static AtomicInteger counter = new AtomicInteger();

  public FileKafka generateFile(String extension) {
    var name = "file-" + counter.incrementAndGet();
    var size = ThreadLocalRandom.current().nextLong(100, 10_000);

    return new File(name, size, extension);
  }
}
```
---

## 5. Genera 6 objetos de tipo FileKafka en la clase principal

```java
@Autowired
private FileKafkaService fileKafkaService;

@Autowired
private FileKafkaProducer fileKafkaProducer;

@Override
public void run(String... args) throws Exception {
  FileKafka file1 = fileKafkaService.generateFile("pdf");
  FileKafka file2 = fileKafkaService.generateFile("xslt");
  FileKafka file3 = fileKafkaService.generateFile("xml");
  FileKafka file4 = fileKafkaService.generateFile("pdf");
  FileKafka file5 = fileKafkaService.generateFile("csv");
  FileKafka file6 = fileKafkaService.generateFile("txt");

  fileKafkaProducer.sendFileToPartition(file1, 0);
  fileKafkaProducer.sendFileToPartition(file2, 0);
  fileKafkaProducer.sendFileToPartition(file3, 0);

  fileKafkaProducer.sendFileToPartition(file4, 1);
  fileKafkaProducer.sendFileToPartition(file5, 1);
  fileKafkaProducer.sendFileToPartition(file6, 1);
}

```

- Ejecuta el producer y páralo
- Copia la clase FileKafka al proyecto consumer

---

## 6. Crea la clase FileKafkaConsumer

```java
@Service
@Slf4j
public class FileKafkaConsumer {

    @Autowired
    private ObjectMapper objectMapper;

    @KafkaListener(topics = "t-file", containerFactory = "fileRetryContainerFactory", concurrency = "2")
    public void consume(ConsumerRecord<String, String> consumerRecord) throws JsonMappingException, JsonProcessingException {
        var file = objectMapper.readValue(consumerRecord.value(), FileKafka.class);

        if (file.getExtension().equalsIgnoreCase("xslt")) {
            log.warn("Throwing exception on partition {} for file {}", consumerRecord.partition(), file);
            throw new IllegalArgumentException("Simulate API call failed");
        }

        LOG.info("Processing on partition {} for file {}", consumerRecord.partition(), file);
    }
}
```

- Crea el bean `FileRetryContainerFactory` dentro de la clase KafkaConfig


```java
@Bean(name = "fileRetryContainerFactory")
public ConcurrentKafkaListenerContainerFactory<Object, Object> fileRetryContainerFactory(
        ConcurrentKafkaListenerContainerFactoryConfigurer configurer) {
    var factory = new ConcurrentKafkaListenerContainerFactory<Object, Object>();
    configurer.configure(factory, consumerFactory());

    factory.setCommonErrorHandler(new DefaultErrorHandler(new FixedBackOff(10_000, 3)));

    return factory;
}
```

- Al ejecutar el proyecto producer y el consumer nos damos cuenta de que
  - La segunda imagen en la partición 0 lanza una excepción
  - El consumer esperará 10 segundos y volverá a reintentar
  - Después de 3 reintentos el procesado falla y el consumer continuará procesando el siguiente mensaje en la partición 0
  - En la partición 1 todos los mensajes son procesados sin problema
  - Este tipo de reintentos es conocido como blocking retry
    - El bloqueo es adecuado cuando los mensajes necesitan ser procesados secuencialmente
    - Puede provocar cuellos de botella al detener el procesado
    - Para reducir este inconveniente, debemos reintentar lo justo
    - Se bloquea solo la partición que contiene errores