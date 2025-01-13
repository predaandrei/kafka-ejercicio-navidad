### 1. Crear el topic t-intervalo-fijo
- Indicar además que crear los topics es opcional, ya que debido a la configuración que tenemos en el docker compose, los topics se crearán automáticamente si no existen la primera vez que se publique en ellos.

```bash
docker compose -f docker-compose-core-yml up -d
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-intervalo-fijo --partitions 1
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

---

### 2. Crear el producer IntevaloFijoProducer

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class IntevaloFijoProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    private int i = 0;

    @Scheduled(fixedRate = 1000)
    public void sendMessage() {
        i++;
        log.info("i is {}", i);
        kafkaTemplate.send("t-intervalo-fijo", "Mensaje de intervalo fijo " + i);
    }
}
```

---

### 3. Modificar la clase principal de kafka-core-producer para ejecutar el producer

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class KafkaCoreProducerApplication implements CommandLineRunner {

    public static void main(String[] args) {
        SpringApplication.run(KafkaCoreProducerApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
    }
}
```

- Ejecutar KafkaCoreproducer
- Detener el KafkaCoreProducer para que no se nos pete Kafka

---

### 4. Crear la clase IntervaloFijoConsumer

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class IntervaloFijoConsumer {

    @KafkaListener(topics = "t-intervalo-fijo")
    public void consume(String message) {
        log.info("Consumiendo el mensaje: {}", message);
    }
}
```

- Ejecutar KafkaCoreConsumer
- Como el producer está parado no consumimos ningún mensaje a pesar de que ya habíamos generado varios mensajes.
- Ejecutar el producer y demostrar cómo los mensajes se consumen en tiempo real.

---

### 5. Modificar la configuración en el application.yml para que se consuman los mensajes creados desde el inicio

```yaml
spring:
  application:
    name: kafka-core-consumer
  kafka:
    consumer:
      group-id: kafka-core-consumer-group
      auto-offset-reset: earliest
```

- Los valores posibles para auto-offset-reset son latest | earliest
- El valor por defecto es latest