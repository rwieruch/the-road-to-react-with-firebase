# React Application Setup: create-react-app

You are going to implement a whole authentication mechanism in React with sign up, sign in and sign out. Furthermore, it should be possible to reset a password or change a password as a user. The latter option is only available for an authenticated user. Last, but not least, it should be possible to protect certain routes (URLs) to be accessible by authenticated users only. Therefore you will build a proper authorization solution around it.

The application will be bootstrapped with Facebook's official React boilerplate project [create-react-app](https://github.com/facebookincubator/create-react-app). You can install it once globally on the command line, and make use of it whenever you want afterward.

{title="Command Line",lang="text"}
~~~~~~~~
npm install -g create-react-app
~~~~~~~~

After you have installed it, you can bootstrap your project with it on the command line. You don't need to give it the same name.

{title="Command Line",lang="text"}
~~~~~~~~
create-react-app react-firebase-authentication
cd react-firebase-authentication
~~~~~~~~

Now you have the following commands on your command line to start and test your application. Unfortunately, the tutorial doesn't cover testing yet.

{title="Command Line",lang="text"}
~~~~~~~~
npm start
npm test
~~~~~~~~

You can start your application and visit it in the browser. Afterward, let us install a couple of libraries on the command line which are needed for the authentication and the routing in the first place. You will use the official firebase node package for the authentication / database and react-router-dom (React Router) to enable multiple routes for your application. Furthermore, you will need the prop-types node package for React's context later on. You don't need to worry about it for now.

{title="Command Line",lang="text"}
~~~~~~~~
npm install firebase prop-types react-router-dom
~~~~~~~~

In addition, create a *components/* folder in your application's *src/* folder on the command. That's where all your components will be implemented eventually.

{title="Command Line",lang="text"}
~~~~~~~~
cd src
mkdir components
~~~~~~~~

Next, move the App component and all its related files to the *components/* folder. That way, you will start off with a well-structured folder/file hierarchy.

{title="Command Line",lang="text"}
~~~~~~~~
mv App.js components/
mv App.test.js components/
mv App.css components/
mv logo.svg components/
~~~~~~~~

Last, but not least, fix the relative path to the App component in the *src/index.js* file. Since you have moved the App component, you need to add the */components* subpath.

{title="src/index.js",lang=javascript}
~~~~~~~~
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
# leanpub-start-insert
import App from './components/App';
# leanpub-end-insert
import registerServiceWorker from './registerServiceWorker';

ReactDOM.render(<App />, document.getElementById('root'));
registerServiceWorker();
~~~~~~~~

Next, run your application on the command line again. It should work again and be accessible in the browser for you. Make yourself familiar with it if you haven't used create-react-app before.

Before implementing the authentication in React, it would be great to have a couple of pages (e.g. sign-up page, sign-in page) to split up the application on multiple URLs (routes). Therefore, let's implement the routing with React Router first before diving into Firebase and the authentication afterward.
