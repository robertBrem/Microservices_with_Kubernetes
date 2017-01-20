# The command REST service

We're going to change the existing REST service to be the command service.

We delete the `InMemoryCache` class because this class was only responsible 
for querying.

Delete the `getUsers` method in the `UserResource` class.  
Delete the `getUsers` and the `getUsersFilteredByNickname` methods in the
`UserService` class.

## Add Kafka
We use Kafka for pub sub messaging. Therefore we need the following dependency:

```
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.10.0.0</version>
</dependency>
```

With the clients library we're able to implement a provide for the Kafka
consumer and producer:

```
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;

import javax.annotation.PostConstruct;
import javax.ejb.ConcurrencyManagement;
import javax.ejb.ConcurrencyManagementType;
import javax.ejb.Singleton;
import javax.ejb.Startup;
import javax.enterprise.inject.Produces;
import java.util.Arrays;
import java.util.Properties;
import java.util.UUID;

@Startup
@Singleton
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
public class KafkaProvider {

    public static final String KAFKA_ADDRESS = System.getenv("KAFKA_ADDRESS");

    public static final String TOPIC = "battleapp";
    public static final String GROUP_ID = "battleapp";

    private KafkaProducer<String, String> producer;
    private KafkaConsumer<String, String> consumer;

    @PostConstruct
    public void init() {
        this.producer = createProducer();
        this.consumer = createConsumer();
    }

    @Produces
    public KafkaProducer<String, String> getProducer() {
        return producer;
    }

    @Produces
    public KafkaConsumer<String, String> getConsumer() {
        return consumer;
    }

    public KafkaProducer<String, String> createProducer() {
        Properties properties = new Properties();
        properties.put("bootstrap.servers", KAFKA_ADDRESS);
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        return new KafkaProducer<>(properties);
    }

    public KafkaConsumer<String, String> createConsumer() {
        Properties properties = new Properties();
        properties.put("bootstrap.servers", KAFKA_ADDRESS);
        properties.put("group.id", GROUP_ID + UUID.randomUUID().toString());
        properties.put("auto.offset.reset", "earliest");
        properties.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties);
        consumer.subscribe(Arrays.asList(TOPIC));
        return consumer;
    }

}
```

And a worker thread that handles the initialization events of the query
services:

```
import com.airhacks.porcupine.execution.boundary.Dedicated;
import ninja.disruptor.battleapp.eventstore.control.EventStore;
import ninja.disruptor.battleapp.eventstore.control.JsonConverter;
import ninja.disruptor.battleapp.user.entity.User;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;

import javax.annotation.PostConstruct;
import javax.ejb.ConcurrencyManagement;
import javax.ejb.ConcurrencyManagementType;
import javax.ejb.Singleton;
import javax.ejb.Startup;
import javax.inject.Inject;
import javax.json.Json;
import javax.json.JsonObject;
import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.nio.charset.Charset;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;

@Startup
@Singleton
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
public class KafkaConsumerWorker {

    public static final Charset CHARSET = Charset.forName("UTF-8");
    public static final String TOPIC_NAME = "topicName";

    @Dedicated
    @Inject
    ExecutorService kafka;

    @Inject
    JsonConverter converter;

    @Inject
    EventStore store;

    @Inject
    KafkaProducer<String, String> producer;

    @Inject
    KafkaConsumer<String, String> consumer;

    @PostConstruct
    public void init() {
        CompletableFuture
                .runAsync(this::handleKafkaEvent, kafka);
    }

    public void handleKafkaEvent() {
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(200);
            for (ConsumerRecord<String, String> record : records) {
                switch (record.topic()) {
                    case KafkaProvider.TOPIC:
                        try {
                            handleInitializationEvent(record);
                        } catch (Exception e) {
                            System.out.println("ERROR: " + e.getMessage());
                        }
                        break;
                    default:
                        System.out.println("ERROR: Illegal topic: " + record.topic());
                }
            }
        }
    }

    private void handleInitializationEvent(ConsumerRecord<String, String> record) {
        String jsonAsString = record.value();
        InputStream inputStream = new ByteArrayInputStream(jsonAsString.getBytes(CHARSET));
        JsonObject event = Json.createReader(inputStream).readObject();
        if (event == null) {
            return;
        }
        String topicName = event.getString(TOPIC_NAME);
        if (topicName == null) {
            return;
        }
        String eventsAsJsonString = converter
                .convertToJson(store
                        .loadEventStream(User.class.getName())
                        .getEvents())
                .toString();

        producer.send(new ProducerRecord<>(
                topicName,
                eventsAsJsonString));
    }
}
```

In the `EventStore` we've to replace the JavaEE events with Kafka events.  
Replace the `Event<String> bus` with the `KafkaProducer<String, String> producer`:

```
...
@Inject
Event<String> bus;
...
String eventsAsJsonString = converter
                                .convertToJson(events)
                                .toString();
bus.fire(eventsAsJsonString);
...
```
```
...
@Inject
KafkaProducer<String, String> producer;
...
String eventsAsJsonString = converter
                                .convertToJson(events)
                                .toString();
producer.send(new ProducerRecord<>(
                    KafkaProvider.TOPIC,
                    eventsAsJsonString));
...
```

