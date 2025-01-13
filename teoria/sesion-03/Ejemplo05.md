## Consumer groups
- Vamos a crear un consumer con consumer group
- El producer publicará acciones
- Hay dos funcionalidades
  - Actualizar una UI
  - Enviar notificaciones
- Se leerá el mismo mensaje pero para funcionalidades distintas

## 1. Crear el topic t-stocks con 1 partición

```bash
docker compose -f docker-compose-core-yml up -d
docker exec -it kafka bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic t-stocks --partitions 1
```

### 2. Crear la clase Stock.java dentro del paquete modelo del producer

- Simbolo (APPL, Meta, Amazon...)
- Nombre de la compañia 
- Precio Actual (double)
- Precio al cierre anterior (double)
- volumen (int)
- timestamp (long)

- CREAR TO STRING Y USAR LOMBOCK PARA GETTERS, SETTERS Y CONSTRUCTOR. EN EL CONSTRUCTOR USAR SETPRICE METHOD
- EN EL MÉTODO SETPRICE Y SETCLOSEPRICE USAR 2 DÍGITOS

```java
public class Stock {

    private String simbolo;
    private String companyName;
    private double price;
    private long timestamp;
    
    public void setPrice(double price) {
        this.price = Math.round(price * 100.0) / 100.0;
    }
}
```
### 3. Crear el servicio StockService

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ThreadLocalRandom;

import org.springframework.stereotype.Service;


@Service
public class StockService {

	private static final Map<String, Stock> STOCK_MAP = new HashMap<>();
	private static final String APPLE = "APPL";
	private static final String META = "META";
	private static final double MAX_ADJUSTMENT = 250.05d;
	private static final double MIN_ADJUSTMENT = 100.95d;
		
	static {
		var timestamp = System.currentTimeMillis();

        STOCK_MAP.put(APPLE, new Stock(APPLE, "Apple", 4_834.57, timestamp));
        STOCK_MAP.put(META, new Stock(META, "Meta", 1_185.29, timestamp));
	}
	
	public List<Stock> createTestStocks() {
		var result = new ArrayList<Commodity>();
        STOCK_MAP.keySet().forEach(c -> result.add(createTestStock(c)));
		
		return result;
	}
	
	public Commodity createTestStock(String name) {
		if (!STOCK_MAP.keySet().contains(name)) {
			throw new IllegalArgumentException("No existe esta acción : " + name);
		}
		
		var stock = STOCK_MAP.get(name);
		var price = stock.getPrice();
		var newPrice = price * ThreadLocalRandom.current().nextDouble(MIN_ADJUSTMENT, MAX_ADJUSTMENT);

        stock.setPrice(newPrice);
        stock.setTimestamp(System.currentTimeMillis());
	
		return stock;
	}
	
}
```

### 4. Añadir la dependencia web de spring boot para crear una api que devuelva las acciones

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 5. Crear la clase StockApi dentro del paquete controller en el producer

```java
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.imagina.kafka.model.Stock;
import com.imagina.kafka.service.StockService;

@RestController
@RequestMapping("/api/stock/v1")
public class StockApi {

	@Autowired
	private StockService stockService;
	
	@GetMapping(value = "/all")
	public List<Commodity> generateAllCommodities() {
		return stockService.createTestStocks();
	}
	
}
```

- Levantar la aplicación y probar el endpoint en el navegador
- Comprobar que al refrescar la página da valores distintos

### 6. Crear la clase StockProducer

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import com.imagina.kafka.model.Stock;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
public class StockProducer {

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(Stock stock) throws JsonProcessingException {
        var json = objectMapper.writeValueAsString(stock);
        kafkaTemplate.send("t-stocks", stock.getSimbolo(), json);
    }

}
```

### 7. Crear el scheduler que llama a la api dentro del paquete scheduler

```java
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.web.client.RestTemplate;

import com.imagina.kafka.model.Stock;
import com.imagina.kafka.producer.StockProducer;
import com.fasterxml.jackson.core.JsonProcessingException;

@Service
public class StockScheduler {

	private RestTemplate restTemplate = new RestTemplate();

	@Autowired
	private StockProducer producer;

	@Scheduled(fixedRate = 5000)
	public void retrieveStocks() {
		var stocks = restTemplate.exchange("http://localhost:8080/api/stock/v1/all", HttpMethod.GET, null,
				new ParameterizedTypeReference<List<Stock>>() {
				}).getBody();

        stocks.forEach(t -> {
			try {
				producer.sendMessage(t);
			} catch (JsonProcessingException e) {
				e.printStackTrace();
			}
		});
	}
}
```

- Añadir la anotación EnableScheduling a la clase principal

```java
@EnableScheduling
```

- Usar la consola de kafka para mostrar los mensajes enviados

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic t-stocks --offset earliest --partition 0
```

### 8. Copiar la clase stocks al proyecto consumer

### 9. Crear la clase StockUIConsumer dentro de la carpeta consumers

```java
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;

import com.imagina.kafka.model.Stock;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
@Slf4j
public class StockUIConsumer {
	
	@Autowired
	private ObjectMapper objectMapper;
	
	@KafkaListener(topics = "t-stocks", groupId = "consumer-group-ui")
	public void consume(String message) throws JsonMappingException, JsonProcessingException, InterruptedException {
		var stock = objectMapper.readValue(message, Stock.class);		
		
		// var randomDelayMillis = ThreadLocalRandom.current().nextLong(500, 2000);
		// TimeUnit.MILLISECONDS.sleep(randomDelayMillis);
		
		log.info("Procesado de la UI para : {}", stock);
	}
}
```

- Si definimos el groupId en la anotación KafkaListener sobreescribe el group id de la aplicación que tenemos configurado en el application.yml

### 10. Crear la clase StockNotificationConsumer dentro de la carpeta consumers

```java
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;

import com.imagina.kafka.model.Stock;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
@Slf4j
public class StockNotificationConsumer {
	
	@Autowired
	private ObjectMapper objectMapper;
	
	@KafkaListener(topics = "t-stocks", groupId = "consumer-group-notification")
	public void consume(String message) throws JsonMappingException, JsonProcessingException, InterruptedException {
		var stock = objectMapper.readValue(message, Stock.class);
		
		log.info("Procesado de las Notificaciones para : {}", stock);
	}
}
```

- Iniciar el productor y el consumidor y esperar un tiempo
- Revisar los logs
- Simular un funcionamiento lento del consumidor de la UI descomentando las líneas comentadas
- Comprobar que el consumer de notificaciones sigue consumiendo los mensajes sin verse afectado por el consumer de la UI

