# Changes to the frontend

We've to tell the `user.service.ts` to differ between the command
and the query side. The new service results in the following code:

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

  public getUsersCommandUrl(env: any): string {
    let usersUrl = 'http://' + env.host + ':' + env.command_port + '/battleapp/resources/users/';
    return usersUrl;
  }

  public getUsersQueryUrl(env: any): string {
    let usersUrl = 'http://' + env.host + ':' + env.query_port + '/battleapp/resources/users/';
    return usersUrl;
  }

  public getAll = (): Observable<User[]> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .get(this.getUsersQueryUrl(env))
        .map(res => res.json());
    });
  };

  public search = (nickname: string): Observable<User[]> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .get(this.getUsersQueryUrl(env) + "?nickname=" + nickname)
        .map(res => res.json());
    });
  };

  public find = (id: number): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .get(this.getUsersQueryUrl(env) + id)
        .map(res => res.json());
    });
  };

  public create = (id: string, nickname: string, firstName: string, lastName: string): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      var toAdd = JSON.stringify({id: id, nickname: nickname, firstName: firstName, lastName: lastName});
      return this.authHttp
        .post(this.getUsersCommandUrl(env), toAdd)
        .map(res => res.json());
    });
  };

  public update = (id: number, itemToUpdate: User): Observable<User> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .put(this.getUsersCommandUrl(env) + id, JSON.stringify(itemToUpdate))
        .map(res => res.json());
    });
  };

  public delete = (id: number): Observable<Response> => {
    return this.environment.flatMap((env: any) => {
      return this.authHttp
        .delete(this.getUsersCommandUrl(env) + id);
    });
  };
}
```

Therefore we also have to update the `environment.json` with the different
ports for command and query:

```
{
  "host": "disruptor.ninja",
  "command_port": 30080,
  "query_port": 30081
}
```

And the same for the test environment:

```
{
  "host": "disruptor.ninja",
  "command_port": 31080,
  "query_port": 31081
}
```

Delete the existing config maps and create the config maps
again with the updated `environment.json`.