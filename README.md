## Demo

Live at https://magic-nextjs.vercel.app/login

## Quick Start

```
$ git clone https://github.com/magiclabs/example-nextjs
$ cd example-nextjs
$ mv .env.local.example .env.local
// enter your Magic API keys in your env variables
$ yarn install
$ yarn dev // starts server in http://localhost:3000
```

#### Environment Variables

```
NEXT_PUBLIC_MAGIC_PUBLISHABLE_KEY=get-this-from-magic-dashboard
MAGIC_SECRET_KEY=get-this-from-magic-dashboard
NEXT_PUBLIC_SERVER_URL=http://localhost:3000
JWT_SECRET=enter-your-32-character-secret-here
```

## Tutorial

### Introduction

Magic is a passwordless authentication sdk that lets you plug and play different auth methods into your app. Magic supports passwordless email login via magic links, social login (such as Login with Google), and WebAuthn (a protocol that lets users authenticate a hardware device using either a YubiKey or fingerprint).

Flow of the application: a user goes through any of the auth methods --> Magic returns a decentralized ID (DID) token, which is proof of authentication --> send it to our server to validate --> our server issues a JWT inside a cookie --> on subsequent requests to the server, we validate and refresh the JWT to persist sessions.

### Magic Setup

Your Magic setup will depend on what login options you want. For magic link and webauthn, minimal setup is required. For social login integrations, follow our documentation for configuration instructions https://docs.magic.link/social-login.

Once you have social logins configured (if applicable), grab your API keys from Magic’s dashboard and in `.env.local` enter your Test Publishable Key such as `NEXT_PUBLIC_MAGIC_PUBLISHABLE_KEY=pk_test1234567890` and your Test Secret Key such as `MAGIC_SECRET_KEY=sk_test_1234567890`.

### Magic Link Auth

In `pages/login.js`, we handle `magic.auth.loginWithMagicLink()` which is what triggers the magic link to be emailed to the user. It takes an object with two parameters, `email` and an optional `redirectURI`. Magic allows you to configure the email link to open up a new tab, bringing the user back to your application. With the redirect in place, a user will get logged in on both the original and new tab. Once the user clicks the email link, send the `DID token` to our server endpoint at `/api/login` where we validate it.

```js
async function handleLoginWithEmail(email) {
  try {
    let didToken = await magic.auth.loginWithMagicLink({
      email,
      redirectURI: `${process.env.NEXT_PUBLIC_SERVER_URL}/callback`,
    });
    await authenticateWithServer(didToken);
  } catch (error) {
    // handle error
  }
}

async function authenticateWithServer(didToken) {
  const res = await fetch('/api/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + didToken,
    },
  });
  res.status === 200 && Router.push('/');
}
```

### Social Login Auth

The social login implementation is similar. `magic.oauth.loginWithRedirect()` takes an object with provider, and a redirectURI for where to redirect back to once the user is authenticated. We authenticate with the server on the `/callback` page.

```js
async function handleLoginWithSocial(provider) {
  await magic.oauth.loginWithRedirect({
    provider, // 'google', 'apple', etc
    redirectURI: `${process.env.NEXT_PUBLIC_SERVER_URL}/callback`,
  });
}
```

### WebAuthn

Finally, you can let users login to your app with just a fingerprint (on supported browsers) or yubikey. This is powered by the WebAuthn protocol, and allows users to register and login with their hardware device. The low-level implementation of WebAuthn has a different flow for registering and logging in, but to keep from breaking out our form into login and signup for WebAuthn, we check if the login fails, and if so, we retry by registering the device.

```js
async function handleLoginWithWebauthn(email) {
  try {
    let didToken = await magic.webauthn.login({ username: email });
    await authenticateWithServer(didToken);
  } catch (error) {
    try {
      let didToken = await magic.webauthn.registerNewUser({ username: email });
      await authenticateWithServer(didToken);
    } catch (error) {
      // handle error
    }
  }
}
```

### Completing the Login Redirect

In the `/callback` page, check if the query parameters include a `provider`, and if so, finish the social login, otherwise, it’s a user completing the email login.

