# Reactive Auth Store

Firebase like reactive authentication store for user state and authentication tokens.

# Features
* Store user data and tokens.
* State is synchronized across tabs.
* State can be used in react.
* State can be used outside react, in places like axios interceptors, etc.
* Your own types for user and tokens
* Uses local storage, and is extendable to use other storage options.

# General Usage

## Step 1: Create the `getStore` Function

`store.ts`
```typescript
import {
    AuthStore,
    LocalStoragePersistance
} from 'reactive-auth-store';

// Define type for user
export interface AuthUser {
    id: string
    email: string
}

// Define type for tokens
export interface AuthTokens {
    access_token: string
    refresh_token: string
    access_token_expiration: number
    refresh_token_expiration: number
}


let store: AuthStore<AuthUser, AuthTokens> | null = null;

const AUTH_STORAGE_KEY = '__auth__'

// function to lazily create store when required
export default function getStore() {
    if (store === null) {
        store = new AuthStore(
            // create persistor to save state
            new LocalStoragePersistance(AUTH_STORAGE_KEY),
            // a function to compares user equality
            (userA, userB) => (userA?.id || null) === (userB?.id || null)
        )
    }
    return store;
}
```

## Step 2: Use the Auth Store

Login and Logout Users
```ts
import getStore from './store';
import type { AuthUser, AuthTokens } from './store';

// Login User
async function login(user : AuthUser, tokens: AuthTokens) {
    const store = getStore();
    await store.setUser(user);
    await store.setTokens(tokens);
}

// Logout
async function logout() {
    const store = getStore();
    await store.setUser(null);
    await store.setTokens(null);
}
```

Get Current State
```typescript
const user = getStore().getUser();
const tokens = getStore().getTokens();
```

Subscribe to Changes
```typescript
import getStore from './store';

const store = getStore();

store.on('initialized', function() {
    console.log('storeInitialized');
});

store.on('userChanged', function(user : AuthUser) {
    console.log('userChanged', user);
});
```

# Usecase Examples

## Creating a `useAuthUser` Hook and a `Protected` Component

Create a hook to get the current user from the store:

`use-auth-user.ts`

```ts
import getStore from './store';
import type { AuthUser, AuthTokens } from './store';
import { useEffect, useState } from "react"

export const useAuthUser = () => {

    const [initialized, setInitialized] = useState(false);
    const [user, setUser] = useState<AuthUser | null>(null);

    useEffect(() => {

        const store = getStore();

        const onInitialized = () => {
            setInitialized(true);
        }

        const onUserChanged = (user: AuthUser | null) => {
            setUser(user);
        }

        // initial values
        setInitialized(store.getInitialized());
        setUser(store.getUser());

        // listen to changes
        store.on('initialized', onInitialized);
        store.on('userChanged', onUserChanged);

        return () => {
            // cleanup
            store.off('initialized', onInitialized);
            store.off('userChanged', onUserChanged);
        }
    }, []);

    return { initialized, user }
}
```

Create a component that allows only logged-in users:

`protected.tsx`

```jsx
import { useAuthUser } from './user-auth-user';
import { Navigate } from "react-router-dom";
import * as React from "react";

export function Protected ({ children }: { children: React.ReactElement }) {
    const { initialized, user } = useAuthUser();
    if (!initialized) {
        return null;
    }
    if (!user) {
        return <Navigate to='login' replace={true} />
    }
    return children;
}
```

## Axios Interceptor to Attach the Access Token

`attach-token-interceptor.ts`

```typescript
import { InternalAxiosRequestConfig } from "axios";
import getStore from "./store";

export default function attachTokenInterceptor(
    config : InternalAxiosRequestConfig<any>
) {
    // attach access token if present
    const accessToken = getStore().getTokens()?.accessToken;
    if (accessToken) {
        config.headers.set('Authorization', `Bearer ${accessToken}`);
    }

    return config;
}
```

## Axios Interceptor to Refresh the Access Token

`refresh-token-interceptor.ts`

```typescript
import { InternalAxiosRequestConfig } from 'axios';
import getStore from './store';

export default async function refreshTokenInterceptor(
    config : InternalAxiosRequestConfig<any>
) {

    // user is not logged
    const user = getStore().getUser();
    if (!user) {
        return config;
    }

    // refresh access token if expired
    await refreshTokenIfExpired();

    return config;
}

// refresh access token if it has expired
async function refreshTokenIfExpired() {

    // Get current tokens
    const store = getStore();
    const tokens = store.getTokens();

    // Destructure the token properties with default values
    const {
        access_token='',
        refresh_token='',
        access_token_expiration=0,
        refresh_token_expiration=0,
    } = tokens || {};

    // check if access token is expired
    const now = new Date();
    const expiration = new Date(access_token_expiration * 1000);
    const tokenExpired = expiration.getTime() <= now.getTime();

    // token is still valid 
    if (!tokenExpired) {
        return;
    }

    // api call to refresh access token
    const {
        token,
        expiration
    } = await refreshAccessTokenApi();

    // update tokens in store
    await store.setTokens({
        access_token: token,
        access_token_expiration: expiration,
        refresh_token: refresh_token,
        refresh_token_expiration: refresh_token_expiration,
    })
}
```

Feel free to use and extend the Reactive Auth Store to simplify user authentication and state management in your react application.