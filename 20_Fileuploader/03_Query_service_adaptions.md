# Changes we've to do in the query service
In the backend we've to create a new event `UserProfilePictureChanged`.
The following files have to be changed or created:

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

`User`
```
...
private String profilePicture;
...
} else if (event instanceof UserProfilePictureChanged) {
    this.profilePicture = ((UserProfilePictureChanged) event).getProfilePicture();
...
```

`UserResource`
```
...
@GET
@Path("{id}")
public void getUser(@Suspended AsyncResponse response, @PathParam("id") String id) {
    CompletableFuture
        .supplyAsync(() -> service.getUser(id), usersPool)
        .thenAccept(response::resume);
}
...
```

`UserService`
```
...
public User getUser(String id) {
    return cache
            .getUsers()
            .get(id);
}
...
```