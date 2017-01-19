# Secure the Angular2 frontend service
This article is based on 
[this article](http://paulbakker.io/java/jwt-keycloak-angular2/).
To add Keycloak security to an Angular2 application we've to install the
JavaScript Keycloak adapter.

```
npm install keycloak-js --save
```

Now we've to intercept every call to the frontend. We can achieve that in the
`main.browser.ts` file:

```
import * as Keycloak from "keycloak-js";

let keycloak = Keycloak('keycloak/keycloak.json');
window['_keycloak'] = keycloak;

window['_keycloak'].init(
  {onLoad: 'login-required'}
)
  .success(function (authenticated) {

    if (!authenticated) {
      window.location.reload();
    }

    // refresh login
    setInterval(function () {

      keycloak.updateToken(70).success(function (refreshed) {
        if (refreshed) {
          console.log('Token refreshed');
        } else {
          console.log('Token not refreshed, valid for '
            + Math.round(keycloak.tokenParsed.exp + keycloak.timeSkew - new Date().getTime() / 1000) + ' seconds');
        }
      }).error(function () {
        console.error('Failed to refresh token');
      });

    }, 60000);

    console.log("Loading...");

    platformBrowserDynamic().bootstrapModule(AppModule);

  });
```

Then we've to create a folder `src/keycloak` with the file `keycloak.json` and
the following content:
```
{
  "realm": "battleapp-local",
  "auth-server-url": "http://localhost:8280/auth",
  "ssl-required": "none",
  "resource": "battleapp-frontend",
  "public-client": true
}
```

To include Keycloak in our services we can use the following library:
```
npm install angular2-jwt --save
```

Then we can configure the auth provider in `app.module.ts`:
```
...
import {provideAuth} from "angular2-jwt";
...
providers: [ // expose our Services and Providers into Angular's dependency injection
  ENV_PROVIDERS,
  APP_PROVIDERS,
  provideAuth({
    globalHeaders: [{'Content-Type': 'application/json'}],
    noJwtError: true,
    tokenGetter: () => {
      return window['_keycloak'].token;
    }
  })
]
...
```

Now we can use the `AuthHttp` in our `users.service.ts` class.
```
import {Injectable} from "@angular/core";
import {Response, Http} from "@angular/http";
import {Observable} from "rxjs";
import {User} from "./user";
import {AuthHttp} from "angular2-jwt";

@Injectable()
export class UserService {
  private environment: Observable<any>;

  constructor(private authHttp: AuthHttp, private http: Http) {
    this.environment = this.http
      .get('/environment/environment.json')
      .map(res => res.json());
  }

  public getUsersUrl(env: any): string {
    let baseUsersUrl = '/battleapp/';
    let api: string = 'resources/';
    let usersUrl = 'http://' + env.host + ':' + env.port + baseUsersUrl + api + 'users/';
    return usersUrl;
  }

  public getAll = (): Observable<User[]> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .get(this.getUsersUrl(env))
        .map(res => res.json());
    });
  };

  public search = (nickname: string): Observable<User[]> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .get(this.getUsersUrl(env) + "?nickname=" + nickname)
        .map(res => res.json());
    });
  };

  public find = (id: number): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .get(this.getUsersUrl(env) + id)
        .map(res => res.json());
    });
  };

  public create = (firstName: string, lastName: string): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      var toAdd = JSON.stringify({firstName: firstName, lastName: lastName});
      return this.authHttp
        .post(this.getUsersUrl(env), toAdd)
        .map(res => res.json());
    });
  };

  public update = (id: number, itemToUpdate: User): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .put(this.getUsersUrl(env) + id, JSON.stringify(itemToUpdate))
        .map(res => res.json());
    });
  };

  public delete = (id: number): Observable<Response> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .delete(this.getUsersUrl(env) + id);
    });
  };
}
```

## Create test environment with security
In the `start.js` script form the start test environment script we've to
add `keycloak.json` with a ConfigMap. Therefore we've to create a new file
`keycloak.json` in the `battleapp-frontend` and `battleapp-frontend-test`
folder with the corresponding content:

```
{
  "realm": "battleapp",
  "auth-server-url": "https://disruptor.ninja:30182/auth",
  "ssl-required": "none",
  "resource": "battleapp-frontend",
  "public-client": true
}
```

```
{
  "realm": "battleapp-test",
  "auth-server-url": "https://disruptor.ninja:30182/auth",
  "ssl-required": "none",
  "resource": "battleapp-frontend",
  "public-client": true
}
```

Now we've to delete the existing ConfigMaps and recreate them:
```
kc delete configmap battleapp-frontend
kc delete configmap battleapp-frontend-test
kc create configmap battleapp-frontend-test --from-file=battleapp-frontend-test
kc create configmap battleapp-frontend --from-file=battleapp-frontend
```

And test it:
```
kc get configmap battleapp-frontend-test -o yaml
```
```
apiVersion: v1
data:
  environment.json: |
    {
      "host": "disruptor.ninja",
      "port": 31080
    }
  keycloak.json: |
    {
      "realm": "battleapp-test",
      "auth-server-url": "https://disruptor.ninja:30182/auth",
      "ssl-required": "none",
      "resource": "battleapp-frontend",
      "public-client": true
    }
kind: ConfigMap
metadata:
  creationTimestamp: 2016-12-31T15:54:44Z
  name: battleapp-frontend-test
  namespace: default
  resourceVersion: "1049486"
  selfLink: /api/v1/namespaces/default/configmaps/battleapp-frontend-test
  uid: 72a5399b-cf71-11e6-a836-0050563cad2a
```

Now we can add the ConfigMap in the `start.js` script form the start 
test environment script:
```
...
dfw.write("        - name: keycloak\n");
dfw.write("          mountPath: /usr/share/nginx/html/keycloak\n");
...
dfw.write("      - name: keycloak\n");
dfw.write("        configMap:\n");
dfw.write("          name: battleapp-frontend-test\n");
dfw.write("          items:\n");
dfw.write("          - key: keycloak.json\n");
dfw.write("            path: keycloak.json\n");
...
```

And in the `start.js` script from the canary release:
```
...
dfw.write("        - name: keycloak\n");
dfw.write("          mountPath: /usr/share/nginx/html/keycloak\n");
...
dfw.write("      - name: keycloak\n");
dfw.write("        configMap:\n");
dfw.write("          name: battleapp-frontend\n");
dfw.write("          items:\n");
dfw.write("          - key: keycloak.json\n");
dfw.write("            path: keycloak.json\n");
...
```

