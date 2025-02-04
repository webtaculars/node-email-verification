# node email verification
[![NPM](https://nodei.co/npm/email-verification.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/email-verification/)

Verify user signup with Node and MongoDB!

The way this works is as follows:

- temporary user is created with a randomly generated URL assigned to it and then saved to a MongoDB collection
- email is sent to the email address the user signed up with
- when the URL is accessed, the user's data is transferred to the real collection

A temporary user document has a TTL of 24 hours by default, but this (as well as many other things) can be configured. See the options section for more details. It is also possible to resend the verification email if needed.

## Installation
via npm:

```
npm install email-verification
```

## Examples
This little guide of sorts assumes you have a directory structure like so:

```
app/
-- userModel.js
-- tempUserModel.js
node_modules/
server.js
```

All of the code in this section takes place in server.js. Note that `mongoose` has to be passed as an argument when requiring the module:

```javascript
var User = require('./app/userModel'),
    mongoose = require('mongoose'),
    nev = require('email-verification')(mongoose);
mongoose.connect('mongodb://localhost/YOUR_DB');
```

Before doing anything, make sure to configure the options (see the section below for more extensive detail on this):

```javascript
nev.configure({
    verificationURL: 'http://myawesomewebsite.com/email-verification/${URL}',
    persistentUserModel: User,
    tempUserCollection: 'myawesomewebsite_tempusers',

    transportOptions: {
        service: 'Gmail',
        auth: {
            user: 'myawesomeemail@gmail.com',
            pass: 'mysupersecretpassword'
        }
    },
    verifyMailOptions: {
        from: 'Do Not Reply <myawesomeemail_do_not_reply@gmail.com>',
        subject: 'Please confirm account',
        html: 'Click the following link to confirm your account:</p><p>${URL}</p>',
        text: 'Please confirm your account by clicking the following link: ${URL}'
    }
});
```

Any options not included in the object you pass will take on the default value specified in the section below. Calling `configure` multiple times with new options will simply change the previously defined options.

To create a temporary user model, you can either generate it using a built-in function, or you can predefine it in a separate file. If you are pre-defining it, it must be IDENTICAL to the user model with an extra field for the URL; the default one is `GENERATED_VERIFYING_URL: String`.

```javascript
// configuration options go here...

// generating the model, pass the User model defined earlier
nev.generateTempUserModel(User);

// using a predefined file
var TempUser = require('./app/tempUserModel');
nev.configure({
    tempUserModel: TempUser 
});
```

Then, create an instance of the User model, and then pass it as well as a custom callback to `createTempUser`. The callback should take two parameters: an error if any occured, and one for the new temporary user that's created. If the user already exists in the temporary *or* permanent collection, or if there are any errors, then this parameter will be `null`.

Inside the `createTempUser` callback, make a call to the `sendVerificationEmail` function, which takes three parameters: the user's email, the URL assigned to the user, and a callback. This callback takes two parameters: an error if any occured, and the information returned by Nodemailer.

```javascript
// get the credentials from request parameters or something
var email = "...",
    password = "...";

var newUser = User({
    email: email,
    password: password
});

nev.createTempUser(newUser, function(err, newTempUser) {
    // some sort of error
    if (err)
        // handle error...

    // a new user
    if (newTempUser) {
        var URL = newTempUser[nev.options.URLFieldName];
        nev.sendVerificationEmail(email, URL, function(err, info) {
            if (err)
                // handle error...
            
            // flash message of success
        });

    // user already exists in our temporary OR permanent collection
    } else {
        // flash message of failure...
    }
});
```

An email will be sent to the email address that the user signed up with. If you are interested in hashing the password (which you probably should be), all you need to do is set the option `hashingFunction` to a function that takes the parameters `password, tempUserData, insertTempUser, callback` and returns `insertTempUser(hash, tempUserData, callback)`, e.g.:

```javascript
// sync version of hashing function
var myHasher = function(password, tempUserData, insertTempUser, callback) {
  var hash = bcrypt.hashSync(password, bcrypt.genSaltSync(8), null);
  return insertTempUser(hash, tempUserData, callback);
};

// async version of hashing function
myHasher = function(password, tempUserData, insertTempUser, callback) {
  bcrypt.genSalt(8, function(err, salt) {
    bcrypt.hash(password, salt, function(err, hash) {
      return insertTempUser(hash, tempUserData, callback);
    });
  });
};
```

To move a user from the temporary storage to 'persistent' storage (e.g. when they actually access the URL we sent them), we call `confirmTempUser`, which takes the URL as well as a callback with two parameters: an error, and the instance of the User model (or `null` if there are any errors, or if the user wasn't found - i.e. their data expired).

If you want to send a confirmation email, then inside the `confirmTempUser` callback,  make a call to the `sendConfirmationEmail` function, which takes two parameters: the user's email and a callback. This callback takes two parameters: an error if any occured, and the information returned by Nodemailer.

```javascript
var url = '...';
nev.confirmTempUser(url, function(err, user) {
    if (err)
        // handle error...

    // user was found!
    if (user) {
        // optional
        nev.sendConfirmationEmail(user['email_field_name'], function(err, info) {
            // redirect to their profile...
        });
    }

    // user's data probably expired...
    else
        // redirect to sign-up
});
```

If you want the user to be able to request another verification email, simply call `resendVerificationEmail`, which takes the user's email address and a callback with two parameters: an error, and a boolean representing whether or not the user was found.

```javascript
var email = '...';
nev.resendVerificationEmail(email, function(err, userFound) {
    if (err)
        // handle error...

    if (userFound)
        // email has been sent
    else
        // flash message of failure...
});
```

To see a fully functioning example that uses Express as the backend, check out the [**examples section**](https://github.com/SaintDako/node-email-verification/tree/master/examples/express).

**NEV supports Bluebird's PromisifyAll!** Check out the examples section for that too.

## API
### `configure(optionsToConfigure, callback(err, options))`
Changes the default configuration by passing an object of options to configure (`optionsToConfigure`); see the section below for a list of all options. `options` will be the result of the configuration, with the default values specified below if they were not given. If there are no errors, `err` is `null`.

### `generateTempUserModel(UserModel, callback(err, tempUserModel))`
Generates a Mongoose Model for the temporary user based off of `UserModel`, the persistent user model. The temporary model is essentially a duplicate of the persistent model except that it has the field `{GENERATED_VERIFYING_URL: String}` for the randomly generated URL by default (the field name can be changed in the options). If the persistent model has the field `createdAt`, then an expiration time (`expires`) is added to it with a default value of 24 hours; otherwise, the field is created as such:

```javascript
{
    ...
    createdAt: {
        type: Date,
        expires: 86400,
        default: Date.now
    }
    ...
}
```

`tempUserModel` is the Mongoose model that is created for the temporary user. If there are no errors, `err` is `null`.

Note that `createdAt` will not be transferred to persistent storage (yet?).

### `createTempUser(user, callback(err, newTempUser))`
Attempts to create an instance of a temporary user model based off of an instance of a persistent user, `user`, and add it to the temporary collection. `newTempUser` is the temporary user instance if the user doesn't exist in the temporary collection, or `null` otherwise. If there are no errors, `err` is `null`.

If a temporary user model hasn't yet been defined (generated or otherwise), `err` will NOT be `null`.

### `sendVerificationEmail(email, url, callback(err, info))`
Sends a verification email to to the email provided, with a link to the URL to verify the account. If sending the email succeeds, then `err` will be `null` and `info` will be some value. See [Nodemailer's documentation](https://github.com/andris9/Nodemailer#sending-mail) for information.

### `confirmTempUser(url, callback(err, newPersistentUser))`
Transfers a temporary user (found by `url`) from the temporary collection to the persistent collection and removes the URL assigned with the user. `newPersistentUser` is the persistent user instance if the user has been successfully transferred (i.e. the user accessed URL before expiration) and `null` otherwise; this can be used for redirection and what not. If there are no errors, `err` is `null`.

### `sendConfirmationEmail(email, callback(err, info))`
Sends a confirmation email to to the email provided. If sending the email succeeds, then `err` will be `null` and `info` will be some value. See [Nodemailer's documentation](https://github.com/andris9/Nodemailer#sending-mail) for information.

### `resendVerificationEmail(email, callback(err, userFound))`
Resends the verification email to a user, given their email. `userFound` is `true` if the user has been found in the temporary collection (i.e. their data hasn't expired yet) and `false` otherwise. If there are no errors, `err` is `null`.


### Options
Here are the default options:

```javascript
var options = {
    verificationURL: 'http://example.com/email-verification/${URL}',
    URLLength: 48,

    // mongo-stuff
    persistentUserModel: null,
    tempUserModel: null,
    tempUserCollection: 'temporary_users',
    emailFieldName: 'email',
    passwordFieldName: 'password',
    URLFieldName: 'GENERATED_VERIFYING_URL',
    expirationTime: 86400,

    // emailing options
    transportOptions: {
        service: 'Gmail',
        auth: {
            user: 'user@gmail.com',
            pass: 'password'
        }
    },
    verifyMailOptions: {
        from: 'Do Not Reply <user@gmail.com>',
        subject: 'Confirm your account',
        html: '<p>Please verify your account by clicking <a href="${URL}">this link</a>. If you are unable to do so, copy and ' +
                'paste the following link into your browser:</p><p>${URL}</p>',
        text: 'Please verify your account by clicking the following link, or by copying and pasting it into your browser: ${URL}'
    },
    sendConfirmationEmail: true,
    confirmMailOptions: {
        from: 'Do Not Reply <user@gmail.com>',
        subject: 'Successfully verified!',
        html: '<p>Your account has been successfully verified.</p>',
        text: 'Your account has been successfully verified.'
    },

    hashingFunction: null,
}
```

- **verificationURL**: the URL for the user to click to verify their account. `${URL}` determines where the randomly generated part of the URL goes, and is needed. Required.
- **URLLength**: the length of the randomly-generated string. Must be a positive integer. Required.

- **persistentUserModel**: the Mongoose Model for the persistent user.
- **tempUserModel**: the Mongoose Model for the temporary user. you can generate the model by using `generateTempUserModel` and passing it the persistent User model you have defined, or you can define your own model in a separate file and pass it as an option in `configure` instead.
- **tempUserCollection**: the name of the MongoDB collection for temporary users.
- **emailFieldName**: the field name for the user's email. If the field is nested within another object(s), use dot notation to access it, e.g. `{local: {email: ...}}` would use `'local.email'`. Required.
- **passwordFieldName**: the field name for the user's password. If the field is nested within another object(s), use dot notation to access it (see above). Required.
- **URLFieldName**: the field name for the randomly-generated URL. Required.
- **expirationTime**: the amount of time that the temporary user will be kept in collection, measured in seconds. Must be a positive integer. Required.

- **transportOptions**: the options that will be passed to `nodemailer.createTransport`.
- **verifyMailOptions**: the options that will be passed to `nodemailer.createTransport({...}).sendMail` when sending an email for verification. You must include `${URL}` somewhere in the `html` and/or `text` fields to put the URL in these strings.
- **sendConfirmationEmail**: send an email upon the user verifiying their account to notify them of verification.
- **confirmMailOptions**: the options that will be passed to `nodemailer.createTransport({...}).sendMail` when sending an email to notify the user that their account has been verified. You must include `${URL}` somewhere in the `html` and/or `text` fields to put the URL in these strings.

- **hashingFunction**: the function that hashes passwords. Must take four parameters `password, tempUserData, insertTempUser, callback` and return `insertTempUser(hash, tempUserData, callback)`.

### Developer-Related Stuff
To beautify the code:

```
npm run format:main
npm run format:examples
npm run format:test
npm run format  # runs all
```

To lint the code (will error if there are any warnings):

```
npm run lint:main
npm run lint:examples
npm run lint:test
npm run lint  # runs all
```

To test:

```
npm test
```

### TODO
- **development**: add a task runner WE NEED TESTS
- **development**: throw more errors
- *nice to have*: working examples with Sails and HapiJS (maybe Koa and Total as well?)

### Acknowledgements
thanks to [Frank Cash](https://github.com/frankcash) for looking over my code.

### license
ISC
