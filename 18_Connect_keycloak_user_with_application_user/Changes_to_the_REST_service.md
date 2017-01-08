# Changes we've to do in the REST service
The login event just contains the username and the id. But
the user enters his first and last name as well. To set this
two properties in the REST service we've to change the
`UserService`:

```
...
private static final String KEYCLOAK_MASTER_CLIENT_ID = System.getenv("KEYCLOAK_MASTER_CLIENT_ID");
private static final String KEYCLOAK_MASTER_USER_NAME = System.getenv("KEYCLOAK_MASTER_USER_NAME");
private static final String KEYCLOAK_MASTER_PASSWORD = System.getenv("KEYCLOAK_MASTER_PASSWORD");
private static final String REALM_NAME = System.getenv("REALM_NAME");
private static final String AUTH_SERVER_URL = System.getenv("AUTH_SERVER_URL");

public static final String KEYCLOAK_MASTER_ADMIN_CLIENT_ID = "admin-cli";
...
public Supplier<Response> save(User user, UriInfo info) {
    user = updateUserWithLastAndFirstNameFromKeycloak(user);

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
            .entity(postSaveUser)
            .build();
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
...
```

And add the following dependency:

```
<dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-admin-client</artifactId>
    <version>2.4.0.Final</version>
</dependency>
```