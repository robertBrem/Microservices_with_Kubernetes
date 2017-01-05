# Change the REST service to use event sourcing and Cassandra

## Event Sourcing in the REST service
In the book 
[Implementing Domain Driven Design](https://www.amazon.de/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577)
there's a chapter with a C# implementation of Event Sourcing. I've implemented
my version of the Event Sourcing based on this example from the book.
The implementation is under the following package 
`ninja.disruptor.battleapp.eventstore` in 
[this repository](http://disruptor.ninja:30130/rob/battleapp/src/master/src/main/java/ninja/disruptor/battleapp/eventstore).

Under `ninja.disruptor.battleapp.user.entity.event` we've to create the needed events for the `User`.
Our user has the following properties:
```
private String id;
private String nickname;
private String firstName;
private String lastName;
```

Therefore we need these event classes:

```
@AllArgsConstructor
@Getter
public class UserEvent implements CoreEvent {
    private final String id;
}
```

```
public class UserCreated extends UserEvent {
    public UserCreated(String id) {
        super(id);
    }
}
```

```
public class UserDeleted extends UserEvent {
    public UserDeleted(String id) {
        super(id);
    }
}
```

```
@Getter
public class UserFirstNameChanged extends UserEvent {
    private final String firstName;

    public UserFirstNameChanged(String id, String firstName) {
        super(id);
        this.firstName = firstName;
    }
}
```

```
@Getter
public class UserLastNameChanged extends UserEvent {
    private final String lastName;

    public UserLastNameChanged(String id, String lastName) {
        super(id);
        this.lastName = lastName;
    }

}
```

```
@Getter
public class UserNicknameChanged extends UserEvent {
    private final String nickname;

    public UserNicknameChanged(String id, String nickname) {
        super(id);
        this.nickname = nickname;
    }

}
```

At next we've to adapt our `User` entity itself:

```
@NoArgsConstructor
@ToString
@XmlAccessorType(XmlAccessType.FIELD)
@Data
public class User {
    private String id;
    private String nickname;
    private String firstName;
    private String lastName;

    @XmlTransient
    private final List<CoreEvent> changes = new ArrayList<>();

    public User(User copy) {
        this.id = copy.getId();
        this.nickname = copy.getNickname();
        this.firstName = copy.getFirstName();
        this.lastName = copy.getLastName();
    }

    public User(List<CoreEvent> events) {
        for (CoreEvent event : events) {
            mutate(event);
        }
    }

    public void changeFirstName(String firstName) {
        apply(new UserFirstNameChanged(id, firstName));
    }

    public void changeLastName(String lastName) {
        apply(new UserLastNameChanged(id, lastName));
    }

    public void changeNickname(String nickname) {
        apply(new UserNicknameChanged(id, nickname));
    }

    public void create(String id) {
        apply(new UserCreated(id));
    }

    public void delete() {
        apply(new UserDeleted(id));
    }

    public void mutate(CoreEvent event) {
        when(event);
    }

    public void apply(CoreEvent event) {
        changes.add(event);
        mutate(event);
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

Implement a `createUser` method in our `UserService` EJB:

```
...

@Inject
EventStore store;

public Supplier<Response> save(User user, UriInfo info) {
    User saved = create(user.getId());
    if (user.getFirstName() != null) {
        saved = changeFirstName(saved.getId(), user.getFirstName());
    }
    if (user.getLastName() != null) {
        saved = changeLastName(saved.getId(), user.getLastName());
    }
    if (user.getNickname() != null) {
        saved = changeNickname(saved.getId(), user.getNickname());
    }
    String id = saved.getId();
    URI uri = info
            .getAbsolutePathBuilder()
            .path("/" + id)
            .build();
    final User postSaveUser = new User(saved);
    return () -> Response
            .created(uri)
            .entity(new User(postSaveUser))
            .build();
}

public User create(String id) {
    User player = new User(new ArrayList<>());
    player.create(id);
    store.appendToStream(new EventIdentity(User.class, id), 0L, player.getChanges());
    return player;
}

public User changeFirstName(String id, String firstName) {
    EventStream stream = store.loadEventStream(new EventIdentity(User.class, id));
    User user = new User(stream.getEvents());
    user.changeFirstName(firstName);
    store.appendToStream(new EventIdentity(User.class, id), stream.getVersion(), user.getChanges());
    return user;
}

public User changeLastName(String id, String lastName) {
    EventStream stream = store.loadEventStream(new EventIdentity(User.class, id));
    User user = new User(stream.getEvents());
    user.changeLastName(lastName);
    store.appendToStream(new EventIdentity(User.class, id), stream.getVersion(), user.getChanges());
    return user;
}

public User changeNickname(String id, String nickname) {
    EventStream stream = store.loadEventStream(new EventIdentity(User.class, id));
    User user = new User(stream.getEvents());
    user.changeNickname(nickname);
    store.appendToStream(new EventIdentity(User.class, id), stream.getVersion(), user.getChanges());
    return user;
}

...
```

To expose the `create` method we've to create a REST service method in the `UserResource` as well.

```
@POST
public void save(@Suspended AsyncResponse response, @Context UriInfo info, User user) {
    CompletableFuture
            .supplyAsync(service.save(user, info), usersPool)
            .thenAccept(response::resume);
}
```

Our event sourcing implementation uses Cassandre therefore we've to add the following dependencies:

```
<dependency>
    <groupId>com.datastax.cassandra</groupId>
    <artifactId>cassandra-driver-core</artifactId>
    <version>3.1.2</version>
</dependency>
<dependency>
    <groupId>com.datastax.cassandra</groupId>
    <artifactId>cassandra-driver-mapping</artifactId>
    <version>3.1.2</version>
</dependency>
```

To configure Cassandra also have to create a `CassandraProvider`:

```
@Singleton
public class CassandraProvider {
    public static final String CASSANDRA_ADDRESS = "CASSANDRA_ADDRESS";
    public static final String KEYSPACE = "battleapp";

    private Session session;

    @Produces
    public Session getSession() {
        if (session == null) {
            String address = "localhost";
            String cassandraEnv = System.getenv(CASSANDRA_ADDRESS);
            if (cassandraEnv != null && !cassandraEnv.isEmpty()) {
                address = cassandraEnv;
            }
            Cluster cluster = Cluster.builder()
                    .addContactPoints(address)
                    .build();
            session = cluster.connect(KEYSPACE);
        }
        return session;
    }

}
```

To display the created events we need a database or in our case just a simple `Map` to store the
users.

```
@Startup
@Singleton
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
public class InMemoryCache {

    @Getter
    private Map<String, User> users = new HashMap<>();

    @Inject
    JsonConverter converter;

    @Inject
    EventRepository repository;

    @PostConstruct
    public void init() {
        repository
                .findAllUsers()
                .parallelStream()
                .forEach(u -> users.put(u.getId(), u));
    }

    public void onEvent(@Observes String jsonEventString) {
        List<CoreEvent> events = converter.convertToEvents(jsonEventString);
        for (CoreEvent event : events) {
            handle(event);
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
            throw new NotImplementedException();
        }
    }

}
```

Now we can change our `users` REST service to use the users from the db instead the hardcoded
test users. In the `UserService` we've to include the following parts:

```
...
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
...
```

And in our `UserResource`:

```
@GET
public void getUsers(@Suspended AsyncResponse response, @QueryParam("nickname") String nickname) {
    CompletableFuture
            .supplyAsync(service.getUsersFilteredByNickname(nickname), usersPool)
            .thenAccept(response::resume);
}
```

## Changes to the test environment
That we can use the Docker repository we've to create the same 
`docker-registry` `secret` from the production environment for the 
test environment too.

```
kc --namespace test create secret docker-registry registrykey --docker-username=rob --docker-password=1234 --docker-email=brem_robert@hotmail.com --docker-server=disruptor.ninja:30500
```

The following changes have to be made to the `start.js`:

``` 
...
var name = "battleapp";
var cassandraHost = "cassandra";
var namespace = "test";
...
var deleteDeployment = kubectl + " --namespace " + namespace + " delete deployment " + name;
...
dfw.write("metadata:\n");
dfw.write("  namespace: " + namespace + "\n");
...
dfw.write("        - name: CASSANDRA_ADDRESS\n");
dfw.write("          value: \"" + cassandraHost + "\"\n");
...
var deleteService = kubectl + " --namespace " + namespace + " delete service " + name;
...
sfw.write("metadata:\n");
sfw.write("  name: " + name + "\n");
sfw.write("  namespace: " + namespace + "\n");
...
```

## Changes to the canary release
The following changes have to be made to the `start.js`:

```
...
var cassandraHost = "cassandra";
...
dfw.write("        - name: CASSANDRA_ADDRESS\n");
dfw.write("          value: \"" + cassandraHost + "\"\n");
...
```

## Changes to the consumer driven contract test
We need the following two dependencies that we can call the create method of the REST service:

```
<dependency>
    <groupId>org.glassfish</groupId>
    <artifactId>javax.json</artifactId>
    <version>1.0.4</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.glassfish.jersey.media</groupId>
    <artifactId>jersey-media-json-processing</artifactId>
    <version>2.12</version>
    <scope>test</scope>
</dependency>
```

The test itself has to be replaced with the following code:

```
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class BattleAppIT {

    public static final String ID = "id";
    public static final String NICKNAME = "nickname";
    public static final String FIRST_NAME = "firstName";
    public static final String LAST_NAME = "lastName";

    public static final String LOCATION = "Location";

    @Rule
    public JAXRSClientProvider provider = buildWithURI("http://" + System.getenv("HOST") + ":" + System.getenv("PORT") + "/battleapp/resources/users");

    private String nickname = "rob";
    private String firstName = "Robert";
    private String lastName = "Brem";


    @Test
    public void a01_shouldCreateRob() throws IOException {
        JsonObjectBuilder userBuilder = Json.createObjectBuilder();
        JsonObject playerToCreate = userBuilder
                .add(ID, UUID.randomUUID().toString())
                .add(NICKNAME, nickname)
                .add(FIRST_NAME, firstName)
                .add(LAST_NAME, lastName)
                .build();

        String token = KeycloakTokenCreator
                .getTokenResponse(
                        System.getenv("APPLICATION_USER_NAME"),
                        System.getenv("APPLICATION_PASSWORD"))
                .getToken();

        Response postResponse = provider
                .target()
                .request()
                .header("Authorization", "Bearer " + token)
                .post(Entity.json(playerToCreate));
        System.out.println("postResponse = " + postResponse);
        assertThat(postResponse.getStatus(), is(201));
        String location = postResponse.getHeaderString(LOCATION);
    }

    @Test
    public void a02_shouldReturnRob() throws IOException {
        String token = KeycloakTokenCreator
                .getTokenResponse(
                        System.getenv("APPLICATION_USER_NAME"),
                        System.getenv("APPLICATION_PASSWORD"))
                .getToken();
        String response = provider
                .target()
                .request()
                .header("Authorization", "Bearer " + token)
                .get(String.class);
        assertThat(response, containsString(nickname));
    }

}
```

