# Changes we've to do in the REST service
In the backend we've to create a new event `UserProfilePictureChanged`.
The following files have to be changed or created:

`JsonConverter`
```
...
} else if (event instanceof UserProfilePictureChanged) {
    UserProfilePictureChanged changedEvent = (UserProfilePictureChanged) event;
    jsonEvent = jsonEvent
        .add("profilePicture", changedEvent.getProfilePicture());
...
} else if (UserProfilePictureChanged.class.getName().equals(name)) {
    String profilePicture = eventObj.getString("profilePicture");
    UserProfilePictureChanged event = new UserProfilePictureChanged(id, profilePicture);
    events.add(event);
...
```

`UserResource`
```
import com.airhacks.porcupine.execution.boundary.Dedicated;
import ninja.disruptor.battleapp.user.entity.User;

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

    @POST
    public void save(@Suspended AsyncResponse response, User user) {
        CompletableFuture
                .supplyAsync(service.save(user), usersPool)
                .thenAccept(response::resume);
    }

    @PUT
    @Path("{id}")
    public void update(@Suspended AsyncResponse response, @PathParam("id") String id, User user) {
        CompletableFuture
                .supplyAsync(service.update(id, user), usersPool)
                .thenAccept(response::resume);
    }

}
```

`UserService`
```
import ninja.disruptor.battleapp.eventstore.control.EventStore;
import ninja.disruptor.battleapp.eventstore.control.EventStream;
import ninja.disruptor.battleapp.eventstore.entity.EventIdentity;
import ninja.disruptor.battleapp.user.entity.User;
import org.keycloak.admin.client.Keycloak;
import org.keycloak.admin.client.resource.UsersResource;
import org.keycloak.representations.idm.UserRepresentation;

import javax.ejb.Stateless;
import javax.inject.Inject;
import javax.ws.rs.core.Response;
import java.util.ArrayList;
import java.util.function.Supplier;

@Stateless
public class UserService {

    private static final String KEYCLOAK_MASTER_CLIENT_ID = System.getenv("KEYCLOAK_MASTER_CLIENT_ID");
    private static final String KEYCLOAK_MASTER_USER_NAME = System.getenv("KEYCLOAK_MASTER_USER_NAME");
    private static final String KEYCLOAK_MASTER_PASSWORD = System.getenv("KEYCLOAK_MASTER_PASSWORD");
    private static final String REALM_NAME = System.getenv("REALM_NAME");
    private static final String AUTH_SERVER_URL = System.getenv("AUTH_SERVER_URL");

    public static final String KEYCLOAK_MASTER_ADMIN_CLIENT_ID = "admin-cli";

    @Inject
    EventStore store;

    public Supplier<Response> save(User user) {
        user = updateUserWithLastAndFirstNameFromKeycloak(user);

        User saved = create(user.getId());
        saved = updateUser(user, saved);

        final User postSaveUser = new User(saved);
        return () -> Response
                .status(Response.Status.CREATED)
                .entity(postSaveUser)
                .build();
    }

    public Supplier<Response> update(String id, User user) {
        User updated = updateUser(id, user);
        return () -> Response
                .ok()
                .entity(updated)
                .build();
    }

    public User updateUser(User template, User toUpdate) {
        if (template.getFirstName() != null) {
            toUpdate = changeFirstName(toUpdate.getId(), template.getFirstName());
        }
        if (template.getLastName() != null) {
            toUpdate = changeLastName(toUpdate.getId(), template.getLastName());
        }
        if (template.getNickname() != null) {
            toUpdate = changeNickname(toUpdate.getId(), template.getNickname());
        }
        if (template.getProfilePicture() != null) {
            toUpdate = changeProfilPicture(toUpdate.getId(), template.getProfilePicture());
        }
        return toUpdate;
    }

    public User updateUserWithLastAndFirstNameFromKeycloak(User user) {
        User copy = new User(user);
        if (copy.getLastName() == null || copy.getFirstName() == null) {
            Keycloak keycloak = Keycloak.getInstance(
                    AUTH_SERVER_URL,
                    KEYCLOAK_MASTER_CLIENT_ID,
                    KEYCLOAK_MASTER_USER_NAME,
                    KEYCLOAK_MASTER_PASSWORD,
                    KEYCLOAK_MASTER_ADMIN_CLIENT_ID);
            UsersResource users = keycloak
                    .realm(REALM_NAME)
                    .users();
            UserRepresentation keycloakUser = users
                    .get(copy.getId())
                    .toRepresentation();
            if (copy.getLastName() == null) {
                copy.setLastName(keycloakUser.getLastName());
            }
            if (copy.getFirstName() == null) {
                copy.setFirstName(keycloakUser.getFirstName());
            }
        }

        return copy;
    }

    public User create(String id) {
        User player = new User(new ArrayList<>());
        player.create(id);
        store.appendToStream(new EventIdentity(User.class, id), 0L, player.getChanges());
        return player;
    }

    public User updateUser(String id, User user) {
        EventStream stream = store.loadEventStream(new EventIdentity(User.class, id));
        User loaded = new User(stream.getEvents());
        return updateUser(user, loaded);
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

    public User changeProfilPicture(String id, String profilePicture) {
        EventStream stream = store.loadEventStream(new EventIdentity(User.class, id));
        User user = new User(stream.getEvents());
        user.changeProfilePicture(profilePicture);
        store.appendToStream(new EventIdentity(User.class, id), stream.getVersion(), user.getChanges());
        return user;
    }

}
```

`User`
```
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import ninja.disruptor.battleapp.eventstore.entity.CoreEvent;
import ninja.disruptor.battleapp.user.entity.event.*;

import javax.xml.bind.annotation.XmlAccessType;
import javax.xml.bind.annotation.XmlAccessorType;
import javax.xml.bind.annotation.XmlTransient;
import java.util.ArrayList;
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
    private String profilePicture;

    @XmlTransient
    private final List<CoreEvent> changes = new ArrayList<>();

    public User(User copy) {
        this.id = copy.getId();
        this.nickname = copy.getNickname();
        this.firstName = copy.getFirstName();
        this.lastName = copy.getLastName();
        this.profilePicture = copy.getProfilePicture();
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

    public void changeProfilePicture(String profilePicture) {
        apply(new UserProfilePictureChanged(id, profilePicture));
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
        } else if (event instanceof UserProfilePictureChanged) {
            this.profilePicture = ((UserProfilePictureChanged) event).getProfilePicture();
        }
    }

}
```

`UserProfilePictureChanged`
```
import lombok.Getter;

@Getter
public class UserProfilePictureChanged extends UserEvent {
    private final String profilePicture;

    public UserProfilePictureChanged(String id, String profilePicture) {
        super(id);
        this.profilePicture = profilePicture;
    }
}
```

