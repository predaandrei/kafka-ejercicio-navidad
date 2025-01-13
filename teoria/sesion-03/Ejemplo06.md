## Período de Retención
- El consumer lee los mensajes secuencialmente
- El consumer puede caerse
- Kafka tiene un período de retención que por defecto es de 7 días
  - Si el consumer está caído por más de 7 días, el offset es inválido porque los datos ya no existen
- El período de retención se puede configurar con la propiedad `offset.retention.minutes`

---

## Auto Offset Reset

- El comportamiento de un nuevo consumer group ID se determina desde Spring Kafka con la propiedad `spring.kafka.consumer.auto-offset-reset`
- Los posibles valores son
  - earliest: Se empiezan a consumir los mensajes desde el ultimo offset registrado, si no hay ningún offset se empieza en el offset 0
  - latest (default): Se empiezan a consumir los mensajes desde el último offset registrado, pero teniendo en cuenta que sólo consumira mensajes desde el momento en que es iniciado e ignorará los anteriores 
  - none: Se lanza una excepción si no hay offset o el offset está fuera de rango. Con esto nos aseguramos de que no se consume ningún mensaje si el offset es inválido

- El consumer lag se refiere a la diferencia que hay entre el último mensaje y el mensaje que ha consumido el consumer group
- Se calcula restando el último mensaje y el offset
- Sirve para monitorizar e identificar posibles cuellos de botella si su valor es demasiado alto

---

## Replay Data

- Reprocesar mensajes consumidos previamente
- Casos de uso
  - Corregir errores
  - Recuperar datos
  - Análisis histórico
  - Testing
- Para hacer el replay hay que resetear el offset del consumer a una posición previa
- Consumir los mensajes desde el offset reseteado en adelante

---

### 1. Crear un topic llamado t-contador con 1 partición

```bash
docker compose -f docker-compose-core-yml up -d
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-contador --partitions 1
```

---

### 2. Crear la clase ContadorProducer dentro del producer

```java
@Service
public class ContadorProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void send(int number) {
        for (int i = 0; i < number; i++) {
            String message = "Data " + i;
            kafkaTemplate.send("t-contador", message);
        }
    }
}
```

---

### 3. Modificar la clase principal para enviar datos

```java

@Autowired
private ContadorProducer producer;

@Override
public void run(String... args) throws Exception {
    producer.send(100);
}
```

- Ejecutar la aplicación

---

### 4. Crear ContadorConsumer en la aplicación de consumer

```java
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class ContadorConsumer {
    
  @KafkaListener(topics = "t-contador", groupId = "contador-group-fast")  
  public void readFast(String message) {
    log.info("Fast: " + message);
  }

  @KafkaListener(topics = "t-contador", groupId = "contador-group-slow")
  public void readSlow(String message) throws InterruptedException {
    TimeUnit.SECONDS.sleep(3);
    log.info("Slow: " + message);
  }
}
```

- Comprobar que el topic t-contador tiene 100 mensajes

```bash
kafka-console-consumer.sh --from-beginning --bootstrap-server localhost:9092 --property print.key=false --property print.value=false --topic t-contador --timeout-ms 5000 | tail -n 10|grep "Processed a total of"
```

- Describe los grupos de kafka
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group contador-group-fast --describe
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group contador-group-slow --describe
```
- Ambos comandos lanzarán un error debido a que los grupos no han sido creados
- En el consumer comprobar que la propiedad `auto-offset-reset` esta seteada a earliest
- Iniciar el proyecto consumer y pararlo después de unos segundos (cuando el contador-group-slow haya consumido algunos mensajes)
- Volver a inicial el proyecto consumer:
  - El contador-group-fast no procesa ningún mensaje porque ya los había procesado todos
  - El contador-group-slow empieza a procesar los mensajes desde el principio porque no había hecho commit del offset y el `auto-offset-reset` esta a earliest
  - La configuración de spring kafaka offset hará que solo se guarde el offset si se han procesado todos los mensajes del topic

- Describir los grupos de kafka de nuevo
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group contador-group-fast --describe
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group contador-group-slow --describe
```

- Se puede ver como el contador-group-fast no tiene lag porque todos los mensajes han sido consumidos
- En el caso del contador-group-slow dará un error porque aún no se ha guardado el offset

---

### 5. Modificar la configuración para guardar el Offset

<b>OPCIONES</b>
- Establecerlo en el application.yml `spring.kafka.enable.auto.commit`
  - true: se guarda el offset de acuerdo a la configuración del broker
  - false (default): se establece la forma de guardado manualmente
- Para establecer el ack mode manualmente se puede usar la propiedad `spring.kafka.listener.ack-mode`
  - record: se guarda el offset después de procesar un record
  - batch (default): se guarda el offset si todos los mensajes en el batch se han procesado
  - time: se guarda el offset cuando todos los mensajes en el pull se han procesado si el tiempo transcurrido es mayor al ack time
  - count: se guarda el offset cuando todos los mensajes en el pull se han procesado si el número de mensajes procesado es mayor al valor de count
  - count_time: se guarda el offset si la condición time o la condición count son verdaderas
  - manual: se debe hacer el commit explícitamente después de procesar los mensajes
  - manual_immediate: el ack es inmediato después de hacer commit del offset a kafka

Modifica `spring.kafka.listener.ack-mode` en el `application.yml` del consumer

```yaml
spring:
  kafka:
    listener:
      ack-mode: record
```

- Con esta configuración el offset será guardado justo después de procesar cada mensaje
- Ejecuta el consumer, deja que el `contador-group-slow` procese algunos mensajes y para el consumer
- Si volvemos a ejecutar el consumer podemos ver cómo continúa procesando los mensajes desde el último que procesó
- Describe el `contador-group-slow`
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group contador-group-slow --describe
```

- Podemos ver que el current-offset tiene un valor y que tenemos un lag
- Modifica el valor de `spring.kafka.listener-ack-mode` a batch y configura el valor de `spring.kafka.consumer.max-poll-records`

```yaml
spring:
  kafka:
    listener:
      ack-mode: batch
    consumer:
      max-poll-records: 10
```
- Si ejecutamos ahora el consumer y lo detenemos antes de que se hayan procesado 10 mensajes podemos ver como el offset no ha sido guardado
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group contador-group-slow --describe
```
- Si volvemos a ejecutar el consumer veremos que vuelve a procesar los mensajes desde el principio
- Si le dejamos procesar 10 mensajes veremos como el offset ahora si, ha sido guardado
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group contador-group-slow --describe
```
- Vuelve a establecer el `spring.kafka.listener-ack-mode` a `record`

---

### 6. Configurar un ack-mode para cada consumer

Crea la clase de configuración de Kafka

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "default-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean(name = "recordAckFactory")
    public ConcurrentKafkaListenerContainerFactory<String, String> recordAckFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
        return factory;
    }

    @Bean(name = "batchAckFactory")
    public ConcurrentKafkaListenerContainerFactory<String, String> batchAckFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);
        return factory;
    }
}

```

Asigna una factoria distinta a cada consumer

```java
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class ContadorConsumer {

  @KafkaListener(topics = "t-contador", groupId = "contador-group-fast", containerFactory = "recordAckFactory")
  public void readFast(String message) {
    log.info("Fast: " + message);
  }

  @KafkaListener(topics = "t-contador", groupId = "contador-group-slow", containerFactory = "batchAckFactory")
  public void readSlow(String message) throws InterruptedException {
    TimeUnit.SECONDS.sleep(3);
    log.info("Slow: " + message);
  }
}

```