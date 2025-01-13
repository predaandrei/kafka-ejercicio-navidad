## At-Least-Once Delivery Semantic
- Se garantiza que cada mensaje se publica
- Se puede dar el caso de que un mensaje se publique más de una vez por ejemplo si hay un fallo de red y no se ha confirmado la recepción del mensaje, el publisher puede volver a publicarlo
- El consumer consume todo los mensajes, incluyendo los duplicados
- Para gestionar esto podemos hacer que los mensajes sean idempotentes
  - Si tener mensajes duplicados no tiene efecto
    - El resultado de procesar un mensaje es siempre el mismo incluso para mensajes duplicados
    - Ejemplo: actualizar un índice en un motor de búsqueda
  - Si tener mensajes duplicados puede ser peligroso
    - Se pueden dar transacciones duplicadas
    - Ejemplo: un mensaje para crear un pago
    - En este caso, debemos filtrar los mensajes duplicados

---

## ¿Cómo evitar duplicados?
- Crear un valor único asociado a cada mensaje
- El consumer chequea este valor único cuando recibe el mensaje
  - Si no se ha procesado --> se almacena el valor único y después se procesa el mensaje
  - Si se ha procesado --> el mensaje no se procesa
- Podemos usar bases de datos para almacenar los valores únicos
- También se pueden usar cachés para almacenar temporalmente los valores únicos
- Si el volumen de procesado es alto es mejor usar una Cache
  - Mejor rendimiento
  - Si sabemos que los mensajes no se duplicaran después de un tiempo (por ejemplo 2h) borramos los datos de la caché
  - Ejemplo: Redis
- Si los datos pueden estar duplicados durante más tiempo 
  - Usar una base de datos
  - Virtualmente almacenamiento ilimitado en oposición a la cache que suele almacenar los datos en memoria

---

## 1. Crea el topic t-solicitud-compra

```bash
docker compose -f docker-compose-core-yml
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-solicitud-compra --partitions 1
```

---

## 2. En el proyecto producer crea la clase SolicitudCompra

```java
public class SolicitudCompra {

	private UUID solicitudId;
	private String scNumber;
	private int cantidad;
	private String divisa;
    
}
```

---

## 3. En el proyecto producer crea la clase SolicitudCompraProducer

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import com.imagina.kafka.model.SolicitudCompra;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
public class SolicitudCompraProducer {

	@Autowired
	private KafkaTemplate<String, String> kafkaTemplate;

	@Autowired
	private ObjectMapper objectMapper;

	public void send(SolicitudCompra solicitudCompra) throws JsonProcessingException {
		var json = objectMapper.writeValueAsString(solicitudCompra);
		kafkaTemplate.send("t-solicitud-compra", solicitudCompra.getScNumber(), json);
	}

}

```

---

## 4. Crear Solicitudes de compra en la clase principal

- Simulamos que algo ha ido mal y que se ha publicado el mensaje para la solicitud1 dos veces

```java
import java.util.UUID;

@Autowired
private SolicitudCompraProducer solicitudCompraProducer;

@Override
public void run(String... args) throws Exception {
  SolicitudCompra solicitud1 = new SolicitudCompra(UUID.randomUUID(), "SOL-001", 100, "EUR");
  SolicitudCompra solicitud2 = new SolicitudCompra(UUID.randomUUID(), "SOL-002", 200, "EUR");
  SolicitudCompra solicitud3 = new SolicitudCompra(UUID.randomUUID(), "SOL-003", 300, "USD");
  
  solicitudCompraProducer.send(solicitud1);
  solicitudCompraProducer.send(solicitud2);
  solicitudCompraProducer.send(solicitud3);

  solicitudCompraProducer.send(solicitud1);
  
}
```

---

## 5. Añade las dependencias para la cache caffeine en el pom del consumer

```xml
<!-- https://mvnrepository.com/artifact/com.github.ben-manes.caffeine/caffeine -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
```
Crear la clase de configuración para la cache en el proyecto consumer

```java
import java.time.Duration;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

@Configuration
public class CacheConfig {

	@Bean(name = "cacheSolicitudCompra")
	public Cache<Integer, Boolean> cacheSolicitudCompra() {
		return Caffeine.newBuilder().expireAfterWrite(Duration.ofMinutes(2)).maximumSize(1000).build();
	}

}

```

Copiar y pegar la clase `SolicitudCompra` en el proyecto consumer

---

## 6. Crea la clase SolicitudCompraConsumer en el proyecto consumer

```java
import java.util.Optional;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.kafka.annotation.KafkaListener;

import com.imagina.kafka.model.SolicitudCompra;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.benmanes.caffeine.cache.Cache;

@Service
@Slf4j
public class SolicitudCompraConsumer {
	
	@Autowired
	private ObjectMapper objectMapper;
	
	@Autowired
	@Qualifier("cacheSolicitudCompra")
	private Cache<Integer, Boolean> cache;
	
	private boolean isInCache(int solicitudCompraId) {
		return Optional.ofNullable(cache.getIfPresent(solicitudCompraId)).orElse(false);
	}
	
	@KafkaListener(topics = "t-solicitud-compra")
	public void consume(String message) throws JsonMappingException, JsonProcessingException {
		var solicitudCompra = objectMapper.readValue(message, SolicitudCompra.class);
		
		var processed = isInCache(solicitudCompra.getId());
		
		if (processed) {
			return;
		}
		
		log.info("Procesando solicitud de compra {}", solicitudCompra);
		
		cache.put(solicitudCompra.getId(), true);
	}
	
}
```
- Ejecuta el consumer y el producer
- Si reiniciamos el producer sin que hayan pasado los 5 minutos que tenemos configurados en la cache, los mensajes no serán procesados
- Sin embargo, si comprobamos los mensajes que hay en el topic, veremos que hay 8
