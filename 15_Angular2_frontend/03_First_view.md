# Creation of the first view
We're going to create an own view that displays the users from the REST service.

## Creation of the template
Therefore we create a new folder `src/app/users` with the following template
`users.component.html`:

```
<md-input placeholder="Nickname" (keyup)="search($event)"></md-input>

<md-card *ngFor="let user of users">
  Name: {{ user.name }}
</md-card>
```

We're going to use [Angular 2 Material](https://github.com/angular/material2). 
Therefore we have to install it.

```
npm install --save @angular/material
```

In the `src/app/app.module.ts` file we've to include Material:

```
...
import {MaterialModule} from "@angular/material";
...
imports: [ // import Angular's modules
    BrowserModule,
    FormsModule,
    HttpModule,
    MaterialModule.forRoot(),
    RouterModule.forRoot(ROUTES, {useHash: true, preloadingStrategy: PreloadAllModules})
  ],
...
```

Install `hammerjs`. 
```
npm install --save hammerjs 
```

In `src/index.html` add the following line in the header:

```
<link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
```

Create an empty `users.component.css` file in the same folder as the 
html template.

## Creation of the service
For the service we need a `user.ts` object in the same folder as the 
template files:

```
export class User {
  name: string;

  constructor() {
  }

}
```

The next class is the service for the component:

```
import {Injectable} from "@angular/core";
import {RequestOptions, Response, Headers, Http} from "@angular/http";
import {Observable} from "rxjs";
import {User} from "./user";

@Injectable()
export class UserService {
  private environment: Observable<any>;

  constructor(private http: Http) {
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
      return this.http
        .get(this.getUsersUrl(env), new RequestOptions({headers: this.getHeaders()}))
        .map(res => res.json());
    });
  };

  public search = (nickname: string): Observable<User[]> => {
    return this.environment.flatMap((env: any) => {
      return this.http
        .get(this.getUsersUrl(env) + "?nickname=" + nickname, new RequestOptions({headers: this.getHeaders()}))
        .map(res => res.json());
    });
  };

  public find = (id: number): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      return this.http
        .get(this.getUsersUrl(env) + id, new RequestOptions({headers: this.getHeaders()}))
        .map(res => res.json());
    });
  };

  public create = (firstName: string, lastName: string): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      var toAdd = JSON.stringify({firstName: firstName, lastName: lastName});
      return this.http
        .post(this.getUsersUrl(env), toAdd, new RequestOptions({headers: this.getHeaders()}))
        .map(res => res.json());
    });
  };

  public update = (id: number, itemToUpdate: User): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      return this.http
        .put(this.getUsersUrl(env) + id, JSON.stringify(itemToUpdate), new RequestOptions({headers: this.getHeaders()}))
        .map(res => res.json());
    });
  };

  public delete = (id: number): Observable<Response> => {
    return this.environment.flatMap((env: any) => {
      return this.http
        .delete(this.getUsersUrl(env) + id, new RequestOptions({headers: this.getHeaders()}));
    });
  };

  private getHeaders() {
    let headers = new Headers();
    headers.append('Content-Type', 'application/json');
    headers.append('Accept', 'application/json');
    return headers;
  }
}
```

We're going to read the host and the port of the REST service url from a file
called `environment.json`. Therefore we're going to create this file
`src/environment/environment.json` with the following content:

```
{
  "host": "localhost",
  "port": 8080
}
```

## Creation of the component
The next step is to create the TypeScript component `users.component.ts` in the
same folder as the service and the other files:

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

  constructor(private usersService: UserService) {
  }

  ngOnInit() {
    this.getUsers();
  }

  public search(event: any) {
    let searchTerm: string = event.target.value;
    this.usersService
      .search(searchTerm)
      .subscribe((data: User[]) => {
          this.users = data;
        },
        error => console.log(error)
      );
  }

  private getUsers() {
    this.usersService
      .getAll()
      .subscribe((data: User[]) => {
          this.users = data;
        },
        error => console.log(error)
      );
  };

}
```

## Include the component in site
In `src/app/app.component.ts` we've to include the newly created user page.
Therefore we refactor the template of the app component in a own template
file:

```
@Component({
  selector: 'app',
  encapsulation: ViewEncapsulation.None,
  styleUrls: [
    './app.component.css'
  ],
  templateUrl: './app.component.html'
})
```

This file contains the navigation with our new user page:
```
<md-sidenav-layout>

  <md-sidenav #sidenav mode="side" class="app-sidenav">
    <nav>
      <p>
        <a [routerLink]=" ['./'] ">
          Index
        </a>
      </p>
      <p>
        <a [routerLink]=" ['./home'] ">
          Home
        </a>
      </p>
      <p>
        <a [routerLink]=" ['./detail'] ">
          Detail
        </a>
      </p>
      <p>
        <a [routerLink]=" ['./about'] ">
          About
        </a>
      </p>
      <p>
        <a [routerLink]=" ['./users'] ">
          Users
        </a>
      </p>
    </nav>
  </md-sidenav>

  <md-toolbar color="primary">
    <button class="app-icon-button" (click)="sidenav.toggle()">
      <i class="material-icons app-toolbar-menu">menu</i>
    </button>

    Battle App

  </md-toolbar>

  <div class="app-content">
    <main>
      <router-outlet></router-outlet>
    </main>
  </div>

</md-sidenav-layout>

<footer>
  Robert Brem
</footer>
```

The `app.module.ts` class gets extended with our `UserComponent`:
```
...
@NgModule({
  bootstrap: [AppComponent],
  declarations: [
    AppComponent,
    AboutComponent,
    HomeComponent,
    UserComponent,
    NoContentComponent,
    XLarge
  ],
...
```

Like the `app.module.ts` the `app.routes.ts` gets extended with our 
`UserComponent`:
```
export const ROUTES: Routes = [
  {path: '', component: HomeComponent},
  {path: 'home', component: HomeComponent},
  {path: 'about', component: AboutComponent},
  {
    path: 'detail', loadChildren: () => System.import('./+detail')
    .then((comp: any) => comp.default),
  },
  {path: 'users', component: UserComponent},
  {path: '**', component: NoContentComponent},
];
```