```js
useEffect(() => {
  !magic &&
    setMagic(
      new Magic(process.env.NEXT_PUBLIC_MAGIC_PUBLISHABLE_KEY, {
        extensions: [new OAuthExtension()],
      })
    );
  magic && router.query.provider ? finishSocialLogin() : finishEmailRedirectLogin();
}, [magic, router.query]);

const finishSocialLogin = async () => {
  try {
    let {
      magic: { idToken },
    } = await magic.oauth.getRedirectResult();
    await authenticateWithServer(idToken);
  } catch (error) {
    // handle error
  }
};

const finishEmailRedirectLogin = async () => {
  if (router.query.magic_credential) {
    try {
      let didToken = await magic.auth.loginWithCredential();
      await authenticateWithServer(didToken);
    } catch (error) {
      // handle error
    }
  }
};
```

### Server-side

In `/api/login`, verify the `DID token`, create a `JWT`, then set it inside a cookie. On subsequent requests to the server, to check if the user is already authenticated for example, all we have to do is verify the JWT to know if the user has already been authenticated, and is authorized.

```js
export default async function login(req, res) {
  try {
    const didToken = req.headers.authorization.substr(7);
    await magic.token.validate(didToken);
    const metadata = await magic.users.getMetadataByToken(didToken);
    let token = jwt.sign(
      {
        ...metadata,
        exp: Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 7, // one week
      },
      process.env.JWT_SECRET
    );
    setTokenCookie(res, token);
    res.status(200).json({ done: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
}
```

### Persisting Sessions

To make sessions persist, we rely on the JWT that’s stored in a cookie and automatically sent on each request to our server. The endpoint we check is `/api/user` which verifies the token, then refreshes it on each request. If we didn’t refresh the token, we could run into the scenario where a user logs in, then is browsing our site a week later and gets logged out in the middle of their session because the cookie and token we set had expired.

```js
export default async function user(req, res) {
  try {
    if (!req.cookies.token) return res.json({ user: null });
    let token = req.cookies.token;
    let user = jwt.verify(token, process.env.JWT_SECRET);
    let { issuer, publicAddress, email } = user;
    let newToken = jwt.sign(
      {
        issuer,
        publicAddress,
        email,
        exp: Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 7, // one week
      },
      process.env.JWT_SECRET
    );
    setTokenCookie(res, newToken);
    res.status(200).json({ user });
  } catch (error) {
    res.status(200).json({ user: null });
  }
}
```

Leveraging Vercel’s SWR (stale while revalidate) library, our `useUser` hook sends a request to the server with the `JWT`, and as long as we get a user returned, we know it’s valid and to keep the user logged in.

```js
const fetchUser = (url) =>
  fetch(url)
    .then((r) => r.json())
    .then((data) => {
      return { user: data?.user || null };
    });

export function useUser({ redirectTo, redirectIfFound } = {}) {
  const { data, error } = useSWR('/api/user', fetchUser);
  const user = data?.user;
  const finished = Boolean(data);
  const hasUser = Boolean(user);

  useEffect(() => {
    if (!redirectTo || !finished) return;
    if (
      // If redirectTo is set, redirect if the user was not found.
      (redirectTo && !redirectIfFound && !hasUser) ||
      // If redirectIfFound is also set, redirect if the user was found
      (redirectIfFound && hasUser)
    ) {
      Router.push(redirectTo);
    }
  }, [redirectTo, redirectIfFound, finished, hasUser]);
  return error ? null : user;
}
```

### Logout

To complete the authentication, we need to allow users to log out. In `/api/logout` we use Magic’s admin-sdk method to clear the cookie containing the JWT and log the user out of their session with Magic.

```js
export default async function logout(req, res) {
  try {
    if (!req.cookies.token) return res.status(401).json({ message: 'User is not logged in' });
    let token = req.cookies.token;
    let user = jwt.verify(token, process.env.JWT_SECRET);
    await magic.users.logoutByIssuer(user.issuer);
    removeTokenCookie(res);
    res.writeHead(302, { Location: '/login' });
    res.end();
  } catch (error) {
    res.status(401).json({ message: 'User is not logged in' });
  }
}
```
