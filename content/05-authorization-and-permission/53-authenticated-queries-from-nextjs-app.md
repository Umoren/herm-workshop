---
title: "5.3 Authenticated Queries from Next.js App"
metaTitle: "Authenticated Queries from Next.js App"
---

## Objectives
- Run GraphQL queries from a Next.js app
- Using Next.js `API` as a proxy
- Injecting headers/cookies in GraphQL queries


## Exercise 1: Run a Query from Next.js

In the beginning, when we set up the Next.js project, we used the Apollo GraphQL template. This setup installed Apollo and GraphQL libraries and even added a server-rendering logic for GraphQL queries in `lib/apollo.js`. Let’s use Apollo to fetch accounts from our protected GraphQL API.

**Task 1: Create an Account Component**

First create an `Account` component in `components/Account`:

```js
import React from 'react';

const Account = () => {

  return (
    <div>
      
    </div>
  );
};

export default Account;
``` 

Import Apollo and GraphQL libraries in the component:

```js
import React from 'react';
import gql from 'graphql-tag';
import { useQuery } from '@apollo/react-hooks';
```

Use the `gql` tag to create a GraphQL query:

```js
//...
import { useQuery } from '@apollo/react-hooks';

export const ALL_ACCOUNTS_QUERY = gql`
  query AllAccountsQuery {
    account {
      account_name
    }
  }
`;

const Account = () => {...}
```

This query fetches all account names in the `account` table.

Next, create a query using Apollo:

```js
const Account = () => {
  const { loading, error, data } = useQuery(ALL_ACCOUNTS_QUERY);
}
```

`useQuery` returns an object that contains three different possible states of your query. If no response has yet been received, the `loading` state is true. If a response was received, but an error occurred, the information about that error is stored in `error`. Lastly, if a response was returned and nothing went wrong, the `data` property is populated with the payload or an empty array for empty tables.

Here is how to handle these 4 possible cases:

```js
const Account = () => {
  const { loading, error, data } = useQuery(ALL_ACCOUNTS_QUERY);

  if (error) {
    console.log(error);
    return <div>Error</div>;
  }

  if (loading) return <div>Loading</div>;

  if (data.account.length < 1) return <div>Your query was successful but no account was found.</div>;

  return (
    <div>
      {data.account.map(account => (
        <div>{account.account_name}</div>
      ))}
    </div>
  );
};
```

Note that if there was a valid response, there is a possibility that the payload is empty and just sends an empty array. In that case, we should handle by checking the length of the response payload, `data.account.length`.

**Task 2: Setup Home Page for Apollo**

To use the component, import it in the `index.js` page file:

```js
import React from 'react';
import fetch from 'node-fetch';
import { Text } from 'herm';

+import { withApollo } from '../lib/apollo';
+import Account from '../components/account';
```

I also imported the SSR logic for Apollo. To be able to render an Apollo-based component in a Next.js page, we need to wrap the Next.js page with `withApollo`:

```js
export default withApollo(Index);
```

You can now render the `Account` component in your `Index` component:

```js
function Index({ me }) {
  return (
    <Layout me={me}>
      <Text fontSize="32px">Hello, {me.name}</Text>
+     <Account />
    </Layout>
  );
}
```

The `./lib/apollo` is setup with a demo GraphQL endpoint — we need to update it to use ours. To do that, find the `createApolloClient` function and replace `uri` with your GraphQL endpoint (`http://localhost:3100/v1/graphql`):

```js
function createApolloClient(initialState = {}) {
  return new ApolloClient({
    ssrMode: typeof window === 'undefined',
    link: new HttpLink({
+     uri: `${process.env.BASE_URL}/api/graphql`,
      credentials: 'same-origin',
      fetch,
    }),
    cache: new InMemoryCache().restore(initialState),
  })
}
```    

If you reload the page, you should get the following error:


