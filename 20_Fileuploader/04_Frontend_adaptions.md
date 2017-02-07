# Changes we've to do in the fronend
The following files have to be changed or created:

`fileupload.component.css`
```
.button {
    overflow: hidden;
    position: relative;
    background-color: #212121;
    color: whitesmoke;
    padding: 0.5em;
}

.button [type=file] {
    cursor: inherit;
    display: inline-block;
    font-size: 999px;
    filter: alpha(opacity=0);
    min-height: 100%;
    min-width: 100%;
    opacity: 0;
    position: absolute;
    right: 0;
    text-align: right;
    top: 0;
}
```

`fileupload.component.html`
```
<label class="button">
    Upload...
    <input
            type="file"
            (change)="upload($event)"
            [multiple]="multiple"/>
</label>
```

`fileupload.component.ts`
```
import {Component, Input, Output, EventEmitter} from "@angular/core";
import {Headers, Http, RequestOptions} from "@angular/http";
import {UserService} from "../users/users.service";
import {User} from "../users/user";

@Component({
    selector: 'file-upload',
    templateUrl: './fileupload.component.html',
    styleUrls: ['./fileupload.component.css']
})
export class FileUploadComponent {
    @Input()
    multiple: boolean = false;

    @Input()
    url: string = "";

    @Output()
    notify: EventEmitter<string> = new EventEmitter<string>();

    constructor(private http: Http, private service: UserService) {
    }

    public upload = (event: EventTarget) => {
        let _self = this;
        window['_keycloak']
            .loadUserInfo()
            .success(function (userInfo) {
                _self.uploadWithUserId(userInfo.sub, _self.url, event);
            });
    };

    public uploadWithUserId = (userId: string, url: string, event: EventTarget) => {
        let eventObj: MSInputMethodContext = <MSInputMethodContext> event;
        let target: HTMLInputElement = <HTMLInputElement> eventObj.target;
        let files: FileList = target.files;
        let fileCount: number = files.length;
        let formData = new FormData();

        if (fileCount > 0) {
            for (let i = 0; i < fileCount; i++) {
                let fileUrl = '/battleapp/users/' + userId + '/profilepicture';
                formData.append(fileUrl, files.item(i));
            }

            let token = window['_keycloak'].token;
            let headers = new Headers();
            headers.append('Authorization', 'Bearer ' + token);
            let options = new RequestOptions({headers: headers});

            this.http
                .post(url, formData, options)
                .map(res => res.json())
                .subscribe((data: any) => {
                        for (var arrayKey in data) {
                            for (var key in data[arrayKey]) {
                                var value = data[arrayKey][key];
                                let user: User = new User(userId, undefined, undefined, undefined, value);
                                this.service
                                    .update(userId, user)
                                    .subscribe((updatedUser: User) => {
                                            this.notify.emit(updatedUser.profilePicture);
                                        },
                                        error => console.log('ERROR: ' + error)
                                    );
                            }
                        }
                    },
                    error => console.log('ERROR: ' + error)
                );
        }

    }

}
```

`profile.component.css`
```
md-card img {
    max-width: 90%;
    max-height: 300px;
    height: auto;
    width: auto;
    margin: 1em auto;
    display: block;
}

md-card {
    max-width: 600px;
}
```

`profile.component.html`
```
<md-card>
    <md-card-header>
        <md-card-title>{{ user.nickname }}</md-card-title>
        <md-card-subtitle>{{ user.firstName + ' ' + user.lastName }}</md-card-subtitle>
    </md-card-header>
    <img md-card-image src="{{ user.profilePicture }}">
    <md-card-content>
        <p>Upload a profile picture!</p>
        <file-upload
                (notify)="onUploadFinished($event)"
                [multiple]="true"
                [url]="getProfilePicture()"></file-upload>
    </md-card-content>
</md-card>
```

