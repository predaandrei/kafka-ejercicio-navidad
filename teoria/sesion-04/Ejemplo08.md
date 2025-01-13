## Filtrado de Mensajes
- Se pueden filtrar los mensajes de forma que sólo son procesados si cumplen con el criterio
- Los mensajes que no cumplan con el criterio no serán procesados pero permanecerán en el topic
- Podemos aplicar un filtro a cada listener de este modo, un listener procesará todos los mensajes y otro sólo los que cumplan ciertos criterios
- En el siguiente ejemplo tendremos un servicio que recibe la posición de aviones constantemente
  - Por un lado, tendremos un consumer que procesa todas las posiciones
  - Por otro lado tendremos un consumer que sólo procesa las posiciones de los aviones que estén a una cierta distancia

---

## 1. Crea un topic t-posicion

```bash
docker compose -f docker-compose-core-yml
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-posicion --partitions 1
```

---

## 2. Crea la clase PlanePosition dentro del paquete model

```java
public class PlainPosition {

	private String planeId;
	private long timestamp;
	private int distancia;

	public PlainPosition() {

	}

	public PlainPosition(String planeId, long timestamp, int distancia) {
		super();
		this.planeId = planeId;
		this.timestamp = timestamp;
		this.distancia = distancia;
	}
}
```

---

## 3. Crea la clase PlanePositionProducer

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import com.imagina.kafka.model.PlainPosition;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
public class PlanePositionProducer {

	@Autowired
	private KafkaTemplate<String, String> kafkaTemplate;
	
	@Autowired
	private ObjectMapper objectMapper;
	
	public void send(PlainPosition planePosition) throws JsonProcessingException {
		var json = objectMapper.writeValueAsString(planePosition);
		kafkaTemplate.send("t-posicion", json);
	}
	
}
```

---

## 4. Crea la clase PlanePositionScheduler

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;

import com.imagina.kafka.model.PlanePosition;
import com.imagina.kafka.producer.PlanePositionProducer;
import com.fasterxml.jackson.core.JsonProcessingException;

@Service
@Slf4j
public class PlanePositionScheduler {

	private PlanePosition planeOne;
	private PlanePosition planeTwo;
	private PlanePosition planeThree;

	@Autowired
	private PlanePositionProducer producer;

	public PlanePositionScheduler() {
		var now = System.currentTimeMillis();

        planeOne = new PlanePosition("plane-1", now, 0);
        planeTwo = new PlanePosition("plane-2", now, 110);
        planeThree = new PlanePosition("plane-3", now, 95);
	}

	@Scheduled(fixedRate = 10000)
	public void generatePlanePosition() throws JsonProcessingException {
		var now = System.currentTimeMillis();

        planeOne.setTimestamp(now);
        planeTwo.setTimestamp(now);
        planeThree.setTimestamp(now);

        planeOne.setDistance(planeOne.getDistance() + 1);
        planeTwo.setDistance(planeTwo.getDistance() - 1);
        planeThree.setDistance(planeThree.getDistance() + 1);
		
		producer.send(planeOne);
		producer.send(planeTwo);
		producer.send(planeThree);
		
		LOG.info("Sent : {}", planeOne);
		LOG.info("Sent : {}", planeTwo);
		LOG.info("Sent : {}", planeThree);
	}

}

```

---

## 5. Crea la clase PlanePositionConsumer en el proyecto consumer (Copia la clase PlanePosition del proyecto producer)

- Indicar que una forma de filtrar el mensaje sería simplemente añadir un if dentro del método que queremos filtrar (listenFar)
- Pero que lo que vamos a hacer es usar una containerFactory para evitar que los mensajes que no cumplan los criterios ni siquiera sean escuchados

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;

import com.imagina.kafka.model.PlanePosition;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
@Slf4j
public class PlanePositionConsumer {
	
	@Autowired
	private ObjectMapper objectMapper;
	
	@KafkaListener(topics = "t-posicion", groupId = "consumer-group-all-position")
	public void listenAll(String message) throws JsonMappingException, JsonProcessingException {
		var planePosition = objectMapper.readValue(message, PlanePosition.class);
		log.info("listenAll : {}", planePosition);
	}
	

	@KafkaListener(topics = "t-posicion", groupId = "consumer-group-near-position",containerFactory = "nearPositionContainerFactory")
	public void listenFar(String message) throws JsonMappingException, JsonProcessingException {
		var planePosition = objectMapper.readValue(message, PlanePosition.class);		
		LOG.info("listenNear: {}", planePosition);		
	}
}

```

## 6. Crea el bean NearPositionContainerFactory dentro de la clase de configuración de Kafka

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.kafka.ConcurrentKafkaListenerContainerFactoryConfigurer;
import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.kafka.listener.adapter.RecordFilterStrategy;
import org.springframework.util.backoff.FixedBackOff;

import com.course.kafka.entity.CarLocation;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Configuration
public class KafkaConfig {

	@Autowired
	private KafkaProperties kafkaProperties;

	@Autowired
	private ObjectMapper objectMapper;

	@Bean
	public ConsumerFactory<Object, Object> consumerFactory() {
		var properties = kafkaProperties.buildConsumerProperties();

		properties.put(ConsumerConfig.METADATA_MAX_AGE_CONFIG, "120000");

		return new DefaultKafkaConsumerFactory<>(properties);
	}

	@Bean(name = "nearPositionContainerFactory")
	public ConcurrentKafkaListenerContainerFactory<Object, Object> nearPositionContainerFactory(
			ConcurrentKafkaListenerContainerFactoryConfigurer configurer) {
		var factory = new ConcurrentKafkaListenerContainerFactory<Object, Object>();
		configurer.configure(factory, consumerFactory());

		factory.setRecordFilterStrategy(new RecordFilterStrategy<Object, Object>() {

			@Override
			public boolean filter(ConsumerRecord<Object, Object> consumerRecord) {
				try {
					PlanePosition planePosition = objectMapper.readValue(consumerRecord.value().toString(),
                            PlanePosition.class);
					return planePosition.getDistancia() <= 10;
				} catch (JsonProcessingException e) {
					return false;
				}
			}
		});

		return factory;
	}
}
```

- Enable the `@EnableScheduling` annotation in the main class
- Comprueba que listenAll procesa todos los mensajes mientras que listenNear sólo procesa aquellos cuya distancia es menor a 10 kilómetros