![](https://paper-attachments.dropbox.com/s_1160B8630BA1CF41765159DA18D4020C2250C8909DFA5035D976331A3777F078_1584939634807_image.png)


This error makes complete sense because we protected our API, and we can’t access it unless we send a token. Let’s send one.


## Exercise 2: Use Next.js API as a Proxy

We need to find a way to intercept all API requests and add JWT to the requests' header. Though we are dealing with just one GraphQL API, it is a good practice to use Next.js API to intercept each request and sneak in an auth header.

**Task 1: Create a Proxy**

Create a `graphql.js` fine in `pages/api`. Just like other API functions we have created, it should have an async handler function:

```js
import auth0 from '../../utils/auth0';

export default async function graphQL(req, res) {
  try {
    //...coming up next
  } catch (error) {
    console.error(error);
    res.status(error.status || 500).end(error.message);
  }
}
```   

**Task 2: Get JWT from Incoming Request**

Use the `auth0` util to extract the token from the request calling this API:

```js
import auth0 from '../../utils/auth0';

export default auth0.requireAuthentication(async function graphQL(req, res) {
  try {
    const tokenCache = await auth0.tokenCache(req, res);
    const { accessToken } = await tokenCache.getAccessToken({
      scope: ['openid', 'profile']
    });

    
  } catch (error) {
    console.error(error);
    res.status(error.status || 500).end(error.message);
  }
})
```   

The `getAccessToken` returns an object with the access token which is the user’s JWT.  `auth0.requireAuthentication` ensures that users can’t access this API unless they are logged in.


> Instead of getting the token from `auth0.getSessionData`, we are using the token cache. This API handles refresh token out of the box and keeps the user logged in for as long as the session lifetime, which in our case we set to 1 day in the Auth0 config file. Refreshing the token is handy because our JWTs can live for  1 hour by default. You can elongate the life of a token from the Auth0 client settings, but security recommendation demands that we make them live for a short time.

**Task 3: Send JWT to GraphQL API**

Finally, use the `request` module to send a request to the GraphQL API and attach the token. When a response comes back, forward the response to the Next.js page that asked for it:

```js
import request from 'request';
import util from 'util';
import auth0 from '../../utils/auth0';
import config from '../../utils/config';

export default auth0.requireAuthentication(async function graphQL(req, res) {
  try {    
    const tokenCache = await auth0.tokenCache(req, res);
    const { accessToken } = await tokenCache.getAccessToken({
      scope: ['openid', 'profile']
    });

    const headers = {
      // Attach token to header
      Authorization: `Bearer ${accessToken}`,
      // Set content type to JSON
      'Content-Type': 'application/json'
    };
    const asyncReqPost = util.promisify(request.post);
    // Send request
    const graphQLApiResponse = await asyncReqPost({
      url: `${config.baseAPI}/v1/graphql`,
      headers,
      json: req.body,
      timeout: 5000, // give queries more time to run
      gzip: true
    });

    // Set response header
    res.setHeader('Content-Type', 'application/json');
    // Send response
    res.end(JSON.stringify(graphQLApiResponse.body));

  } catch (error) {
    console.error(error);
    res.status(error.status || 500).end(error.message);
  }
})
```    

You will need to install the `request` module:

```bash
npm install --save request
```

The most important line where we are setting up the `Authorization` header:

```js
{Authorization: `Bearer ${accessToken}`}
```

Since Next.js API is promise-based, we need to convert the `request.post` method to a promise, which is what `util.promisify` does. It takes a function with `func(err, cb){}` signature and turns it into a promise.

Finally, we send a request to the GraphQL API and forward the response to Next.js page or whatever initiated a request to this Next.js API. The API endpoint is stored on the config object, so we need to update the config code in `utils/config.js`:

```js
require('dotenv').config()

export default {
  baseUrl: process.env.APP_BASE_URL,
+ baseAPI: process.env.APP_BASE_API,
  auth0: {...}
};
```

Then add a `.env` variable as well:

```bash
  APP_BASE_URL="http://localhost:3000"
+ APP_BASE_API="http://localhost:3100"
```

Restart the Next.js app and reload your page, you should start getting a successful message:


![](https://paper-attachments.dropbox.com/s_1160B8630BA1CF41765159DA18D4020C2250C8909DFA5035D976331A3777F078_1584942845843_image.png)


**Task 5: Refactor Apollo SSR Code**

When I started this workshop, I did not know about the `next-with-apollo` npm library. This library will help us get rid of everything in `lib/apollo.js` and just call an imported function to setup Apollo.

Delete everything in `lib/apollo.js` and replace with:

```js
import React from 'react'
import { ApolloProvider } from '@apollo/react-hooks'
import { ApolloClient } from 'apollo-client'
import { InMemoryCache } from 'apollo-cache-inmemory'
import { HttpLink } from 'apollo-link-http'

import withApollo from 'next-with-apollo';


export default withApollo(
  ({ initialState }) => {
    return new ApolloClient({
      link: new HttpLink({
        uri: `${process.env.BASE_URL}/api/graphql`,
        credentials: 'same-origin'
      }),
      cache: new InMemoryCache().restore(initialState || {})
    });
  },
  {
    render: ({ Page, props }) => {
      return (
        <ApolloProvider client={props.apollo}>
          <Page {...props} />
        </ApolloProvider>
      );
    }
  }
);
```

You will keep getting the same expected result:

![](https://paper-attachments.dropbox.com/s_1160B8630BA1CF41765159DA18D4020C2250C8909DFA5035D976331A3777F078_1585046669330_image.png)


**Task 6: Test Query with Real Data**

Head back to the Hasura console and run the following query:

![](https://paper-attachments.dropbox.com/s_1160B8630BA1CF41765159DA18D4020C2250C8909DFA5035D976331A3777F078_1585047006981_image.png)


```gql
mutation MyQuery {
  insert_account(objects: {account_name: "test", access_token: "test"}) {
    affected_rows
  }
}
```

Note that the `x-hasura-admin-secret` is enabled because we have not given the user role the permission to insert in the database. Until we give this privilege to the user, we still need to be an admin to insert.

After inserting, uncheck the `x-hasura-admin-secret` header.

Now reload the browser, and you should see the test account you added is listed:


![](https://paper-attachments.dropbox.com/s_1160B8630BA1CF41765159DA18D4020C2250C8909DFA5035D976331A3777F078_1585047186346_image.png)


