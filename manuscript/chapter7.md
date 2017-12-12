# Authentication in React, Firebase and Redux

The section dives into using Redux on top of React and Firebase for the state management. Basically, you will exchange React's local state (user management: e.g. list of users on home page) and React's context (session management: e.g. authenticated user object) with Redux. The section builds on top of the last section which concluded the authentication and authorization in plain React and Firebase.

This section is divided into two parts. The first part will setup Redux and all the necessary parts for it. You will implement the state layer separately from the view layer. The second part exchanges the current state layer (local state for users, the context for an authenticated user) with the Redux state layer. It is the part where the new state layer is connected to the view layer.

First of all, you should install [redux](https://redux.js.org/) and [react-redux](https://github.com/reactjs/react-redux) on the command line.

{title="Command Line",lang="text"}
~~~~~~~~
npm install redux react-redux
~~~~~~~~

Furthermore, you will have to install [recompose](https://github.com/acdlite/recompose) on the command line to compose more than one higher order component on a component. You will enhance your component not only once, but multiple times by using the composing functionality of recompose.

{title="Command Line",lang="text"}
~~~~~~~~
npm install recompose
~~~~~~~~

Now let's setup the Redux state layer. First of all, you need the Redux store implementation. Therefore, create a folder and file for it.

{title="Command Line: src/",lang="text"}
~~~~~~~~
mkdir store
cd store
touch index.js
~~~~~~~~

Second, implement the store as singleton instance, because there should be only one Redux store. The store creation takes a root reducer which isn't defined yet. You will do it in the next step.

{title="src/store/index.js",lang=javascript}
~~~~~~~~
import { createStore } from 'redux';
import rootReducer from '../reducers';

const store = createStore(rootReducer);

export default store;
~~~~~~~~

Third, create a dedicated module for the reducers. You will have a reducer for the session state (e.g. authenticated user) and a reducer for the user state (e.g. list of users from the database). In addition, you will have an entry point file to the module to combine those reducers as root reducer to pass it to the Redux store which you already did in the previous step.

{title="Command Line: src/",lang="text"}
~~~~~~~~
mkdir reducers
cd reducers
touch index.js session.js user.js
~~~~~~~~

Now, implement the two reducers. First, the session reducer which manages simply the `authUser` object. Remember that the authenticated user represents our session in the application.

{title="src/reducers/session.js",lang=javascript}
~~~~~~~~
const INITIAL_STATE = {
  authUser: null,
};

const applySetAuthUser = (state, action) => ({
  ...state,
  authUser: action.authUser
});

function sessionReducer(state = INITIAL_STATE, action) {
  switch(action.type) {
    case 'AUTH_USER_SET' : {
      return applySetAuthUser(state, action);
    }
    default : return state;
  }
}

export default sessionReducer;
~~~~~~~~

Second, the user reducer which deals with the list of users from the Firebase realtime database:

{title="src/reducers/user.js",lang=javascript}
~~~~~~~~
const INITIAL_STATE = {
  users: {},
};

const applySetUsers = (state, action) => ({
  ...state,
  users: action.users
});

function userReducer(state = INITIAL_STATE, action) {
  switch(action.type) {
    case 'USERS_SET' : {
      return applySetUsers(state, action);
    }
    default : return state;
  }
}

export default userReducer;
~~~~~~~~

Finally, combine both reducers in a root reducer to make it accessible for the store creation.

{title="src/reducers/index.js",lang=javascript}
~~~~~~~~
import { combineReducers } from 'redux';
import sessionReducer from './session';
import userReducer from './user';

const rootReducer = combineReducers({
  sessionState: sessionReducer,
  userState: userReducer,
});

export default rootReducer;
~~~~~~~~

You have already passed the root reducer with all its reducers to the Redux store creation. The Redux setup is done. Now, you can connect your state layer with your view layer. The Redux store can be provided to the component hierarchy by using Redux's bridging Provider component.

{title="src/index.js",lang=javascript}
~~~~~~~~
import React from 'react';
import ReactDOM from 'react-dom';
# leanpub-start-insert
import { Provider } from 'react-redux';
# leanpub-end-insert
import './index.css';
import App from './components/App';
# leanpub-start-insert
import store from './store';
# leanpub-end-insert
import registerServiceWorker from './registerServiceWorker';

ReactDOM.render(
# leanpub-start-insert
  <Provider store={store}>
# leanpub-end-insert
    <App />
# leanpub-start-insert
  </Provider>,
# leanpub-end-insert
  document.getElementById('root')
);

registerServiceWorker();
~~~~~~~~

Now comes the refactoring part where you exchange a part of the React state layer with the Redux state layer. You will replace React's context entirely in this part by passing the state down the component tree with the Redux store.

Let's start with the simpler part of the refactoring: the user state layer. On the home page, you will use Redux's bridging library to connect the state via `mapStateToProps()` to the component. In addition, you will connect actions to your component as well via `mapDispatchToProps()` to store the users coming from the Firebase realtime database to your Redux store.

{title="src/components/Home.js",lang=javascript}
~~~~~~~~
import React, { Component } from 'react';
# leanpub-start-insert
import { connect } from 'react-redux';
import { compose } from 'recompose';
# leanpub-end-insert

import withAuthorization from './withAuthorization';
import { db } from '../firebase';

class HomePage extends Component {
  componentDidMount() {
# leanpub-start-insert
    const { onSetUsers } = this.props;

    db.onceGetUsers().then(snapshot =>
      onSetUsers(snapshot.val())
    );
# leanpub-end-insert
  }

  render() {
# leanpub-start-insert
    const { users } = this.props;
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

# leanpub-start-insert
const mapStateToProps = (state) => ({
  users: state.userState.users,
});

const mapDispatchToProps = (dispatch) => ({
  onSetUsers: (users) => dispatch({ type: 'USERS_SET', users }),
});
# leanpub-end-insert

const authCondition = (authUser) => !!authUser;

# leanpub-start-insert
export default compose(
  withAuthorization(authCondition),
  connect(mapStateToProps, mapDispatchToProps)
)(HomePage);
# leanpub-end-insert
~~~~~~~~

Now the users are managed with Redux rather than in React's local state. You have connected the state and actions of Redux with the view layer.

What about the session state layer which should be handled by the session reducer? Essentially you will refactor it the same way as the user state layer before. You will replace the provider pattern, where the authenticated user is stored in React's context, with the state layer from Redux, where the authenticated user will be stored in the Redux store. Thus, instead of passing the authenticated user object down via React's context, you pass it down via the global Redux store by providing the store in a parent component (via the Provider component, which you already did) and connecting it to the components that care about the authenticated user (e.g. Navigation, Account).

The most important component to store the authenticated user object in the Redux store rather than in React's context is the `withAuthentication()` higher order component. We can refactor it to use the Redux store instead of React's context by connecting it to the state layer.

{title="src/components/withAuthentication.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { connect } from 'react-redux';
# leanpub-end-insert

import { firebase } from '../firebase';

const withAuthentication = (Component) => {
  class WithAuthentication extends React.Component {
    componentDidMount() {
# leanpub-start-insert
      const { onSetAuthUser } = this.props;
# leanpub-end-insert

      firebase.auth.onAuthStateChanged(authUser => {
        authUser
# leanpub-start-insert
          ? onSetAuthUser(authUser)
          : onSetAuthUser(null);
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
  const mapDispatchToProps = (dispatch) => ({
    onSetAuthUser: (authUser) => dispatch({ type: 'AUTH_USER_SET', authUser }),
  });

  return connect(null, mapDispatchToProps)(WithAuthentication);
# leanpub-end-insert
}

export default withAuthentication;
~~~~~~~~

Now you have the authenticated user available in the Redux store. As consequence all components which rely on the authenticated user in React's context need to be refactored now. It's similar to the Home component which uses the list of users from the Redux store instead of React's local state.

In the Navigation component, the authenticated user is used to display different routing options. Thus you will need to refactor the component to connect it to the Redux store.

{title="src/components/Navigation.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { connect } from 'react-redux';
# leanpub-end-insert
import { Link } from 'react-router-dom';

import SignOutButton from './SignOut';
import * as routes from '../constants/routes';

# leanpub-start-insert
const Navigation = ({ authUser }) =>
# leanpub-end-insert
  <div>
    { authUser
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
const mapStateToProps = (state) => ({
  authUser: state.sessionState.authUser,
});

export default connect(mapStateToProps)(Navigation);
# leanpub-end-insert
~~~~~~~~

In the Account component, the authenticated user is used to display the email address of the user. There you need to connect it to the Redux store as well.

{title="src/components/Account.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { connect } from 'react-redux';
import { compose } from 'recompose';
# leanpub-end-insert

import { PasswordForgetForm } from './PasswordForget';
import PasswordChangeForm from './PasswordChange';
import withAuthorization from './Session/withAuthorization';

# leanpub-start-insert
const AccountPage = ({ authUser }) =>
# leanpub-end-insert
  <div>
    <h1>Account: {authUser.email}</h1>
    <PasswordForgetForm />
    <PasswordChangeForm />
  </div>

# leanpub-start-insert
const mapStateToProps = (state) => ({
  authUser: state.sessionState.authUser,
});
# leanpub-end-insert

const authCondition = (authUser) => !!authUser;

# leanpub-start-insert
export default compose(
  withAuthorization(authCondition),
  connect(mapStateToProps)
)(AccountPage);
# leanpub-end-insert
~~~~~~~~

Furthermore, don't forget that the authorization higher order component used the authenticated user from React's context as well for the fallback conditional rendering. You have to refactor it too.

{title="src/components/withAuthorization.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { connect } from 'react-redux';
import { compose } from 'recompose';
# leanpub-end-insert
import { withRouter } from 'react-router-dom';

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
      return this.props.authUser ? <Component /> : null;
# leanpub-end-insert
    }
  }

# leanpub-start-insert
  const mapStateToProps = (state) => ({
    authUser: state.sessionState.authUser,
  });

  return compose(
    withRouter,
    connect(mapStateToProps),
  )(WithAuthorization);
# leanpub-end-insert
}

export default withAuthorization;
~~~~~~~~

That's it. In this section, you have introduced Redux as state management library to manage your session and user state. Instead of relying on React's context for the authenticated user object and React's local state for the list of users from the Firebase database, you are storing these objects in the Redux store. You can find the project with a slightly different folder structure in this [GitHub repository](https://github.com/rwieruch/react-redux-firebase-authentication).