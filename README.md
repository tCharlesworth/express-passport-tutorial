# express-passport-tutorial

Passport tutorial

This tutorial expects a simple understanding of node, express, and npm. It is designed to
act as a basis for adding passport authentication to your existing servers and does not
include a tutorial for settting up the server.

For basic file setup all we need is access to the file where our express server runs, which
I will refer to as server.js, and a file for a passportCtrl.js file where our passport
logic will exist. Once we have these files ready we can go ahead and install passport and
passport-local using npm.

	npm install passport passport-local --save
 
Now lets open up our passportCtrl file and configure passport. We will need to require
passport and passport-local. With passport local all we need is the Strategy so we will
just grab the strategy. The next thing we need to do is setup our strategy. A strategy 
is how we tell passport how to authenticate. Using the local strategy we will provide
a function that passport will call whenever we want to authenticate locally. Passport
will provide us with 3 variables, an email, password, and a callback function we use
when we are finished determining whether or not the credentials are valid. Here is the
basic code.

	var Passport = require('passport');
	var LocalStrategy = require('passport-local').Strategy;
	Passport.use(new LocalStrategy(function(email, password, done) {
	}));

As you can see we are telling passport what a Local Strategy looks like when it needs to use
one. Inside this function is where we will write code to determine if the email and password
are correct. Once we decide, then we call done with specific parameters to let passport
know if the credentials are correct. For this example we will just see if email and password
match a specific pre-determined email/password rather than trying to setup a database call
to verify authentication, this way we can keep this simple and short.

Here are the different parameters for the done function, first of all the first parameter
we pass back will always be null. All the online tutorials I have seen don't say why, neither
do the passport docs, but they always return null as the first parameter. The second parameter
is whether or not the user is valid. If they are valid, return an object that represents
the user, otherwise return false as the second parameter. The third parameter is a where we
can provide an object to send customized messages as to why the authentication failed. Here is
what your passportCtrl file could look like:

	var Passport = require('passport');
	var LocalStrategy = require('passport-local').Strategy;
	Passport.use(new LocalStrategy(function(email, password, done) {
		if(email === 'admin@admin.com') {
			if(password === 'adminPassword') {
				//Valid Credentials
				return done(null, {username: 'admin', email: email});
			} else {
				//Invalid Password
				return done(null, false, {message: 'Invalid password'});
			}
		} else {
			//Invalid Email
			return done(null, false, {message: 'Invalid email.'});
		}	
	}));

Ideally there would be a call to a database to see if the user was valid but remember this is just
a simple example so we will only test this one user scenario. Next step is our serialization and
deserialization methods. These methods are called when a user logs in or logs out. You can
use these methods to customize the user object you are sending back. For example if you did not
want to send back the password you could write the javascript to do so in these methods. Here
is what they look like in this example: 


	//Serialization
	passport.serializeUser(function(user, done) {
		done(null, user);
	})
	passport.deserializeUser(function(obj, done) {
		done(null, obj);
	})

Here is that done function again. We pass null, and then the new user object after we have modified it
to our liking. Here we are just passing it on because we are not worried about security yet. 

Next lets setup a logout function through module.exports. Passport uses a logout function that is
attatched to the request object that we use to logout. It will remove the user object from the
requests and terminate any sessions. 

	module.exports = {
		logout: function(req, res) {
			req.logout();
			res.send();
		}
	}

The last function we will add is a function that we will use as a policy to force users to login before
they access certain server functions. We will call it isAuthenticated. All it will do is check the
isAuthenticated function that passport has given us on the request object. Now our full passport file should
look like this:

	var passport = require('passport');
	var LocalStrategy = require('passport-local').Strategy;
	passport.use(new LocalStrategy(function(email, password, done) {
		if(email === 'admin@admin.com') {
			if(password === 'adminPassword') {
				//Valid Credentials
				return done(null, {username: 'admin', email: email});
			} else {
				//Invalid Password
				return done(null, false, {message: 'Invalid password'});
			}
		} else {
			//Invalid Email
			return done(null, false, {message: 'Invalid email.'});
		}	
	}));
	//Serialization
	passport.serializeUser(function(user, done) {
		done(null, user);
	})
	
	passport.deserializeUser(function(obj, done) {
		done(null, obj);
	})
	//Exports
	module.exports = {
		logout: function(req, res) {
			req.logout();
			res.send();
		},
		isAuthenticated: function (req, res, next) {
			if(!req.isAuthenticated()) {
				//Not Logged In!
				return res.sendStatus(401);
			}
			return next();
		}
	}

So now we just need to plug this in to our server! For the server we will need one more node package
named 'express-session'. Open node and install 'express-session' as a dependency for our project.

Now we need to require 3 things in our server file. Require passport, express-session, and the passport
ctrl we just made. Then we need to tell express to use our sessions and passport. the code you need looks
like this: 

	var Passport = require('passport');
	var Session = require('express-session');
	app.use(Session({secret: 'mySecret'}));
	app.use(Passport.initialize());
	app.use(Passport.session());

The last step is to plug in our endpoints! A login endpoint  simply will call Passport and tell it to authenticate.
To authenticate all you need is a string telling it what strategy to use, in our case local will be what we pass
it. 

If you want to enforce a policy, just call the isAuthenticated function on our passportCtrl before an endpoint
to block users who are not authenticated. The code below is a simple server code and an example of enforcing
this policy can be found on the '/getProducts' endpoint. Here is an example of what your server could look
like:

	//Dependencies
	var Express = require('express');
	var Passport = require('passport');
	var Session = require('express-session');
	var passportCtrl = require('./passportCtrl');
	///Setup
	var app = Express();
	var port = 8080;
	app.use(Session({secret: 'mySecret'}));
	app.use(Passport.initialize());
	app.use(Passport.session());
	//Endpoints
	app.post('/login', Passport.authenticate('local'));
	app.get('/logout', passportCtrl.logout);
	app.get('/products', passportCtrl.isAuthenticated, function(req, res) {
		return 'Products';
	})
	app.listen(port, function() {
		console.log('Server listening on port: '+port);
	});
