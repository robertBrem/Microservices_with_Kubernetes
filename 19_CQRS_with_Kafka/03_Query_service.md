# The query REST service

We create a second REST service as the query service.
The service contains the following files:

`pom.xml`
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>ninja.disruptor</groupId>
    <artifactId>battleapp.query</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
    <dependencies>
        <dependency>
            <groupId>javax</groupId>
            <artifactId>javaee-api</artifactId>
            <version>7.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>0.10.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.12</version>
        </dependency>
        <dependency>
            <groupId>com.airhacks</groupId>
            <artifactId>porcupine</artifactId>
            <version>0.0.4</version>
        </dependency>
    </dependencies>
    <build>
        <finalName>battleapp</finalName>
    </build>
    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <failOnMissingWebXml>false</failOnMissingWebXml>
    </properties>
</project>
```

`Dockerfile`
```
FROM jboss/keycloak-adapter-wildfly:2.4.0.Final

MAINTAINER Robert Brem <brem_robert@hotmail.com>

ENV DEPLOYMENT_DIR ${JBOSS_HOME}/standalone/deployments/

ADD target/battleapp.war ${DEPLOYMENT_DIR}
```

`build.js`
```
#!/usr/bin/jjs -fv

var version = $ENV.VERSION;
var username = $ENV.REGISTRY_USERNAME;
var password = $ENV.REGISTRY_PASSWORD;
var email = $ENV.REGISTRY_EMAIL;

var registry = "disruptor.ninja:30500";
var image = "robertbrem/battleapp-query";
var completeImageName = registry + "/" + image + ":" + version;

var dockerBuild = "docker build -t " + completeImageName + " .";
execute(dockerBuild);

var dockerLogin = "docker login --username=" + username + " --password=" + password + " --email=" + email + " " + registry;
execute(dockerLogin);

var push = "docker push " + completeImageName;
execute(push);

function execute(command) {
    $EXEC(command);
    print($OUT);
    print($ERR);
}
```

`beans.xml`
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/beans_1_1.xsd"
       bean-discovery-mode="all">
</beans>
```

`keycloak.json`
```
{
  "realm": "${env.REALM_NAME}",
  "bearer-only": true,
  "auth-server-url": "${env.AUTH_SERVER_URL}",
  "ssl-required": "none",
  "resource": "battleapp-query",
  "enable-cors": true
}
```

`web.xml`
```
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">

    <security-constraint>
        <web-resource-collection>
            <web-resource-name>health</web-resource-name>
            <url-pattern>/resources/health</url-pattern>
        </web-resource-collection>
        <!-- OMIT auth-constraint -->
    </security-constraint>

    <security-constraint>
        <web-resource-collection>
            <web-resource-name>cors</web-resource-name>
            <url-pattern>/*</url-pattern>
            <http-method>GET</http-method>
            <http-method>POST</http-method>
            <http-method>PUT</http-method>
            <http-method>DELETE</http-method>
        </web-resource-collection>
        <auth-constraint>
            <role-name>user</role-name>
        </auth-constraint>
    </security-constraint>

    <login-config>
        <auth-method>KEYCLOAK</auth-method>
        <realm-name>this is ignored currently</realm-name>
    </login-config>

    <security-role>
        <role-name>admin</role-name>
    </security-role>
    <security-role>
        <role-name>user</role-name>
    </security-role>
</web-app>
```

Copy all the events under `ninja.disruptor.battleapp.user.entity.event` and
the `CoreEvent`.
> All the events have to be in the same package as the events from the
> command side.

Create the following classes:

```
package ninja.disruptor.battleapp.health.boundary;

import com.airhacks.porcupine.execution.boundary.Dedicated;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.container.AsyncResponse;
import javax.ws.rs.container.Suspended;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;

@Path("health")
public class HealthResource {

    @Dedicated
    @Inject
    ExecutorService healthPool;

    @GET
    public void getHealth(@Suspended AsyncResponse response) {
        CompletableFuture
                .supplyAsync(() -> "everything OK!")
                .thenAccept(response::resume);
    }

}
```

```
package ninja.disruptor.battleapp.kafka.control;

import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;

import javax.annotation.PostConstruct;
import javax.ejb.ConcurrencyManagement;
import javax.ejb.ConcurrencyManagementType;
import javax.ejb.Singleton;
import javax.enterprise.inject.Produces;
import java.util.Arrays;
import java.util.Properties;
import java.util.UUID;

@Singleton
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
public class KafkaProvider {
    public static final String TOPIC = "battleapp";
    public static final String KAFKA_ADDRESS = System.getenv("KAFKA_ADDRESS");
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

```
package ninja.disruptor.battleapp.user.boundary;

import com.airhacks.porcupine.execution.boundary.Dedicated;

