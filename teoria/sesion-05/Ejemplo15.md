## Reintento no bloqueante
- Vamos a crear un handler con un mecanismo de reintento no bloqueante
- De este modo, cuando un mensaje da error, el reintento se produce en segundo plano mientras el resto de mensajes continúan siendo procesados

---

## 1. Crea el topic t-file-2

```bash
docker compose -f docker-compose-core-yml
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-file-2 --partitions 2
```

---

## 2. Copia la clase FileKafkaProducer y renómbrala a FileKafka2Producer
- Modifica el topic para que use el topic `t-file-2`

```java
@Service
public class FileKafka2Producer {

  @Autowired
  private KafkaTemplate<String, String> kafkaTemplate;

  @Autowired
  private ObjectMapper objectMapper;

  public void sendFileToPartition(FileKafka fileKafka, int partition) throws JsonProcessingException {
    var json = objectMapper.writeValueAsString(fileKafka);
    kafkaTemplate.send("t-file-2", partition, fileKafka.getExtension(), json);
  }

}
```
---

## 3. Genera 6 objetos de tipo FileKafka en la clase principal

```java
@Autowired
private FileKafkaService fileKafkaService;

@Autowired
private FileKafka2Producer fileKafkaProducer;

@Override
public void run(String... args) throws Exception {
  FileKafka file1 = fileKafkaService.generateFile("pdf");
  FileKafka file2 = fileKafkaService.generateFile("xslt");
  FileKafka file3 = fileKafkaService.generateFile("xml");
  FileKafka file4 = fileKafkaService.generateFile("pdf");
  FileKafka file5 = fileKafkaService.generateFile("csv");
  FileKafka file6 = fileKafkaService.generateFile("txt");

  fileKafkaProducer.sendToPartition(file1, 0);
  fileKafkaProducer.sendToPartition(file2, 0);
  fileKafkaProducer.sendToPartition(file3, 0);

  fileKafkaProducer.sendToPartition(file4, 1);
  fileKafkaProducer.sendToPartition(file5, 1);
  fileKafkaProducer.sendToPartition(file6, 1);
}

```

- Ejecuta el producer y páralo

---

## 6. Copia la clase FileKafkaConsumer y renómbrala como FileKafka2Consumer
- Añade la anotación `@RetryableTopic` y configúrala

```java
@Service
@Slf4j
public class FileKafkaConsumer {

    @Autowired
    private ObjectMapper objectMapper;

    @RetryableTopic(
            autoCreateTopic = "true",
            attempts = "4",
            topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_IDEX_VALUE,
            backoff = @Backoff(
                    delay = 3000,
                    maxDelay = 10000,
                    multiplier = 1.5,
                    random = true
            ),
            dltTopicSuffix = "-dead"
    )
    @KafkaListener(topics = "t-file-2", containerFactory = "fileRetryContainerFactory", concurrency = "2")
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

- Ejecuta el consumer y examina los logs.
- En la partición 1 no tenemos ningún error y todos los mensajes han sido correctamente procesados
- En la partición 0 podemos ver cómo tras el error del segundo mensaje, el mensaje 3 es procesado casi inmediatamente
- Tras tres reintentos, el segundo mensaje falla definitivamente y es enviado al dead letter topic