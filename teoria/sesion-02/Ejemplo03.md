## Crear mensajes con key
- Los mensajes con la misma key van a la misma partición
- Siempre y cuando no cambiemos el númer de particiones

### 1. Crear el topic t-multi-particiones

```bash
docker compose -f docker-compose-core-yml up -d
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-multi-particiones --partitions 3
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic t-multi-particiones
```
---

### 2. Crear el producer KafkaKeyProducer

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class KafkaKeyProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String key, String message) {
        kafkaTemplate.send("t-multi-particiones", key, message);
    }
}
```
---

### 3. Modificar la clase principal de kafka-core-producer para ejecutar el producer

```java
import com.imagina.kafka.producer.HelloKafkaProducer;
import com.imagina.kafka.producer.KafkaKeyProducer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

@SpringBootApplication
public class KafkaCoreProducerApplication implements CommandLineRunner {

    private final KafkaKeyProducer kafkaKeyProducer;

    public KafkaCoreProducerApplication(KafkaKeyProducer kafkaKeyProducer) {
        this.kafkaKeyProducer = kafkaKeyProducer;
    }

    public static void main(String[] args) {
        SpringApplication.run(KafkaCoreProducerApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        for(int i = 1; i <= 30; i++) {
            String key = "key-" + i;
            String message = "Message " + i;
            kafkaKeyProducer.sendMessage(key, message);

            //TimeUnit.SECONDS.sleep(1);
        }
    }

}

```

- Ejecutar KafkaCoreproducer

---

### 4. Usar el kafka-console-consumer para comprobar que los datos han ido a las distintas particiones

- Abrir tres ventanas en la terminal y consumir de distintas particiones en cada una de ellas.

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-multi-particiones --offset earliest --partition 0
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-multi-particiones --offset earliest --partition 1
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-multi-particiones --offset earliest --partition 2
```
---

### 5. Modificar el método para generar mensajes y hacer que vaya hasta 10000 y tenga un sleep de 1 seg

```java
    @Override
    public void run(String... args) throws Exception {
        for(int i = 1; i <= 10000; i++) {
            String key = "key-" + i;
            String message = "Message " + i;
            kafkaKeyProducer.sendMessage(key, message);

            TimeUnit.SECONDS.sleep(1);
        }
    }
```

- Ejecutar el producer de nuevo.
- Inspeccionar el kafka console consumer.
- Podemos ver que los mensajes con la misma key siempre van a la misma partición.

---

### 6. Crear la clase KafkaKeyConsumer

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class KafkaKeyConsumer {

    @KafkaListener(topics = "t-multi-particiones")
    public void consume(ConsumerRecord<String, String> record) throws InterruptedException {
        log.info("Key : {}, Partition: {}, Message: {}", record.key(), record.partition(), record.value());
        TimeUnit.SECONDS.sleep(1);
    }
}
```

- Añadir el thread al log de spring

```yaml
logging:
  pattern:
    console: "[Kafka Core Consumer] %clr(%d{HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:%5p}) %clr(---){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} [%t] %m%n${LOG_EXCEPTION_CONVERSION_WORD:%wEx}"

```

- Ejecutar KafkaCoreConsumer
- Al tener un sólo consumer para las tres particiones podemos ver cómo el mismo está consumiendo de las tres particiones.
- Podemos ver cómo cada mensaje tarda un segundo en ser procesado (que es la latencia que hemos añadido al consumer).
- Detener el consumer, vamos a mejorar esto añadiendo más consumers.

---

### 7. Modificar el número de consumers

```java
@KafkaListener(topics = "t-multi-particiones", concurrency = "2")
```

- En el productor, modificar el comando sleep para que mande los mensajes no cada segundo sino instantáneamente.
- Reiniciar el productor e iniciar el consumidor.
- Ahora si vemos los logs podemos ver que 1 consumer está asignado a dos particiones y el otro a 1 partición.
- Ahora podemos consumir dos mensajes cada segundo.
- Aún así, lo ideal sería tener 3 consumidores, 1 para cada partición.

---

### 8. Modificar el número de consumers de nuevo

```java
@KafkaListener(topics = "t-multi-particiones", concurrency = "3")
```

- Ahora si vemos los logs podemos ver que 1 consumer está asignado a cada partición.
- Ahora podemos consumir tres mensajes cada segundo.

#### ¿Qué ocurriría si establecemos el valor de concurrency a 4?
- En ese caso, un consumer estaria parado y sin asignar a ninguna partición.
- Lo ideal es tener un consumer por partición.

---

### 9. Añadir una partición nueva al topic

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --alter --t-multi-particiones --partitions 4
kafka-topics.sh --bootstrap-server localhost:9092 --describe --t-multi-particiones
```
- Si ahora iniciamos de nuevo el producer y el consumer veremos que los 4 consumers están funcionando.

<b>IMPORTANTE</b>
- Se puede borrar un topic
- No se puede borrar una partición
- Borrar una partición puede provocar perdida de datos
- Borrar una partición puede provocar una incorrecta distribución de las keys
- La única forma de reducir el número de particiones es borrar el topic y recrearlo con el número deseado de particiones
- Pero cuidado porque borrar un topic significa borrar todos los datos de ese topic
