# Firebase Database in React

TODO: Write introdcution to this larger section.

## User Management with Firebase's Database in React

So far, only Firebase knows about your users. There is no way to retrieve the single user or a list of users for you. They are stored internally by Firebase to keep the authentication secured. That's good, because you are never involved in storing sensible data such as passwords. However, you can introduce the Firebase realtime database to keep track of the user entities yourself. And you should do it, because otherwise you never have a way to associate other domain entities (e.g. a todo item) created by your users to your users. Therefore, follow this section to store users in your realtime database in Firebase during the sign up process.

First of all, create a file for your Firebase realtime database API. It goes into the firebase folder next to your file for the authentication API.

{title="Command Line: src/firebase/",lang="text"}
~~~~~~~~
touch db.js
~~~~~~~~

In the file, you will implement the interface to your Firebase realtime database for the user entity. The file defines two functions: one to create a user and one to retrieve all users.

{title="src/firebase/db.js",lang=javascript}
~~~~~~~~
import { db } from './firebase';

// User API

export const doCreateUser = (id, username, email) =>
  db.ref(`users/${id}`).set({
    username,
    email,
  });

export const onceGetUsers = () =>
  db.ref('users').once('value');

// Other Entity APIs ...
~~~~~~~~

In the first asynchronous function, the user is created as an object with the username and email properties. Furthermore, it is stored on the `users/${id}` resource path. So whenever you would want to retrieve a specific user from the Firebase database, you could get the one user via its unique identifier and the entity resource path.

In the second asynchronous function, the users are retrieved from the general user's entity resource path. The function will return all users from the Firebase realtime database.

Note: In the future, you could consider to split up the file into multiple files for different domain entities (e.g. *db/user.js*, *db/todos.js*, ...) in one module (e.g. *db/* folder).

You might have noticed that the file imports a database object from the *src/firebase/firebase.js* file which isn't defined nor declared yet. Let's create it now.

{title="src/firebase/firebase.js",lang=javascript}
~~~~~~~~
import * as firebase from 'firebase';

...

# leanpub-start-insert
const db = firebase.database();
# leanpub-end-insert
const auth = firebase.auth();

export {
# leanpub-start-insert
  db,
# leanpub-end-insert
  auth,
};
~~~~~~~~

Here you can see again how Firebase separates its authentication and database API. You followed the same best practice in both *db.js* and *auth.js* files. Last but not least, don't forget to make the new functionalities from your Firebase database API available in your Firebase module's entry point.

{title="src/firebase/index.js",lang=javascript}
~~~~~~~~
import * as auth from './auth';
# leanpub-start-insert
import * as db from './db';
# leanpub-end-insert
import * as firebase from './firebase';

export {
  auth,
# leanpub-start-insert
  db,
# leanpub-end-insert
  firebase,
};
~~~~~~~~

Finally, you can use those functionalities in your React components to create and retrieve users. Let's start with the user creation. The best place to do it would be the SignUpForm component. It is the most natural place to create a user in the database after the sign up through the Firebase authentication API. You can simply add another API request to create a user when the user signed up successfully.

{title="src/components/SignUp.js",lang=javascript}
~~~~~~~~
import React, { Component } from 'react';
import {
  Link,
  withRouter,
} from 'react-router-dom';

# leanpub-start-insert
import { auth, db } from '../firebase';
# leanpub-end-insert
import * as routes from '../constants/routes';

...

class SignUpForm extends Component {

  ...

  onSubmit = (event) => {
    const {
      username,
      email,
      passwordOne,
    } = this.state;

    const {
      history,
    } = this.props;

    auth.doCreateUserWithEmailAndPassword(email, passwordOne)
      .then(authUser => {

# leanpub-start-insert
        // Create a user in your own accessible Firebase Database too
        db.doCreateUser(authUser.uid, username, email)
          .then(() => {
            this.setState(() => ({ ...INITIAL_STATE }));
            history.push(routes.HOME);
          })
          .catch(error => {
            this.setState(byPropKey('error', error));
          });
# leanpub-end-insert

      })
      .catch(error => {
        this.setState(byPropKey('error', error));
      });

    event.preventDefault();
  }

  ...
}
~~~~~~~~

Notice how all the previous business logic from the first then block moves into the second then block. The previous logic is only called after the second asynchronous API call resolves successfully. In addition, see how the `authUser` object's `uid` and the `username` property from the local state can be used to pass the necessary parameters to your Firebase database API.

Note: It is fine to store user information in your own database. However, you should make sure not to store the password or any other sensible data of the user on your own. Firebase already deals with the authentication and thus there is no need to store the password in your database. There are a bunch of steps necessary to secure such sensible data (e.g. encryption). It would be a security risk to do it on your own, so don't do it if someone else already handles it for you.

That's it for the user creation process. Now you are creating a user once a user signs up in your application. By now it is a lot of business logic in your component's lifecycle method and you could consider extracting the logic on your own to a separate place to keep the component lightweight. If you are going to add tests for your application after the tutorial, it is simpler to test the logic when it is extracted.

Next, only to verify that the user creation is working, you can retrieve all the users from the database in one of your other components. Since the HomePage component isn't of any use yet, you can do in this component to display your users stored in the realtime database of Firebase. The `componentDidMount()` lifecycle method is the perfect place to fetch users from your database API.

{title="src/components/Home.js",lang=javascript}
~~~~~~~~
import React, { Component } from 'react';

import withAuthorization from './withAuthorization';
import { db } from '../firebase';

class HomePage extends Component {
  constructor(props) {
    super(props);

    this.state = {
      users: null,
    };
  }

  componentDidMount() {
    db.onceGetUsers().then(snapshot =>
      this.setState(() => ({ users: snapshot.val() }))
    );
  }

  render() {
    return (
      <div>
        <h1>Home</h1>
        <p>The Home Page is accessible by every signed in user.</p>
      </div>
    );
  }
}

const authCondition = (authUser) => !!authUser;

export default withAuthorization(authCondition)(HomePage);
~~~~~~~~

Afterward, you can display a couple of properties of your list of users. Since the users are an object and not a list when they are retrieved from the Firebase database, you have to use the `Object.keys()` helper function to map over the keys in order to display them.

{title="src/components/Home.js",lang=javascript}
~~~~~~~~
...

class HomePage extends Component {
  ...

  render() {
# leanpub-start-insert
    const { users } = this.state;
# leanpub-end-insert

    return (
      <div>
        <h1>Home</h1>
        <p>The Home Page is accessible by every signed in user.</p>

# leanpub-start-insert
        { !!users && <UserList users={users} /> }
# leanpub-end-insert
      </div>
    );
  }
}

# leanpub-start-insert
const UserList = ({ users }) =>
  <div>
    <h2>List of Usernames of Users</h2>
    <p>(Saved on Sign Up in Firebase Database)</p>

    {Object.keys(users).map(key =>
      <div key={key}>{users[key].username}</div>
    )}
  </div>
# leanpub-end-insert

const authCondition = (authUser) => !!authUser;

export default withAuthorization(authCondition)(HomePage);
~~~~~~~~

That's it for the user entity management. You are in full control of your users now. It is possible to retrieve a user entity or a list of user entities. Furthermore, you can create a user in the realtime database. It is up to you to implement the other [CRUD operations](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) as well in order to update a user, to remove a user and to get a single user entity from the database.
