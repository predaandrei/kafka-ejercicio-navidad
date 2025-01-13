## Formato de los mensajes
- Kafka acepta cualquier cadena de texto como mensaje
- Pero ¿qué pasa con los formatos custom?
- Imagina que se producen mensajes en el productor 1 en string que utilizan `|` como delimitador
- Luego tenemos un productor 2 que utiliza `;` como delimitador
- Si tenemos una aplicación que consume mensajes de ambos productores necesitará conocer esas dos reglas
- Si en un futuro hay más productores con distintas reglas esto se puede hacer inmanejable

## Usando JSON como formato
- Podemos usar JSON como formato standard
- Java y otros lenguajes tienen librerías para crear y parsear JSON
- De este modo el formato es estándar y las aplicaciones se pueden centrar en la lógica de negocio en lugar de en la lógica para parsear los mensajes
- Java usa Jackson o GSON como librerías

### 1. Añadir las dependencias Jackson al proyecto Maven del producer y del consumer

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
```
### 2. Crear el topic t-empleado

```bash
docker compose -f docker-compose-core-yml up -d
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-empleado --partitions 1
```

### 3. Crear el paquete model en el kafka-core-producer. Crear la clase empleado

```java
import java.time.LocalDate;

public class Employee {

    private String employeeId;
    private String name;
    private LocalDate birthDate;

    public Employee() {

    }

    public Employee(String employeeId, String name, LocalDate birthDate) {
        super();
        this.employeeId = employeeId;
        this.name = name;
        this.birthDate = birthDate;
    }

    public LocalDate getBirthDate() {
        return birthDate;
    }

    public String getEmployeeId() {
        return employeeId;
    }

    public String getName() {
        return name;
    }

    public void setBirthDate(LocalDate birthDate) {
        this.birthDate = birthDate;
    }

    public void setEmployeeId(String employeeId) {
        this.employeeId = employeeId;
    }

    public void setName(String name) {
        this.name = name;
    }

}
```

### 4. Crear el paquete config. Crear la clase JsonConfig

```java
import org.springframework.context.annotation.Configuration;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;

@Configuration
public class JsonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        objectMapper.disable(SerializationFeature.WRITE_DATE_AS_TIMESTAMPS);
        return objectMapper;
    }
}
```

### 5. Crear la clase EmpleadoProducer

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import com.imagina.kafka.model.Empleado;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
public class EmpleadoProducer {

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(Empleado empleado) throws JsonProcessingException {
        var json = objectMapper.writeValueAsString(empleado);
        kafkaTemplate.send("t-empleado", json);
    }

}

```

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

### 6. Modificar la clase principal para generar 5 empleados y enviarlos a Kafka

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import java.time.LocalDate;
import java.util.UUID;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

@SpringBootApplication
public class KafkaCoreProducerApplication implements CommandLineRunner {

    private final EmpleadoProducer empleadoProducer;

    public KafkaCoreProducerApplication(EmpleadoProducer empleadoProducer) {
        this.empleadoProducer = empleadoProducer;
    }

    public static void main(String[] args) {
        SpringApplication.run(KafkaCoreProducerApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        for (int i = 0; i < 5; i++) {
            Empleado empleado = new Empleado(UUID.randomUUID().toString(), "Empleado" + i, LocalDate.now().minusYears(20 + i));
            empleadoProducer.sendMessage(empleado);
        }
    }
}
```
- Antes de ejecutar el consumer añadir la propiedad date-format al application.yml

```yaml
spring:
  jackson:
    date-format: dd-MM-yyyy
```

- Ejecutar el productor
- Comprobar el topic usando la consolaç

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-empleado --offset earliest --partition 0
```

### 7. Copiar y pegar la clase de configuración y la clase de Empleado en el proyecto del consumer
- Modificar la clase de Empleado en el consumer para generar el método toString
- Copiar y pegar el jackson date-format al application.yml

### 8. Crear la clase EmpleadoConsumer

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;

import com.course.kafka.entity.Employee;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
@Slf4j
public class EmpleadoConsumer {
	
	@Autowired
	private ObjectMapper objectMapper;
	
	@KafkaListener(topics = "t-empleado")
	public void listen(String message) throws JsonMappingException, JsonProcessingException {
		var empleado = objectMapper.readValue(message, Empleado.class);
		log.info("El empleado es : {}", empleado);
	}
	
}

```

- Ejecutar el Consumer
- Ejecutar el Producer
- Comprobar que el consumer está consumiendo datos del producer