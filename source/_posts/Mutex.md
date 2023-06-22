---
title: Refetch Access Token using Mutex-Async in a TypeScript, React, Redux-Toolkit Application
date: 2023-06-22 14:48:40
tags:
  - React
  - TypeScript
  - Redux
  - Redux-Toolkit
---

Handling token expiry and error scenarios is a critical aspect of building secure and reliable web applications.

Let's explore the complexities of refreshing access tokens, addressing 401 errors, and ensuring a seamless user experience.

<!-- more -->

---

## The Challenge of Concurrent Token Refetch Requests

Initially, the task at hand was to reduce the lifespan of the access token to 5 minutes for enhanced security.
So the first step was to introduce a refresh token and use it to update the access token whenever it expired or when the server returned a 401 error.

Refreshing the token based on its expiration time proved to be a straightforward task. As I thought.
However, addressing the 401 error presented a more complex challenge.

My initial implementation for token refetch was flawed, as it failed to handle the scenario effectively.

One of the major issues with the initial solution was that when a page with multiple requests loaded, each request received a 401 response and attempted to refresh the token independently.
This concurrent nature of the requests resulted in a race condition. It was not only inefficient but also led to undesired behavior.

To overcome this challenge, I revisited a previous project, Time3cker, where I had implemented authentication. Upon reviewing the code, I discovered a relevant snippet:

```tsx
public getToken = async (): Promise<string | null> => {
  if (this.regainTokenPromise) await this.regainTokenPromise;
  if (this.isTokenExpired(this.token)) await this.regainToken();

  if (!this.token) throw new Error('Access token is not available');
  return this.token;
};
```

In this code snippet, when requesting the token, it first checks for the presence of a promise for token refetch.
If a promise exists, it waits until it resolves or rejects.
Then, it just verifies if the token is valid. And if not, it simply waits for the token to be refreshed.

Implementing a similar approach seemed promising to address the race condition and ensure that all other requests would wait for the token to be refreshed first.

## Leveraging a Mutex Pattern for Synchronization

Inspired by the concept I encountered in the previous project, I recalled that the mutex pattern could prevent concurrent access to a shared resource, which was precisely the behavior I needed for the token refetching mechanism.
And just decided to leverage a mutex pattern to synchronize the token refetch process.

It was kind of straightforward. I just installed the `async-mutex` library which provides a simple and reliable implementation of the mutex pattern.

```bash
yarn add async-mutex
```

The mutex pattern ensures exclusive access to a shared resource. And in my case it just allows preventing concurrent token refreshes and effectively resolve the racing.

Let's see on the code:

```ts
import { fetchBaseQuery } from "./baseQuery";
import { Mutex } from "async-mutex";

// ...

const mutex = new Mutex();

export const baseQueryWithReauth: ReturnType<typeof fetchBaseQuery> = async (
  args,
  api,
  extraOptions
): Promise<
  QueryReturnValue<unknown, FetchBaseQueryError, FetchBaseQueryMeta>
> => {
  // Wait for mutex unlock before doing anything
  await mutex.waitForUnlock();

  let result = await baseQuery(args, api, extraOptions);

  if (isRevalidateTokenError(result.error)) {
    // Check if mutex is locked
    if (mutex.isLocked()) {
      // Wait for mutex unlock and retry the same query
      await mutex.waitForUnlock();
      result = await baseQuery(args, api, extraOptions);
    } else {
      // Lock mutex to perform token refresh
      const release = await mutex.acquire();

      try {
        // Token refresh logic
        // ...
      } finally {
        // Unlock mutex
        release();
      }
    }
  }

  // Handle error and return result
  // ...

  return result;
};
```

Code ensures exclusive access during token refresh by utilizing the mutex pattern:

1. Before initiating any token-related operations, it awaits the mutex to be unlocked using `await mutex.waitForUnlock()`.
   This ensures that any concurrent requests will wait until the mutex is released.

