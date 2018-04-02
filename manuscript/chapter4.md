# Authorization in React

TODO: Write introdcution to this larger section.

## Protected Routes in React with Authorization

When you sign out from the home or account page, you will not get any redirect even though these pages should be only accessible for authenticated users. However, it makes no sense to show a non authenticated user the account page. In this section, you will implement a protection for these routes in case a user signs out. This process is called authorization.

The protection you are going to implement is a form of a **general authorization** for your application. It checks whether there is an authenticated user. If there isn't an authenticated user, it will redirect to a public route. Otherwise, it will do nothing. The condition could be defined as simple as: `authUser != null`. In contrast, a more **specific authorization** could be role or permission based authorization. For instance, `authUser.role === 'ADMIN'` would be a role based authorization and `authUser.permission.canEditAccount === true` would be a permission based authorization. Fortunately, we will implement it in a way that you can define the authorization condition (predicate) in a flexible way so that you have full control over it in the long run.

Similar to the higher order `withAuthentication` component, there will be a higher order `withAuthorization` component to shield away from the logic. It is not used on the App component, but can be used on all components which are associated with a route in the App component. Thus it can be reused for the HomePage and AccountPage components. What's the task of the higher order component? First of all, it gets the condition passed as configurational parameter. That way, you can decide on your own if it should be a general or specific authorization rule. Second, it has to decide based on the condition whether it should redirect to a public page because the user isn't authorized to view the current page.

Let's start to implement the higher order component. First, create it on the command line.

{title="Command Line: src/components/",lang="text"}
~~~~~~~~
touch withAuthorization.js
~~~~~~~~

Second, you have again the common framework for the higher order component.

{title="src/components/withAuthorization.js",lang=javascript}
~~~~~~~~
import React from 'react';

const withAuthorization = () => (Component) => {
  class WithAuthorization extends React.Component {
    render() {
      return <Component />;
    }
  }

  return WithAuthorization;
}

export default withAuthorization;
~~~~~~~~

Third, let me paste the whole implementation details for the higher order component and explain it afterward.

{title="src/components/withAuthorization.js",lang=javascript}
~~~~~~~~
import React from 'react';
# leanpub-start-insert
import { withRouter } from 'react-router-dom';
# leanpub-end-insert

# leanpub-start-insert
import AuthUserContext from './AuthUserContext';
import { firebase } from '../firebase';
import * as routes from '../constants/routes';
# leanpub-end-insert

# leanpub-start-insert
const withAuthorization = (authCondition) => (Component) => {
# leanpub-end-insert
  class WithAuthorization extends React.Component {
# leanpub-start-insert
    componentDidMount() {
      firebase.auth.onAuthStateChanged(authUser => {
        if (!authCondition(authUser)) {
          this.props.history.push(routes.SIGN_IN);
        }
      });
    }
# leanpub-end-insert

    render() {
# leanpub-start-insert
      return (
        <AuthUserContext.Consumer>
          {authUser => authUser ? <Component /> : null}
        </AuthUserContext.Consumer>
      );
# leanpub-end-insert
    }
  }

# leanpub-start-insert
  return withRouter(WithAuthorization);
# leanpub-end-insert
}

export default withAuthorization;
~~~~~~~~

Let's break it down. First, have a look at the render method. It renders either the passed component (e.g. HomePage, AccountPage) or nothing. That's just a fallback in case there is no authenticated user passed by the Consumer component from React's context object. It increases the protection of the component by rendering simply nothing. However, the real logic happens in the `componentDidMount()` lifecycle method. Same as the `withAuthentication()` higher order component, it uses the Firebase listener to trigger a callback function in case the authenticated user object changes. Every time the `authUser` changes, it checks the passed `authCondition()` function with the `authUser`. If the authorization fails, the higher order component redirects to the sign in page. If it doesn't fail, the higher order component does nothing. In order to be able to redirect, the higher order component has access to the history object of the Router by using the in-house `withRouter()` higher order component from the React Router library.

In the next step, you can use the higher order component to protect your routes (e.g. /home and /account) with authorization rules. For the sake of keeping it simple, the following two components are only protected with a general authorization rule that checks if the `authUser` object is not null.

First, wrap the HomePage component in the higher order component and define the authorization condition for it. As mentioned, it checks if the user object is not null.

{title="src/components/Home.js",lang=javascript}
~~~~~~~~
import React from 'react';

# leanpub-start-insert
import withAuthorization from './withAuthorization';
# leanpub-end-insert

const HomePage = () =>
  <div>
    <h1>Home Page</h1>
# leanpub-start-insert
    <p>The Home Page is accessible by every signed in user.</p>
# leanpub-end-insert
  </div>

# leanpub-start-insert
const authCondition = (authUser) => !!authUser;
# leanpub-end-insert

# leanpub-start-insert
export default withAuthorization(authCondition)(HomePage);
# leanpub-end-insert
~~~~~~~~

Second, wrap the AccountPage component in the higher order component and define the authorization condition as well. It isn't any different from the previous usage of the higher order component.

{title="src/components/Account.js",lang=javascript}
~~~~~~~~
import React from 'react';

import AuthUserContext from './AuthUserContext';
import { PasswordForgetForm } from './PasswordForget';
import PasswordChangeForm from './PasswordChange';
# leanpub-start-insert
import withAuthorization from './withAuthorization';
# leanpub-end-insert

const AccountPage = () =>
  <AuthUserContext.Consumer>
    {authUser =>
      <div>
        <h1>Account: {authUser.email}</h1>
        <PasswordForgetForm />
        <PasswordChangeForm />
      </div>
    }
  </AuthUserContext.Consumer>

# leanpub-start-insert
const authCondition = (authUser) => !!authUser;
# leanpub-end-insert

# leanpub-start-insert
export default withAuthorization(authCondition)(AccountPage);
# leanpub-end-insert
~~~~~~~~

That's it, your routes are protected now. You can try it yourself by signing out from your application and trying to access */account* or */home*. It should redirect you.

I guess you can imagine how this technique gives you full control over your authorizations in your application. Not only by using general authorization rules, but by being more specific with role and permission based authorizations. For instance, an admin page, which is only available for users with the admin role, could be protected as follows.

{title="Code Playground",lang=javascript}
~~~~~~~~
import React from 'react';

import AuthUserContext from './AuthUserContext';

const AdminPage = () =>
  <AuthUserContext.Consumer>
    {authUser =>
      <div>
        <h1>Admin</h1>
        <p>Restricted area! Only users with the admin rule are authorized.</p>
      </div>
    }
  </AuthUserContext.Consumer>

const authCondition = (authUser) => authUser.role === 'ADMIN';

export default withAuthorization(authCondition)(AdminPage);
~~~~~~~~

Congratulations! You successfully implemented a full fledged authentication mechanisms with Firebase in React, added neat features such as password reset and password change, and protected routes with dynamic authorization conditions. In the next section, you will explore how you can manage the users who sign up in your application. So far, only Firebase knows about them but you have no own database running to store them yourself.