import javax.inject.Inject;
import javax.ws.rs.*;
import javax.ws.rs.container.AsyncResponse;
import javax.ws.rs.container.Suspended;
import javax.ws.rs.core.MediaType;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;

@Path("users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    @Dedicated
    @Inject
    ExecutorService usersPool;

    @Inject
    UserService service;

    @GET
    public void getUsers(@Suspended AsyncResponse response, @QueryParam("nickname") String nickname) {
        CompletableFuture
                .supplyAsync(service.getUsersFilteredByNickname(nickname), usersPool)
                .thenAccept(response::resume);
    }

}
```

```
package ninja.disruptor.battleapp.user.boundary;

import ninja.disruptor.battleapp.InMemoryCache;
import ninja.disruptor.battleapp.user.entity.User;

import javax.ejb.Stateless;
import javax.inject.Inject;
import javax.ws.rs.core.GenericEntity;
import java.util.HashSet;
import java.util.Set;
import java.util.function.Supplier;
import java.util.stream.Collectors;

@Stateless
public class UserService {

    @Inject
    InMemoryCache cache;

    public Supplier<GenericEntity<Set<User>>> getUsersFilteredByNickname(String nickname) {
        if (nickname == null || nickname.isEmpty()) {
            return () -> new GenericEntity<Set<User>>(getUsers()) {
            };
        } else {
            return () -> new GenericEntity<Set<User>>(getUsers()
                    .parallelStream()
                    .filter(u -> u.getNickname().toLowerCase().contains(nickname.toLowerCase()))
                    .collect(Collectors.toSet())) {
            };
        }
    }

    public Set<User> getUsers() {
        return new HashSet<>(cache.getUsers().values());
    }

}
```

```
package ninja.disruptor.battleapp.user.entity;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import ninja.disruptor.battleapp.eventstore.entity.CoreEvent;
import ninja.disruptor.battleapp.user.entity.event.UserCreated;
import ninja.disruptor.battleapp.user.entity.event.UserFirstNameChanged;
import ninja.disruptor.battleapp.user.entity.event.UserLastNameChanged;
import ninja.disruptor.battleapp.user.entity.event.UserNicknameChanged;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import java.util.List;

@NoArgsConstructor
@ToString
@XmlAccessorType(XmlAccessType.FIELD)
@Data
public class User {
    private String id;
    private String nickname;
    private String firstName;
    private String lastName;

    public User(List<CoreEvent> events) {
        for (CoreEvent event : events) {
            mutate(event);
        }
    }

    public void mutate(CoreEvent event) {
        when(event);
    }

    public void when(CoreEvent event) {
        if (event instanceof UserCreated) {
            this.id = event.getId();
        } else if (event instanceof UserFirstNameChanged) {
            this.firstName = ((UserFirstNameChanged) event).getFirstName();
        } else if (event instanceof UserLastNameChanged) {
            this.lastName = ((UserLastNameChanged) event).getLastName();
        } else if (event instanceof UserNicknameChanged) {
            this.nickname = ((UserNicknameChanged) event).getNickname();
        }
    }

}
```

```
package ninja.disruptor.battleapp;

import com.airhacks.porcupine.execution.boundary.Dedicated;
import lombok.Getter;
import ninja.disruptor.battleapp.eventstore.entity.CoreEvent;
import ninja.disruptor.battleapp.kafka.control.KafkaProvider;
import ninja.disruptor.battleapp.user.entity.User;
import ninja.disruptor.battleapp.user.entity.event.UserCreated;
import ninja.disruptor.battleapp.user.entity.event.UserDeleted;
import ninja.disruptor.battleapp.user.entity.event.UserEvent;
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
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;

@Startup
@Singleton
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
public class InMemoryCache {

    @Getter
    private Map<String, User> users = new HashMap<>();

    @Inject
    KafkaConsumer<String, String> consumer;

    @Inject
    KafkaProducer<String, String> producer;

    @Dedicated
    @Inject
    ExecutorService kafka;

    @Inject
    JsonConverter converter;

    @PostConstruct
    public void onInit() {
        String topicName = getTopicName();
        JsonObject event = Json.createObjectBuilder()
                .add("topicName", topicName)
                .build();

        CompletableFuture
                .runAsync(this::handleKafkaEvent, kafka);

        consumer.subscribe(Arrays.asList(KafkaProvider.TOPIC, topicName));

        producer.send(new ProducerRecord<>(
                KafkaProvider.TOPIC,
                event.toString()));
    }

    public String getTopicName() {
        InetAddress localHost = null;
        try {
            localHost = InetAddress.getLocalHost();
        } catch (UnknownHostException e) {
            throw new RuntimeException(e);
        }
        return "replayAllFromStore" + localHost.getHostName();
    }