2. If a 401 error occurs (`isRevalidateTokenError`) and the mutex is not locked, it simply calls the `mutex.acquire()`.
   This locks the mutex, preventing other requests from simultaneously executing the token refresh logic and prevents any other queries from executing.

3. Inside the mutex-protected block, I perform the token refresh logic, such as fetching the `refresh` API endpoint and updating the tokens.

4. Finally, it just releases the mutex using `release()` to unlock it and allow other requests to proceed.

By utilizing the Mutex class from the `async-mutex` package, I ensured that only one thread could acquire the mutex lock at a time.
This guaranteed exclusive access to the token refetch logic, preventing concurrent token refresh operations.

## Actually refetching token

With mutex I made a wrapper for the refetch.
Let's go into a little more detail:

```tsx
try {
  // Trying to fetch the `refresh` API endpoint with `refreshToken`
  const refreshTokenResult = await baseQuery(
    {
      url: "/auth/refresh",
      // Let the `baseQuery` handle header management
      headers: { Authorization: REFRESH_TOKEN_PLACEHOLDER },
    },
    api,
    extraOptions
  );

  // ...
}
```

So when a token refresh is required, it initiates a request to the `/auth/refresh` API endpoint with the `refreshToken`.

I delegate the header management to `baseQuery`. I try to keep the same logic in one place, so there's no conflict later.
Let's see:

```tsx
export const REFRESH_TOKEN_PLACEHOLDER = "_REFRESH_TOKEN_";
const BASE_URL = process.env.REACT_APP_SERVER_ENDPOINT;

export const baseQuery = fetchBaseQuery({
  baseUrl: BASE_URL,
  prepareHeaders: (headers, { getState }) => {
    if (headers.get("Authorization") === REFRESH_TOKEN_PLACEHOLDER) {
      const { refreshToken } = (getState() as RootState).sessionState;

      if (refreshToken) headers.set("Authorization", `Bearer ${refreshToken}`);
      else headers.delete("Authorization");
      return headers;
    }

    const { accessToken } = (getState() as RootState).sessionState;
    if (accessToken) headers.set("Authorization", `Bearer ${accessToken}`);

    return headers;
  },
});
```

The `baseQuery`, based on the incoming parameters from headers, just decides which token to assign to it, including setting the `Authorization` header with the `refreshToken` placeholder.
It's 100% not the most elegant solution and will most likely be redone in the near future, but that's how it is for now:

```tsx
try {
  const refreshTokenResult = await baseQuery(/* args */);

  // Check if we actually got tokens
  if (isTokensResult(refreshTokenResult?.data)) {
    // If so, set 'em and retry the same query
    api.dispatch(setAuthTokens(refreshTokenResult.data));
    result = await baseQuery(args, api, extraOptions);
  } else {
    // If not, just drop the session
    api.dispatch(logout());
    api.dispatch(baseApi.util.resetApiState());
  }
} finally {
  // ...
}
```

1. Upon receiving the response, it checks if the `refreshTokenResult` contains valid tokens by calling the `isTokensResult` function.
   It just verifies whether the data returned from the refresh request conforms to the expected token structure - simple TypeGuard.

2. If valid tokens are obtained, I dispatch the `setAuthTokens` action to update the authentication tokens in the Redux store.
   Subsequently, it retries the original query that triggered the token refresh to ensure a seamless user experience.

On the other hand, if the refresh request fails to provide valid tokens, user should be logged out with an error and a message to SignIn again.
I call the global `logout` action to log out the user and dispatch the `resetApiState` (from RTK Query) action to reset the API state, clearing any existing cached data.

By incorporating this token refresh logic, I ensure that the user's access tokens are continuously updated, providing enhanced security and preventing unauthorized access to sensitive resources.

## Conclusion

Implementing a mutex pattern for token refetch involved addressing the challenge of concurrent token refetch requests and efficiently managing the synchronization of these requests.

By leveraging a minimal implementation of the mutex pattern and incorporating error handling and recovery mechanisms, I achieved a reliable and secure solution.
This approach prevented race conditions and improved the efficiency of token refetching.

Good coding!
