---
layout: post
title:  One Session to Rule Them All
description: How you can manage a single user session across multiple clients
date: 2022-05-02
minutes: 6
image:  '/images/posts/one-session-to-rule-them-all/fingerprint.png'
tags:   [session-management, authorization, auth0, wilco]
---

###### *This post was originally published on the [Wilco](https://www.trywilco.com/blog) blog*

Today, it’s common for web and mobile applications to hide some or even all of their functionality behind a user authentication prompt. And while registration using email and password is still the standard, most apps prefer to complement it with trusted social networking services (SNS) which have authentication capabilities for 3rd party software. Google, Facebook, and Discord are some of the larger ones, but there are many more out there.

That said, even with SNS authentication available, its implementation and the management of login sessions are burdensome, especially for early-stage startups that want to focus on their product’s core functionality. Handling various login methods and managing sensitive data isn't easy.

Fortunately, services such as [Auth0](https://auth0.com/), [Okta](https://www.okta.com/), and [Frontegg](https://frontegg.com/) enable adding both authentication and user session management easily. In this post, we’ll focus on Auth0 and login with SNS scenario. We’ll take a quick look at how user sessions work under the hood, then dive into a more complicated scenario we tackled in Wilco: multiple clients sharing one session.

#### Under the hood
The imaginary gaming company “Wow Amazing Games” offers amazing games (wow!) on their amazing website wow-amazing-games.com. They use the module AuthService to integrate their React frontend with Auth0:

``` js
import auth0 from 'auth0-js';
‍
const webAuth = new auth0.WebAuth({
  clientID: process.env.REACT_APP_AUTH0_CLIENTID,
  domain: process.env.REACT_APP_AUTH0_DOMAIN,
  redirectUri: `${window.location.origin}/auth0/callback`,
});

const login = () => {
 webAuth.authorize();
};
‍
module.exports = {
 login,
};
```

But what happens under the hood when calling **webAuth.authorize()**?

![flow1](/images/posts/one-session-to-rule-them-all/flow1.png)

1. The user is redirected to a unique Auth0 server, which presents all configured login options.
2. Auth0 redirects to the SNS selected by the user, with which she will authenticate herself.
3. The user is redirected back to the Auth0 server, which updates its session to indicate that the user is logged in. This session proves to be very important when more clients are added.
4. Auth0 redirects the user back to the application, along with a session token.

#### One client is too easy. Let’s add another one
“Wow Amazing Games” is doing so well that they decide to add an online store selling merchandise of their successful games. Their product sets three requirements:

1. The store website should use a separate domain from the gaming website. Separate domains will provide greater flexibility with the products they could sell on the store website. They choose to use the domain wow-amazing-**store**.com.
2. Seamless login: if the user is logged in to the gaming website, she won’t need to log into the store website.
3. Single logout: the user won’t be able to access the store website if she logs out from the gaming website.

These requirements mean that the user session should be verified silently, without redirecting to Auth0 and requesting the user to log in again. If a valid session exists, the user should seamlessly continue to the store website. Otherwise, she should be considered logged out and redirected to the login page. How can we perform silent login?

```js
const verifyLoginSessionActive = (callback) => {
    webAuth.checkSession({}, callback);
}
```

The function **checkSession** is doing some cool magic. It redirects to the Auth0 server inside a hidden iframe. This way, the Auth0 server can check if a valid login session exists without redirecting the user or presenting any authentication UI. Auth0 will return either a valid token or an error. If **callback** is called with an error, the user will be redirected to the Auth0 server for authentication.

![flow2](/images/posts/one-session-to-rule-them-all/flow2.png)

Amazing, right? Well, not so fast.

#### Third-party cookies to spoil the party
You may have asked yourself how the Auth0 server checks if a login session exists. The answer is cookies. It stores a token as a cookie when the user logs in, reads the cookie when requested by checkSession, and verifies its validity. It works flawlessly on most browsers. But some browsers block this functionality.

Third-party cookies are cookies that are stored under a different domain than you are currently visiting. The iframe created by the store website is precisely that. It tries to access cookies stored under <domain>.auth0.com  while the user is visiting www-amazing-store.com. Browsers such as Firefox, Safari, and incognito mode in Google Chrome block access to third-party cookies and cause checkSession to fail.

In contrast, first-party cookies are cookies stored under the same domain you are currently visiting. Could we make the cookies accessed by the iframe first-party instead of third-party? We certainly can!

#### One domain to rule them all
First, we need to decide on the primary domain name that both clients and the Auth0 server will share. wow-amazing.com is an excellent choice.

Now, let’s modify both clients and Auth0 server domain names to be sub-domains of wow-amazing.com.
- Gaming website: games.wow-amazing.com
- Store website: store.wow-amazing.com
- New Auth0 Server domain: auth.wow-amazing.com.

![flow3](/images/posts/one-session-to-rule-them-all/flow3.png)

Now, cookies accessed by the iframe are considered first-party cookies and checkSession functionality is no longer blocked.

#### What if the user logs out while already being connected to the store in another tab?
The user successfully passed the initial session verification on the store website but decided to log out from the gaming website running in another tab. Single logout means that we need to recognize the user logged out across all subdomains. Let’s look at a solution that assumes we don’t have to recognize it immediately and can allow him to continue using the website for some time:

```js
import auth from './auth/AuthService';
const VERIFY_LOGIN_SESSION_INTERVAL_MILLISECONDS = 15 * 60 * 1000; //15 Minutes

async function verifyLoginSessionActive() {
  try {
   await auth.verifyLoginSessionActive();
  } catch (err) {
    //No active login session found on Auth0
    if (err.code === 'login_required') {
      redirectToLoginPage();
    }
  }

  function startVerifyLoginSessionInterval() {
    //Verify Auth0 session once in X minutes (according to Auth0 best practices)
    return setInterval(() => {
      verifyLoginSessionActive();
    }, VERIFY_LOGIN_SESSION_INTERVAL_MILLISECONDS);
  }
}
```

This code periodically polls Auth0 using **checkSession()** to see if a session exists. If the session does not exist, the user will be redirected to the login page. We follow Auth0’s recommendation to poll once in 15 minutes to avoid any future issues with the rate-limiting of this call.

Another solution to this edge case is to have the app server notify all clients that the user logged out, redirecting the user to the login page. This is also a great solution but more complicated to implement.

#### Wrapping up...
We started with only one website allowing users to authenticate themselves and ended up with multiple clients providing a seamless login and a unified logout. More clients can be added freely as long as they share the same primary domain.
