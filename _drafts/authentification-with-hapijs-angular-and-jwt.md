---
layout: post
title: "Authentification with Hapi.js, Angular and JWT"
comment: true
---

## Scaffolding the application

{% highlight bash %}
npm init
npm install --save hapi inert angular
{% endhighlight %}

### Server

{% highlight javascript %}
// index.js
const Hapi = require('hapi');
const server = new Hapi.Server();
server.connection({
	port: process.env.port || 3000
});

server.register([ require('inert') ], (err) => {
	if(err) throw err;

	server.route({
		path:	'/{param*}',
		method:	'GET',
		handler: {
			directory: {
				path: 'public'
			}
		}
	});

	server.route({
		path:	'/vendors/{param*}',
		method: 'GET',
		handler: {
			directory: {
				path: 'node_modules'
			}
		}
	});

	server.start((err) => {
		if(err) throw err;
		console.log('server started');
	});
});
{% endhighlight %}

### Client

{% highlight javascript %}
// public/app.js
const app = angular.module('my-secured-app', []);

app.controller('MainCtrl', [function() {
	this.name = 'Guest';
}]);
{% endhighlight %}

{% highlight html %}
<!-- public/index.html -->
<html ng-app="my-secured-app">
	<head>...</head>
	<body ng-controller="MainCtrl as ctrl">
		<h1>Hello \{\{ ctrl.name \}\}!</h1>
		<script src="vendors/angular/angular.min.js"></script>
		<script src="app.js"></script>
	</body>
</html>
{% endhighlight %}

## Adding a login page / ngRoute

{% highlight bash %}
npm install --save angular-route
{% endhighlight %}

{% highlight html %}
<!-- public/index.html -->
<html ng-app="my-secured-app">
	<head></head>
	<body ng-controller="MainCtrl as ctrl">
		<ng-view></ng-view>
		<script src="vendors/angular/angular.min.js"></script>
		<script src="vendors/angular-route/angular-route.min.js"></script>
		<script src="app.js"></script>
	</body>
</html>
{% endhighlight %}

{% highlight html %}
<!-- public/views/login.html -->
<input placeholder="username" type="text" ng-model="username" />
<input placeholder="password" type="password" ng-model="password">
<button type="submit" ng-click="ctrl.login()">Submit</button>
{% endhighlight %}

{% highlight html %}
<!-- public/views/home.html -->
<h1>Hello \{\{ ctrl.name \}\}!</h1>
{% endhighlight %}

{% highlight javascript %}
// public/app.js
...
app.controller('LoginCtrl', ['LoginService', '$location', function(LoginService, $location) {
	var onSuccess = (err, result) => {
		$location.path('/');
	};
	var onFailure = (err, result) => {
		alert('could not login');
	};
	this.login = (username, password) => {
		LoginService.login(username, password).then(onSuccess, onFailure);
	};
}]);
...
{% endhighlight %}

{% highlight javascript %}
// public/app.js
...
app.service('LoginService', ['$http', function($http) {
	this.login = (username, password) => {
		console.log('trying to login');
		return $http.post('/api/login', {
			username: username,
			password: password
		});
	};
}]);
...
{% endhighlight %}

{% highlight javascript %}
// public/app.js
...
app.config(['$routeProvider', function($routeProvider) {
	$routeProvider
	.when('/', {
		templateUrl: 'views/home.html',
		controller: 'MainCtrl',
		controllerAs: 'ctrl'
	})
	.when('/login', {
		templateUrl: 'views/login.html',
		controller: 'LoginCtrl',
		controllerAs: 'ctrl'
	});
}]);
{% endhighlight %}

## Wiring up the backend

{% highlight javascript %}
// index.js
...
server.register([ require('inert') ], (err) => {
	...
	server.route({
		path: '/api/login',
		method: 'POST',
		handler: (req, res) => {
			console.log(req.payload);
			return res('success');
		}
	});

	server.start(...);
});
{% endhighlight %}