`profile.component.ts`
```
import {UserService} from "../users/users.service";
import {Component} from "@angular/core";
import {User} from "../users/user";
import {Observable} from "rxjs";
import {Http} from "@angular/http";

@Component({
    selector: 'profile',
    templateUrl: './profile.component.html',
    styleUrls: ['./profile.component.css'],
    providers: [UserService]
})
export class ProfileComponent {
    private user: User = new User(undefined, undefined, undefined, undefined, undefined);
    private environment: Observable<any>;

    public url: string = "";

    constructor(private http: Http, private service: UserService) {
        this.environment = this.http
            .get('/environment/environment.json')
            .map(res => res.json());

        this.http
            .get('/environment/environment.json')
            .map(res => res.json())
            .subscribe((env: any) => {
                this.url = this.getFileuploaderUrl(env);
            });
    }

    ngOnInit() {
        this.getUser();
    }

    public getProfilePicture = () => {
        return this.url;
    };

    public getFileuploaderUrl = (env: any): string => {
        return 'http://' + env.host + ':' + env.fileuploader_port + '/battleapp/fileupload';
    };

    private getUser = () => {
        let _self = this;
        window['_keycloak']
            .loadUserInfo()
            .success(function (userInfo) {
                _self.service
                    .find(userInfo.sub)
                    .subscribe((responseUser: User) => {
                            _self.loadUser(responseUser);
                        },
                        error => console.log(error)
                    );
            });
    };

    public loadUser = (responseUser: User): void => {
        this.user = new User(
            responseUser.id,
            responseUser.nickname,
            responseUser.firstName,
            responseUser.lastName,
            undefined);
        this.service
            .getFullProfilePictureUrl(responseUser.profilePicture)
            .subscribe((fullUrlArray: string) => {
                    this.user.profilePicture = fullUrlArray;
                },
                error => console.log(error)
            );
    };

    public onUploadFinished = (profilePicture: string): void => {
        this.service
            .getFullProfilePictureUrl(profilePicture)
            .subscribe((fullUrlArray: string) => {
                    this.user.profilePicture = fullUrlArray;
                },
                error => console.log(error)
            );
    };
}
```

`app.module.ts`
```
...
import {FileUploadComponent} from "./fileupload/fileupload.component";
import {ProfileComponent} from "./profile/profile.component";
...
declarations: [
    AppComponent,
    UserComponent,
    ProfileComponent,
    NoContentComponent,
    FileUploadComponent
],
...
```

`environment.json`
```
{
    "host": "localhost",
    "command_port": 8083,
    "query_port": 8082,
    "fileuploader_port": 8084,
    "filedownloader_port": 8085
}
```

`user.ts`
```
export class User {
    public id: string;
    public nickname: string;
    public firstName: string;
    public lastName: string;
    public profilePicture: string;

    constructor(id: string,
                nickname: string,
                firstName: string,
                lastName: string,
                profilePicture: string) {
        this.id = id;
        this.nickname = nickname;
        this.firstName = firstName;
        this.lastName = lastName;
        this.profilePicture = profilePicture;
    }

    public toString = (): string => {
        return 'id: ' + this.id
            + ', nickname: ' + this.nickname
            + ', firstName: ' + this.firstName
            + ', lastName: ' + this.lastName
            + ', profilePicture: ' + this.profilePicture;
    }

}
```

`app.component.html`
```
...
<p>
    <a [routerLink]=" ['./profile'] " (click)="sidenav.toggle()">
        Profile
    </a>
</p>
...
```

`app.routes.ts`
```
...
import {ProfileComponent} from "./profile/profile.component";
...
export const ROUTES: Routes = [
    {path: '', component: ProfileComponent},
    {path: 'profile', component: ProfileComponent},
    {path: 'users', component: UserComponent},
    {path: '**', component: NoContentComponent},
];
...
```

We've to add the two new ports to the Kubernetes configmap of the `environment.json`:

```
{
  "host": "disruptor.ninja",
  "command_port": 30080,
  "query_port": 30081,
  "fileuploader_port": 30082,
  "filedownloader_port": 30031
}
```

And the same for the test configmap: 
```
{
  "host": "disruptor.ninja",
  "command_port": 31080,
  "query_port": 31081,
  "fileuploader_port": 31082,
  "filedownloader_port": 31031
}
```

