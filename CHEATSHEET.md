# Apache Kafka Command-Line Cheat Sheet

Este documento contiene una referencia rápida para ejecutar los comandos más comunes en Apache Kafka utilizando las herramientas de línea de comandos incluidas.

## Tabla de Contenidos

1. [Gestión de Tópicos](#gestión-de-tópicos)
2. [Producción de Mensajes](#producción-de-mensajes)
3. [Consumo de Mensajes](#consumo-de-mensajes)
4. [Grupos de Consumidores](#grupos-de-consumidores)
5. [Verificar Versiones y APIs](#verificar-versiones-y-apis)
6. [Gestión de Configuración](#gestión-de-configuración)
7. [Inspección de Datos](#inspección-de-datos)
8. [Diagnóstico y Monitoreo](#diagnóstico-y-monitoreo)
9. [Administración de Transacciones](#administración-de-transacciones)
10. [Scripts de Utilidades](#scripts-de-utilidades)
11. [Comandos Internos](#comandos-internos)
12. [Depuración de Logs](#depuración-de-logs)

---

## Gestión de Tópicos

### Crear un tópico
```bash
kafka-topics.sh --create --topic <nombre-del-tópico> \
  --bootstrap-server <broker> \
  --partitions <num-particiones> \
  --replication-factor <factor-replicación>
```

### Listar tópicos
```bash
kafka-topics.sh --list --bootstrap-server <broker>
```

### Describir un tópico
```bash
kafka-topics.sh --describe --topic <nombre-del-tópico> --bootstrap-server <broker>
```

### Eliminar un tópico
```bash
kafka-topics.sh --delete --topic <nombre-del-tópico> --bootstrap-server <broker>
```

---

## Producción de Mensajes
### Enviar mensajes a un tópico
```bash
kafka-console-producer.sh --topic <nombre-del-tópico> --bootstrap-server <broker>
```
Escribe mensajes directamente en la consola (uno por línea).

## Consumo de Mensajes
### Leer mensajes de un tópico
```bash
kafka-console-consumer.sh --topic <nombre-del-tópico> \
  --bootstrap-server <broker> \
  --from-beginning
 ```

--from-beginning: Lee todos los mensajes desde el inicio.

---

## Grupos de Consumidores
### Listar grupos de consumidores
```bash
kafka-consumer-groups.sh --list --bootstrap-server <broker>
```

## Describir un grupo de consumidores
```bash
kafka-consumer-groups.sh --describe --group <nombre-del-grupo> --bootstrap-server <broker>
```

### Restablecer el offset de un consumidor
```bash
kafka-consumer-groups.sh --reset-offsets \
  --group <nombre-del-grupo> \
  --topic <nombre-del-tópico> \
  --to-earliest \
  --bootstrap-server <broker> \
  --execute
```

Opciones para --to-*:

- --to-earliest: Restablece al primer mensaje.
- --to-latest: Restablece al último mensaje.
- --shift-by: Mueve los offsets hacia adelante o atrás.

---

## Verificar Versiones y APIs
### Verificar las versiones soportadas por el broker
```bash
kafka-broker-api-versions.sh --bootstrap-server <broker>
```

## Gestión de Configuración
### Ver configuración de un broker
```bash
kafka-configs.sh --bootstrap-server <broker> \
  --entity-type brokers --entity-name <id-del-broker> \
  --describe
```

### Actualizar configuración de un tópico
```bash
kafka-configs.sh --bootstrap-server <broker> \
  --entity-type topics --entity-name <nombre-del-tópico> \
  --alter --add-config <clave>=<valor>
```

#### Ejemplo: Cambiar el período de retención de mensajes.
```bash
--add-config retention.ms=604800000
```

### Eliminar configuración personalizada de un tópico
```bash
kafka-configs.sh --bootstrap-server <broker> \
  --entity-type topics --entity-name <nombre-del-tópico> \
  --alter --delete-config <clave>
```

---

## Inspección de Datos
### Ver metadata del clúster
```bash
kafka-metadata-shell.sh --bootstrap-server <broker>
```

## Diagnóstico y Monitoreo
### Ver estados del consumidor
```bash
kafka-consumer-groups.sh --describe --group <nombre-del-grupo> --bootstrap-server <broker>
```

### Ver réplicas fuera de sincronización
```bash
kafka-topics.sh --describe --bootstrap-server <broker> \
  | grep "UnderReplicated"
```

## Administración de Transacciones
### Inicializar un productor transaccional
```bash
kafka-init-transactions.sh --transactional-id <id-transaccional> --bootstrap-server <broker>
```

### Listar transacciones activas
```bash
kafka-transactions.sh --list --bootstrap-server <broker>
```

---

## Scripts de Utilidades
### Probar latencia del clúster
```bash
kafka-producer-perf-test.sh --topic <nombre-del-tópico> \
  --num-records <número> \
  --record-size <tamaño> \
  --throughput <número> \
  --producer.config <archivo-config>
```

### Probar rendimiento del consumidor
```bash
kafka-consumer-perf-test.sh --bootstrap-server <broker> \
  --topic <nombre-del-tópico> \
  --messages <número>
```

---

## Comandos Internos
### Generar ID único para KRaft
```bash
kafka-storage.sh random-uuid
```

### Formatear almacenamiento para KRaft
```bash
kafka-storage.sh format --config <archivo-config> --cluster-id <id-clúster>
```

## Depuración de Logs
### Ver logs de un broker específico
```bash
docker logs <nombre-contenedor>
```

### Ver logs en tiempo real
```bash
docker logs -f <nombre-contenedor>
```

### Notas
- Cambia <broker> por la dirección del broker, como localhost:9092.
- Todos los scripts (kafka-topics.sh, kafka-console-producer.sh, etc.) están en el directorio bin de Kafka.
- Si usas Docker, ejecuta estos comandos desde dentro de un contenedor Kafka para evitar problemas de compatibilidad.