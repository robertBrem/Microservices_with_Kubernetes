# Create an Angular 2 frontend

## Install NodeJS and NPM
We're going to use NPM as our package manager. To install NodeJS and NPM execute the
following command:

```
curl -sL https://deb.nodesource.com/setup_7.x | sudo -E bash -
sudo apt-get install -y nodejs
```

## Setup the Angular 2 project
We are using a bootstrap project to get started with Angular 2:

```
git clone --depth 1 https://github.com/angularclass/angular2-webpack-starter.git
cd angular2-webpack-starter
npm install
npm run server:dev:hmr
```

Now you can open the frontend on this url:
```
http://localhost:3000
```

