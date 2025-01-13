### 1. Crear el topic t-hola-mundo

```bash
docker compose -f docker-compose-core-yml up -d
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-hola-mundo --partitions 1
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic t-hola-mundo
```

---

### 2. Crear el producer HolaMundoKafka

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class HolaMundoKafkaProducer {

	@Autowired
	private KafkaTemplate<String, String> kafkaTemplate;

	public void sendHolaMundo(String name) {
		kafkaTemplate.send("t-hola-mundo", "Hola " + name);
	}
}
```

---

### 3. Modificar la clase principal de kafka-core-producer para ejecutar el producer

```java
import com.imagina.kafka.producer.HolaMundoKafkaProducer;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import java.util.concurrent.ThreadLocalRandom;

@SpringBootApplication
public class KafkaCoreProducerApplication implements CommandLineRunner {

	private final HolaMundoKafkaProducer holaMundoKafkaProducer;

	public KafkaCoreProducerApplication(HolaMundoKafkaProducer holaMundoKafkaProducer) {
		this.holaMundoKafkaProducer = holaMundoKafkaProducer;
	}

	public static void main(String[] args) {
		SpringApplication.run(KafkaCoreProducerApplication.class, args);
	}

	@Override
	public void run(String... args) throws Exception {
        holaMundoKafkaProducer.sendHolaMundo("David " + ThreadLocalRandom.current().nextInt(1000));
	}
}
```

- Ejecutar KafkaCoreproducer

---

### 4. Crear la clase HolaMundoKafkaConsumer

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class HelloKafkaConsumer {

    @KafkaListener(topics = "t-hola-mundo")
    public void consume(String message) {
        System.out.println(message);
    }
}
```

- Ejecutar KafkaCoreConsumer
- Recibiremos un error indicando que debemos definir el group id.

---

### 5. Añadir el group id a nivel de aplicacion en el application.yml de kafka-core-consumer

```yaml
spring:
  application:
    name: kafka-core-consumer
  kafka:
    consumer:
      group-id: kafka-core-consumer-group
```

- Ejecutar el consumer y después ejecutar el producer.
- Mostar cómo el consumer muestra el mensaje.
- Indicar que a pesar de haber generado 2 mensajes, el consumer sólo ha consumido 1 de ellos. 