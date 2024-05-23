# FedCM for OAuth

FedCM for OAuth describes how OAuth clients and servers can use FedCM.

Note that the language used in FedCM is slightly different from the OAuth terminology. In particular:

* IdP - short for "Identity Provider", also called "authorization server"
* RP - short for "Relying Party", known in OAuth as the "client"

## Authorization Servers

### FedCM Endpoints

This guide assumes you are starting with a functional OAuth server, including authorization endpoint and token endpoint.

The first part of this will be setting up the core FedCM endpoints. You can follow the MDN [IdP Integration](https://developer.mozilla.org/en-US/docs/Web/API/FedCM_API/IDP_integration) guide for this.

#### well-known

Create a `.well-known/web-identity` endpoint at your OAuth server's domain. This MUST be at the eTLD+1, and cannot be at a subdomain. For example, if your OAuth server is `login.example.com` this file must go at `example.com/.well-known/web-identity`. (See [580](issue)(https://github.com/fedidcg/FedCM/issues/580) for a possible DNS alternative.)

The well-known file must be served with the `application/json` content type. The content of the file is a JSON object with the full URL to the IdP's configURL.


`https://example.com/.well-known/web-identity`

```
{
  "provider_urls": ["https://login.example.com/fedcm/config.json"]
}
```

#### IdP config file

The IdP config file contains links to other FedCM endpoints. This is a JSON document as well.

`https://login.example.com/fedcm/config.json`

```
{
  "accounts_endpoint": "/accounts",
  "client_metadata_endpoint": "/client_metadata",
  "id_assertion_endpoint": "/assertion",
  "login_url": "/login"
}
```

The properties are as follows:

* `accounts_endpoint` - The browser will make a request to this endpoint to determine if the user is logged in and fetch their photo/name to display in the widget
* `client_metadata_endpoint` - This returns the client's terms of service and privacy policy URLs to the browser as registered at the IdP. See [issue 581](https://github.com/fedidcg/FedCM/issues/581) for an alternative that lets the client provide these URLs instead.
* `id_assertion_endpoint` - This is the endpoint the browser gets a credential from after the user clicks log in. For OAuth, this will be where the authorization server returns an authorization code to the browser.
* `login_url` - If the user is not logged in, the browser will launch a popup window to this URL so the user can log in.

Note that the browser will not follow any redirects, these URLs in the config file must respond with JSON data directly.

#### accounts endpoint

The accounts endpoint is expected to return the list of accounts the user is logged in to at the IdP.

The browser will make a GET request to this endpoint along with any cookies for this domain, but does not contain a `client_id`, Origin or Referer header.

```
GET /accounts HTTP/1.1
Host: login.example.com
Accept: application/json
Cookie: ebc7940893e0c4035e9179d0b83
Sec-Fetch-Dest: webidentity
```

The IdP checks the Sec-Fetch-Dest header and validates the cookie to determine which user is logged in, then returns the account information:

```
{
  "accounts": [
    {
      "id": "1234",
      "given_name": "Example",
      "name": "Example User",
      "email": "user@example.org",
      "picture": "https://example.com/photo.jpg"
    }
  ]
}
```

Note: The `email` value will be displayed in the account chooser in the browser. This does not have to actually be an email address though, so you can return another user identifier like username or phone number instead. See [issue 317](https://github.com/fedidcg/FedCM/issues/317) to track a proposal to change this parameter.

If the user is not signed in, the IdP should return an `HTTP 401` response.

#### client metadata endpoint

The browser will request the client metadata from the IdP at this endpoint. This assumes this information has been registered at the OAuth server ahead of time.

```
{
  "privacy_policy_url": "https://client.example.net/privacy_policy.html",
  "terms_of_service_url": "https://client.example.net/terms_of_service.html"
}
```

#### ID assertion endpoint

This endpoint is what makes it all work. After the user clicks the browser "continue" button, the browser makes a request to this endpoint finally linking the RP and the IdP. The request is `application/x-www-form-urlencoded`, and includes parameters with details of the attempted sign-in, as well as cookies from the IdP.

* `client_id` - The identifier of the client
* `account_id` - The ID of the account the user selected from the accounts endpoint

Note: in the future, the RP will probably be able to pass arbitrary parameters to this endpoint ([issue 556](https://github.com/fedidcg/FedCM/issues/556)). This will allow the RP to use PKCE to better secure the end-to-end flow. In the mean time, the RP can include a PKCE code_challenge in the `nonce` parameter.

This request will contain cookies, so the IdP will know if the user is logged in and which user is logged in. Other validation steps the IdP does at this stage include:

* Ensure the `Sec-Fetch-Dest: webidentity` header is present
* Validate the `client_id` is an acceptable client and that it's ok to process the authorization for this user for this client

For example, if the request contained scopes that the user has not previously authorized to this client, the IdP should not issue an authorization code at this time, and instead should get the RP to send the client through a normal redirect flow to obtain authorization.

At this point, the IdP can return data to the RP. The IdP builds a JSON string containing an OAuth authorization code. The client will later exchange this authorization code for an access token. The response should look like the below, including the CORS headers:

```
Content-type: application/json
Access-Control-Allow-Origin: https://client.example.org
Access-Control-Allow-Credentials: true

{
  "token": "<authorization code>"
}
```

The fact that this authorization code is returned in a string called `token` is a bit odd. See [issue 578](https://github.com/fedidcg/FedCM/issues/578) to track progress on allowing the IdP to return JSON instead of a "token" string.

Note: The `Access-Control-Allow-Origin` header must not contain a trailing slash. If you don't set the right CORS headers here, the browser will throw an error in the console.

### IdP Implementation Notes

#### Sec-Fetch-Dest header

For any requests that include cookies, the browser will send a `Sec-Fetch-Dest: webidentity` header as well. The IdP MUST validate the presence of this header to protect against CSRF attacks. This applies to the Accounts and ID Assertion endpoints.

#### Login Status API

The Login Status API allows an IdP to inform the browser of its login status. It is possible for the IdP to set this status either from JavaScript or an HTTP header. The IdP should update the status when the user logs in or logs out.

Setting the login status from an HTTP header:

```
Set-Login: logged-in

Set-Login: logged-out
```

Setting the login status from JavaScript:

```
navigator.login.setStatus("logged-in");

navigator.login.setStatus("logged-out");
```

If the IdP does not do this, the RP's call to FedCM will fail with the error "Not signed in with the identity provider" before the browser even attempts to fetch the accounts endpoint.

#### Cookies

The IdP cookies must be set to `SameSite=None`, HTTPOnly, and Secure. If these properties are not set, the browser will not send the cookies to the accounts or assertion endpoint, so it will always appear that the user is logged out.

The `SameSite=None` requirement was added in [Chrome 125](https://developers.google.com/privacy-sandbox/blog/fedcm-chrome-125-updates#samesite). See [issue 587](https://github.com/fedidcg/FedCM/issues/587) to track a proposal to change this to something else.


## OAuth Clients

As an RP (client) using FedCM for OAuth, the process is as follows:

* Call `navigator.credentials.get` with a `configURL` of the OAuth server you are trying to get an access token from
* Upon receiving the authorization code (in the "token" property of the returned IdentityCredential), exchange the authorization code at the token endpoint

A detailed version of these steps is below.

### navigator.credentials.get

First, from JavaScript, call `navigator.credentials.get` with the desired `configURL`. If the user is logged in, this will pop up the widget in the corner asking if they want to continue logging in to this site.

Until the RP can include arbitrary data to the assertion endpoint, you can use the `nonce` parameter to include a PKCE `code_challenge`. (This `code_challenge` should be generated by your server and passed to the JS, ensuring that only your server ever has the `code_verifier` value.)

```
    const identityCredential = await navigator.credentials.get({
      identity: {
        context: "signin",
        providers: [
          {
            configURL: "https://login.example.com/fedcm.json",
            clientId: "1234",
            nonce: "<code_challenge>", // this is probably going away https://github.com/fedidcg/FedCM/issues/556
          },
        ]
      },
    }).catch(e => {
      console.log("Error", e.message);
    });

    // If successful, identityCredential.token will be an OAuth authorization code
```

If the user clicks the "Continue" button, the browser will make a request with the IdP cookies to the ID assertion endpoint and return the response to your JavaScript code.

For OAuth, the response returned will be an authorization code. You'll need to exchange the authorization code for an access token at the token endpoint like in the traditional Authorization Code flow.

Typically an RP has a server-side component, so this also means your server can retrieve the tokens directly from the token endpoint.

So at this point, write some JavaScript to send this authorization code up to your server to continue the work.

```
  if(identityCredential && identityCredential.token) {

    const code = identityCredential.token;

    const response = await fetch("/fedcm-login", {
      method: "POST",
      headers: {
        "Content-type": "application/x-www-form-urlencoded",
      },
      body: new URLSearchParams({
        code: code
      })
    });
    
    const responseData = await response.json();

    // responseData will contain whatever your server responded with
  }
```

### Exchange the authorization code

Your server-side code will receive the authorization code from your JavaScript.

Make an [OAuth access token request](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1.3) to the token endpoint including the authorization code and PKCE `code_verifier`.

```
POST https://login.example.com/token
Content-type: application/x-www-form-urlencoded
Accept: application/json

grant_type=authorization_code
&code=<authorization code>
&client_id=1234
&code_verifier=a6128783714cfda1d388e2e98b6ae8221ac31aca31959e59512c59f5
```

The response will be the typical OAuth token response: https://datatracker.ietf.org/doc/html/rfc6749#section-4.1.4

```
 {
   "access_token":"2YotnFZFEjr1zCsicMWpAA",
   "token_type":"example",
   "expires_in":3600,
   "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"
 }
```

## OAuth Details

### Scope

Ideally the client should be able to request OAuth scopes in the FedCM request. Assuming [issue 556](https://github.com/fedidcg/FedCM/issues/556) is resolved, that would be able to be done in the initial request:

```
let {token} = await navigator.credentials.get({
  identity: {
    providers: [{
      clientId: "1234",
      configURL: "https://login.example.com/fedcm.json",
      params: {
        "scope": "photos:read photos:write",
      }
    },
  }
});
```

This would be sent by the browser to the assertion endpoint as:

```
POST /assertion HTTP/1.1
Host: login.example.com/
Origin: https://client.example.org/
Content-Type: application/x-www-form-urlencoded
Cookie: 0x23223
Sec-Fetch-Dest: webidentity

account_id=123&
client_id=client1234&
disclosure_text_shown=false&
param_scope=photos:read+photos:write
```

This request is effectively equivalent to an OAuth request with the OpenID Connect `prompt=none` parameter, meaning there is no opportunity for the IdP to interact with the user before returning the successful response. So the IdP should only actually grant this request to the client if the user has already previously authorized this client with the requested scopes, following the same logic that would have applied to the IdP deciding to skip the consent screen on subsequent requests.

If the IdP does not want to issue the requested grant, there are two options:

* Return an authorization code for a grant without the full list of requested scopes, only the scopes previously authorized, which may be none
* Return an error response

In the case of returning an authorization code, the client will eventually find out that it wasn't granted the full list of scopes requested once it gets the access token response, at which point it can revert to a normal OAuth redirect flow to get the user's consent for the new scopes.

Alternatively to adding custom paramters in the FedCM request, the backend server could first do a PAR request including scope and any other parameters. See PAR below for details.


### Client Authentication

In the case that the IdP wants to require client authentication, such as when using the [FAPI profile](https://openid.net/specs/fapi-2_0-security-profile.html), the client will need to use a server-side backend to assist with the flow. This is already recommended in the [OAuth for Browser-Based Apps BCP](https://oauth.net/2/browser-based-apps/), which recommends using a backend as the OAuth client, keeping the actual access tokens out of the browser.

The way this use of FedCM is written, returning an authorization code from the FedCM response rather than an ID token, means there is a natural way to require client authentication already.


### Pushed Authorization Requests (PAR)

To better support more complex requests, OAuth clients may want to use Pushed Authorization Requests ([RFC 9126](https://oauth.net/2/pushed-authorization-requests/)).

Since this proposal already recommended the browser JS making an initial call to a backend server to request a PKCE `code_challenge`, that step could also initiate a PAR request from the backend server as well. The response to the JS call would include the response from the PAR request, the `request_uri`, which the browser would include in the FedCM call.

For example:

```
const startFlowResponse = await fetch("/fedcm-start", {
  method: "POST"
});
const startFlowParams = await startFlowResponse.json();

// The code above tells the backend server of the client to initiate a PAR request

const parRequestURI = startFlowParams.request_uri;

// Then the PAR request_uri is included in the FedCM call, where the browser later sends it to the assertion endpoint

const identityCredential = await navigator.credentials.get({
  identity: {
    context: "signin",
    providers: [
      {
        configURL: "https://login.example.com/fedcm.json",
        clientId: "1234",
        params: {
          request_uri: parRequestURI
        }
      },
    ]
  },
}).catch(e => {
  ...
});
```

## OpenID Connect

Adding OpenID Connect to this flow requires minimal changes. In fact, you could turn this into an OpenID Connect request by only adding the OpenID scope `openid` to the FedCM request, and everything else stays the same.

```
let {token} = await navigator.credentials.get({
  identity: {
    providers: [{
      clientId: "1234",
      configURL: "https://login.example.com/fedcm.json",
      params: {
        "scope": "openid",
      }
    },
  }
});
```

In this case, the response would still be an authorization code, which the client backend would use at the token endpoint, getting an ID token in the response.

### OpenID Connect response_mode=id_token

If you really wanted to, you could skip the authorization code step and use `response_mode=id_token` to get the ID token in the FedCM response. The `prompt=none` is implied in this request again, since there is no opportunity to insert user interaction in this flow.

```
let {token} = await navigator.credentials.get({
  identity: {
    providers: [{
      clientId: "1234",
      configURL: "https://login.example.com/fedcm.json",
      params: {
        "scope": "openid",
        "response_mode": "id_token"
      }
    },
  }
});
```

The `token` variable in JS would be the OpenID Connect ID token.

The main downside to this approach is that in order to actually do anything with the ID token, you would need to do the full JWT validation wherever the contents of the ID token are consumed. For example, if the browser JS sent the ID token to a backend server, the backend server would need to do full JWT validation of the signature and claims, since the ID token would have come from the untrusted browser request.

In contrast, using the authorization code flow means the backend server receives the ID token in an HTTP response, so it can [skip the JWT signature validation step](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation).


