## Authentication in React, Firebase and MobX

The section dives into using MobX on top of React and Firebase for the state management. Basically, you will exchange React’s local state (user management: e.g. list of users on home page) and React’s context (session management: e.g. authenticated user object) with MobX.

Note: None of the Redux changes from the previous section are reflected here. We will start with a clean plate from one section before where we didn't use Redux but only plain React and Firebase.

This section is divided into two parts. The first part will setup MobX and all the necessary parts for it. You will implement the state layer separately from the view layer. The second part exchanges the current state layer (local state for users, the context for an authenticated user) with the MobX state layer. It is the part where the new state layer is connected to the view layer.

First of all, you should follow this [short guide to enable decorators in create-react-app](https://www.robinwieruch.de/create-react-app-mobx-decorators/). You can also take the path of not using decorators, to avoid the eject process, but this tutorial only reflects the usage **with decorators**.

You should have installed [mobx](https://mobx.js.org/) and [mobx-react](https://github.com/mobxjs/mobx-react) by now. Furthermore, you will have to install recompose on the command line to compose more than one higher order component on a component. You will enhance your component not only once, but multiple times by using the composing functionality of recompose.

{title="Command Line",lang="text"}
~~~~~~~~
npm install recompose
~~~~~~~~

Now let’s setup the MobX state layer. First of all, you need to implement the MobX stores. Therefore, create a folder and files for it.

{title="Command Line: src/",lang="text"}
~~~~~~~~
mkdir stores
cd stores
touch index.js sessionStore.js userStore.js
~~~~~~~~

You will have a store for the session state (e.g. authenticated user) and a store for the user state (e.g. list of users from the database). In addition, you will have an entry point file to the module to combine those stores as root store. First, the session store which manages simply the authUser object. Remember that the authenticated user represents our session in the application.

{title="src/stores/sessionStore.js",lang=javascript}
~~~~~~~~
import { observable, action } from 'mobx';

class SessionStore {
  @observable authUser = null;

  constructor(rootStore) {
    this.rootStore = rootStore;
  }

  @action setAuthUser = authUser => {
    this.authUser = authUser;
  }
}

export default SessionStore;
~~~~~~~~

Second, the user store which deals with the list of users from the Firebase realtime database:

{title="src/stores/userStore.js",lang=javascript}
~~~~~~~~
import { observable, action } from 'mobx';

class UserStore {
  @observable users = [];

  constructor(rootStore) {
    this.rootStore = rootStore;
  }

  @action setUsers = users => {
    this.users = users;
  }
}

export default UserStore;
~~~~~~~~

Finally, combine both stores in a root store. This can be used to make the stores communicate to each other, but also to provide a way to import only one store (root store) to have access to all of its combined stores later on.

{title="src/stores/index.js",lang=javascript}
~~~~~~~~
import { configure } from 'mobx';

import SessionStore from './sessionStore';
import UserStore from './userStore';

configure({ enforceActions: true });

class RootStore {
  constructor() {
    this.sessionStore = new SessionStore(this);
    this.userStore = new UserStore(this);
  }
}

const rootStore = new RootStore();

export default rootStore;
~~~~~~~~

The MobX setup is done. Now, you can connect your state layer with your view layer. All the MobX stores can be provided to the component hierarchy by using MobX's bridging Provider component. It's not the Provider component you have created before on your own by using React's context API, but this time a Provider component from a MobX library passing down the all stores instead of the authenticated user. The stores (e.g. user store) will keep track of the authenticated user for you. We will use an object spread operator here to pass all combined stores to the Provider component. Keep in mind that this way the root store isn't available in the component hierarchy, but only the combined stores. That's sufficient for the tutorial, but if you want to have access to the root store as well, you need to pass it to the Provider component too.

{title="src/index.js",lang=javascript}
~~~~~~~~
import React from 'react';
import ReactDOM from 'react-dom';
# leanpub-start-insert
import { Provider } from 'mobx-react';
# leanpub-end-insert
import './index.css';
import App from './components/App';
# leanpub-start-insert
import store from './stores';
# leanpub-end-insert
import registerServiceWorker from './registerServiceWorker';

ReactDOM.render(
# leanpub-start-insert
  <Provider { ...store }>
# leanpub-end-insert
    <App />
# leanpub-start-insert
  </Provider>,
# leanpub-end-insert
  document.getElementById('root')
);

registerServiceWorker();
~~~~~~~~

Now comes the refactoring part where you exchange a part of the React state layer with the MobX state layer. You will replace React’s context entirely in this part by passing the state down the component tree with the MobX stores.

Let’s start with the simpler part of the refactoring: the user state layer. On the home page, you will use MobX's bridging library to inject the needed store via `inject()` to the component. In addition, you will make the component observable via `observer()` from the bridging library for MobX reactions. If the MobX state changes, the component will react to it. In addition, the user store is used to store the users coming from the Firebase realtime database.

{title="src/components/Home.js",lang=javascript}
~~~~~~~~
import React, { Component } from 'react';
# leanpub-start-insert
import { inject, observer } from 'mobx-react';
import { compose } from 'recompose';
# leanpub-end-insert

import withAuthorization from './withAuthorization';
import { db } from '../firebase';

class HomePage extends Component {
  componentDidMount() {
# leanpub-start-insert
    const { userStore } = this.props;

    db.onceGetUsers().then(snapshot =>
      userStore.setUsers(snapshot.val())
    );
# leanpub-end-insert
  }

  render() {
# leanpub-start-insert
    const { users } = this.props.userStore;
# leanpub-end-insert

    return (
      <div>
        <h1>Home</h1>
        <p>The Home Page is accessible by every signed in user.</p>

        { !!users && <UserList users={users} /> }
      </div>
    );
  }
}

...

const authCondition = (authUser) => !!authUser;

# leanpub-start-insert
export default compose(
  withAuthorization(authCondition),
  inject('userStore'),
  observer
)(HomePage);
# leanpub-end-insert
~~~~~~~~

Now the users are managed with MobX rather than in React’s local state. You have connected the state from MobX with the view layer.

What about the session state layer which should be handled by the session store? Essentially you will refactor it the same way as the user state layer before. You will replace your own Provider and Consumer components, where the authenticated user was reached through all components by using React's context API, where the authenticated user will be stored in the session store. Thus, instead of passing the authenticated user object down via React’s context, you pass it down via the MobX's session store by providing the store in a parent component. You already provided the stores in a parent component by using the custom Provider component from mobx-react. Afterward, you can connect all the components that care about the authenticated user (e.g. Navigation, Account) to it.

The most important component to store the authenticated user object in the MobX session store rather than in React’s context is the `withAuthentication()` higher order component. We can refactor it to use the MobX session store instead of React’s context by connecting it to the state layer.

{title="src/components/withAuthentication.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { inject } from 'mobx-react';
# leanpub-end-insert

import { firebase } from '../firebase';

const withAuthentication = (Component) => {
  class WithAuthentication extends React.Component {
    componentDidMount() {
# leanpub-start-insert
      const { sessionStore } = this.props;
# leanpub-end-insert

      firebase.auth.onAuthStateChanged(authUser => {
        authUser
# leanpub-start-insert
          ? sessionStore.setAuthUser(authUser)
          : sessionStore.setAuthUser(null);
# leanpub-end-insert
      });
    }

    render() {
      return (
        <Component />
      );
    }
  }

# leanpub-start-insert
  return inject('sessionStore')(WithAuthentication);
# leanpub-end-insert
}

export default withAuthentication;
~~~~~~~~

Now you have the authenticated user available in the MobX session store. As consequence all components which rely on the authenticated user in React’s context need to be refactored now. It’s similar to the Home component which uses the list of users from the MobX user store instead of React’s local state.

In the Navigation component, the authenticated user is used to display different routing options. Thus you will need to refactor the component to inject the MobX session store instead.

{title="src/components/Navigation.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { inject, observer } from 'mobx-react';
import { compose } from 'recompose';
# leanpub-end-insert
import { Link } from 'react-router-dom';

import SignOutButton from './SignOut';
import * as routes from '../constants/routes';

# leanpub-start-insert
const Navigation = ({ sessionStore }) =>
# leanpub-end-insert
  <div>
# leanpub-start-insert
    { sessionStore.authUser
# leanpub-end-insert
        ? <NavigationAuth />
        : <NavigationNonAuth />
    }
  </div>

const NavigationAuth = () =>
  <ul>
    <li><Link to={routes.LANDING}>Landing</Link></li>
    <li><Link to={routes.HOME}>Home</Link></li>
    <li><Link to={routes.ACCOUNT}>Account</Link></li>
    <li><SignOutButton /></li>
  </ul>

const NavigationNonAuth = () =>
  <ul>
    <li><Link to={routes.LANDING}>Landing</Link></li>
    <li><Link to={routes.SIGN_IN}>Sign In</Link></li>
  </ul>

# leanpub-start-insert
export default compose(
  inject('sessionStore'),
  observer
)(Navigation);
# leanpub-end-insert
~~~~~~~~

In the Account component, the authenticated user is used to display the email address of the user. There you need to inject the MobX session store as well.

{title="src/components/Account.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { inject, observer } from 'mobx-react';
import { compose } from 'recompose';
# leanpub-end-insert

import { PasswordForgetForm } from './PasswordForget';
import PasswordChangeForm from './PasswordChange';
import withAuthorization from './withAuthorization';

# leanpub-start-insert
const AccountPage = ({ sessionStore }) =>
# leanpub-end-insert
  <div>
# leanpub-start-insert
    <h1>Account: {sessionStore.authUser.email}</h1>
# leanpub-end-insert
    <PasswordForgetForm />
    <PasswordChangeForm />
  </div>

const authCondition = (authUser) => !!authUser;

# leanpub-start-insert
export default compose(
  withAuthorization(authCondition),
  inject('sessionStore'),
  observer
)(AccountPage);
# leanpub-end-insert
~~~~~~~~

Furthermore, don’t forget that the authorization higher order component used the authenticated user from React’s context as well for the fallback conditional rendering. You have to refactor it too.

{title="src/components/withAuthorization.js",lang=javascript}
~~~~~~~~
import React from 'react';
import { withRouter } from 'react-router-dom';
# leanpub-start-insert
import { inject, observer } from 'mobx-react';
import { compose } from 'recompose';
# leanpub-end-insert

import { firebase } from '../firebase';
import * as routes from '../constants/routes';

const withAuthorization = (condition) => (Component) => {
  class WithAuthorization extends React.Component {
    componentDidMount() {
      firebase.auth.onAuthStateChanged(authUser => {
        if (!condition(authUser)) {
          this.props.history.push(routes.SIGN_IN);
        }
      });
    }

    render() {
# leanpub-start-insert
      return this.props.sessionStore.authUser ? <Component /> : null;
# leanpub-end-insert
    }
  }

# leanpub-start-insert
  return compose(
    withRouter,
    inject('sessionStore'),
    observer
  )(WithAuthorization);
# leanpub-end-insert
}

export default withAuthorization;
~~~~~~~~

That's it. In this section, you have introduced MobX as state management library to manage your session and user state. Instead of relying on React's context API for the authenticated user object and React's local state for the list of users from the Firebase database, you are storing these objects in the MobX stores. You can find the project with a slightly different folder structure in this [GitHub repository](https://github.com/rwieruch/react-mobx-firebase-authentication).