    public void handleKafkaEvent() {
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(200);
            for (ConsumerRecord<String, String> record : records) {
                switch (record.topic()) {
                    case KafkaProvider.TOPIC:
                        handeEvents(record);
                        break;
                    default:
                        handeEvents(record);
                        break;
                }
            }
        }
    }

    private void handeEvents(ConsumerRecord<String, String> record) {
        try {
            String eventText = record.value();
            System.out.println("eventText = " + eventText);
            List<CoreEvent> events = converter.convertToEvents(eventText);
            for (CoreEvent event : events) {
                handle(event);
            }
        } catch (Exception e) {
            System.out.println("Error: " + e.getMessage());
            e.printStackTrace();
        }
    }

    public void handle(CoreEvent event) {
        if (event instanceof UserCreated) {
            List<CoreEvent> events = new ArrayList<>();
            events.add(event);
            User user = new User(events);
            users.put(event.getId(), user);
        } else if (event instanceof UserDeleted) {
            User user = users.get(event.getId());
            if (user == null) {
                System.out.println("rejected!");
                return;
            }
            users.remove(user.getId());
        } else if (event instanceof UserEvent) {
            User user = users.get(event.getId());
            if (user == null) {
                System.out.println("rejected!");
                return;
            }
            user.mutate(event);
        } else {
            System.out.println("Event not found: " + event.toString());
        }
    }

}
```

```
package ninja.disruptor.battleapp;

import javax.ws.rs.ApplicationPath;
import javax.ws.rs.core.Application;

@ApplicationPath("resources")
public class JaxRsConfiguration extends Application {
}
```

```
package ninja.disruptor.battleapp;

import ninja.disruptor.battleapp.eventstore.entity.CoreEvent;
import ninja.disruptor.battleapp.user.entity.event.*;
import sun.reflect.generics.reflectiveObjects.NotImplementedException;

import javax.json.Json;
import javax.json.JsonArray;
import javax.json.JsonObject;
import javax.json.JsonObjectBuilder;
import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.List;

public class JsonConverter {

    public JsonObjectBuilder convertToJson(CoreEvent event) {
        JsonObjectBuilder jsonEvent = Json.createObjectBuilder()
                .add("name", event.getClass().getName())
                .add("id", event.getId());
        if (event instanceof UserCreated) {
            // no more to do
        } else if (event instanceof UserDeleted) {
            // no more to do
        } else if (event instanceof UserFirstNameChanged) {
            UserFirstNameChanged changedEvent = (UserFirstNameChanged) event;
            jsonEvent = jsonEvent
                    .add("firstName", changedEvent.getFirstName());
        } else if (event instanceof UserLastNameChanged) {
            UserLastNameChanged changedEvent = (UserLastNameChanged) event;
            jsonEvent = jsonEvent
                    .add("lastName", changedEvent.getLastName());
        } else if (event instanceof UserNicknameChanged) {
            UserNicknameChanged changedEvent = (UserNicknameChanged) event;
            jsonEvent = jsonEvent
                    .add("nickname", changedEvent.getNickname());
        } else {
            throw new NotImplementedException();
        }
        return jsonEvent;
    }

    public List<CoreEvent> convertToEvents(String jsonAsString) {
        List<CoreEvent> events = new ArrayList<>();
        InputStream inputStream = new ByteArrayInputStream(jsonAsString.getBytes(Charset.forName("UTF-8")));
        JsonArray eventArray = Json.createReader(inputStream).readArray();
        for (int i = 0; i < eventArray.size(); i++) {
            JsonObject eventObj = eventArray.getJsonObject(i);
            String name = eventObj.getString("name");
            String id = eventObj.getString("id");
            if (UserCreated.class.getName().equals(name)) {
                UserCreated event = new UserCreated(id);
                events.add(event);
            } else if (UserDeleted.class.getName().equals(name)) {
                UserDeleted event = new UserDeleted(id);
                events.add(event);
            } else if (UserFirstNameChanged.class.getName().equals(name)) {
                String firstName = eventObj.getString("firstName");
                UserFirstNameChanged event = new UserFirstNameChanged(id, firstName);
                events.add(event);
            } else if (UserLastNameChanged.class.getName().equals(name)) {
                String lastName = eventObj.getString("lastName");
                UserLastNameChanged event = new UserLastNameChanged(id, lastName);
                events.add(event);
            } else if (UserNicknameChanged.class.getName().equals(name)) {
                String nickname = eventObj.getString("nickname");
                UserNicknameChanged event = new UserNicknameChanged(id, nickname);
                events.add(event);
            } else {
                throw new NotImplementedException();
            }
        }
        return events;
    }
}
```

Create also a Jenkins pipeline similar to the existing REST service pipeline.
