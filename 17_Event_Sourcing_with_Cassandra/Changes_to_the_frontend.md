# Change to the frontend service

## Usage of the new REST method
We've created a create method in the REST service now we've to
call this service in the frontend. Therefore the following
files have to be adapted:

`user.ts`
```
export class User {
  id: string;
  nickame: string;
  firstname: string;
  lastname: string;

  constructor() {
  }

}
```

`user.component.html`
```
<form>
  <md-card>
    <md-card-title-group>
      <md-input #id placeholder="ID"></md-input>
      <md-input #nickname placeholder="Nickname"></md-input>
      <md-input #firstName placeholder="First Name"></md-input>
      <md-input #lastName placeholder="Last Name"></md-input>
    </md-card-title-group>
  </md-card>
  <button md-raised-button
          (click)="createUser(id.value, nickname.value, firstName.value, lastName.value)"
          class="btn btn-primary btn-lg">
    create
  </button>
</form>

<md-input placeholder="Nickname" (keyup)="search($event)"></md-input>

<md-card *ngFor="let user of users">
  <md-card-title-group>
    <img md-card-sm-image src="path/to/img.png">
    <md-card-title>{{ user.nickname }}</md-card-title>
    <md-card-subtitle>{{ user.firstName }}</md-card-subtitle>
    <md-card-subtitle>{{ user.lastName }}</md-card-subtitle>
  </md-card-title-group>
</md-card>
```

`user.component.ts`
```
import {User} from "./user";
import {UserService} from "./users.service";
import {Component} from "@angular/core";

@Component({
  selector: 'battleapp-user',
  templateUrl: './users.component.html',
  styleUrls: ['./users.component.css'],
  providers: [UserService],
})
export class UserComponent {
  private users: User[];

  constructor(private service: UserService) {
  }

  ngOnInit() {
    this.getUsers();
  }

  public createUser(id: string, nickname: string, firstName: string, lastName: string) {
    return this.service
      .create(id, nickname, firstName, lastName)
      .subscribe((data: User) => {
          let user: User = data;
          console.log(user);
          this.users.push(user);
        },
        error => console.log(error)
      );
  }

  public search(event: any) {
    let searchTerm: string = event.target.value;
    this.service
      .search(searchTerm)
      .subscribe((data: User[]) => {
          this.users = data;
        },
        error => console.log(error)
      );
  }

  private getUsers() {
    this.service
      .getAll()
      .subscribe((data: User[]) => {
          this.users = data;
        },
        error => console.log(error),
        () => console.log(this.users)
      );
  }
  ;

}
```

`user.service.ts`
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
    let usersUrl = 'http://' + env.host + ':' + env.port + '/battleapp/resources/users/';
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

  public create = (id: string, nickname: string, firstName: string, lastName: string): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      var toAdd = JSON.stringify({id: id, nickname: nickname, firstName: firstName, lastName: lastName});
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

## Alias for test environment
That we can execute commands in the test namespace we're going
to create a new alias:

```
vi ~/.alias
```
```
alias kc='kubectl --kubeconfig /home/battleapp/Desktop/admin.conf'
alias watchkc='watch kubectl --kubeconfig /home/battleapp/Desktop/admin.conf'
alias mountWin='sudo mount -t vboxsf Microservices_with_Kubernetes /media/windows-share'
alias startKeycloak='docker stop keycloak && docker rm keycloak && docker run -d -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin -p 8280:8080 -v /home/battleapp/Desktop/dockervolumes/keycloakdata/:/opt/jboss/keycloak/standalone/data --name keycloak jboss/keycloak:2.4.0.Final'
alias startCass='~/Desktop/startCassandra.sh'
alias kctest='kubectl --kubeconfig /home/battleapp/Desktop/admin.conf --namespace test'
```

## Changes to the test environment
In the frontend we need additionally to the Kubernetes secret
for the registry a Kubernetes configmap:

```
kc delete configmap battleapp-frontend-test
kctest create configmap battleapp-frontend --from-file battleapp-frontend-test
```

Changes to the `start.js` script:

```
...
var name = "battleapp-frontend";
...
dfw.write("        configMap:\n");
dfw.write("          name: battleapp-frontend\n");
...
dfw.write("        configMap:\n");
dfw.write("          name: battleapp-frontend\n");
...
var deleteDeployment = kubectl + " --namespace " + namespace + " delete deployment " + name;
...
dfw.write("kind: Deployment\n");
dfw.write("metadata:\n");
dfw.write("  name: " + name + "\n");
dfw.write("  namespace: " + namespace + "\n");
...
var deleteService = kubectl + " --namespace " + namespace + "delete service " + name;
...
sfw.write("kind: Service\n");
sfw.write("metadata:\n");
sfw.write("  name: " + name + "\n");
sfw.write("  namespace: " + namespace + "\n");
...
```

