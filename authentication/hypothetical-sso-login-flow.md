# SSO Login Flow
A hypothetical implementation of an SSO login flow

## Flows
### Visit landing page
1. User browses to host -> default route
1. Router directs user to auth provider
1. Auth provider checks local storage for JWT
    1. If present, checks expiration of token using the `exp` value.
        1. If not valid -> sets auth object as valid==false
        1. Effect triggered on auth object -> reset auth object
        1. Move onto App entrypoint/application routes
    1. If present and valid
        1. parse token for metadata
        1. fetches authZ details from authorization service
        1. Adds authz details to auth objects (eg roles the user has assigned to them)
        1. move onto App entrypoint/application routes


### App entrypoints
1. Route path "/" uses protected element `element={<ProtectedRoute/>}`
1. ProtectedRoute loads auth object context.
    1. If user isn't authenticated, redirect to `/login` (see "login" section below)
    1. If user is authenticated but not authorised, redirect to `/login?reason=notauthorised`
    1. If user is authenticated and authorised, continue to load element from calling function
1. Default path rewritten to "/whatever" and rest of app loads

From here on in, custom components are used, which have the notion of a "role". This is how authorisation is managed


### Login
1. When user browses (or is redirected) to `/login`, the login page is loaded
1. When the page loads, localStorage of tokens are cleaned up
1. User is presented with login button which takes them to SSO provider link (using window.location.href)
1. The SSO provider link consists of hostname with URL params:
    1. successRedirect (webapp hostname)
    1. failureRedirect (webapp hostname)
1. Auth process is now handed off to SSO provider
1. User logs into SSO and user is redirected to the URL from successRedirect or failureRedirect (depending on the authN outcome)
    1. Success goes `/sso` which parses JWT.
    1. Middleware picks up `/sso` and adds necessary elements (eg headers) which container valid tokens.
    1. Tokens are stored in local storage where needed.
    1. If JWT valid, redirected to "/" where "App Entrypoints" flow starts again
    1. If JWT not valid, redirected to "/login"



https://my.disneystreaming.com/app/ds-sso_dsscarifqa_1/exk3j7gl7ac3KhD1x697/sso/saml?SAMLRequest=nZJvT8IwEMa%2FytL3WzcmwzX8CbIYiagLoC98Q8raQXVrR69D%2BPYWJhETQ4xJX12fu9%2Fdc9cd7MrC2XINQskeCjwfDfpdoGVRkWFt1nLKNzUH41iZBHL86KFaS6IoCCCSlhyIychs%2BDAhLc8nlVZGZapAzjjpocXyOsxZm8Z%2Bpx1F3M%2BZz2LkvJyANsMKAWo%2BlmCoNDbkt0LXb9s3DwJyFZJ27MVx9Iqc9Kv0jZBMyNXlPpaNCMjdfJ666dNsjpzETiIkNUf02pgKCMbl3mMCJN%2BD0ZyWNsfLVIlpVWEGLoBaMICMapFv6CLAfPcevnVWRYdm4f06CXZR3MFWhQ%2FeIGcIwPUBMFIS6pLrGddbkfHn6eQb2ZRzN%2FR3MssBNUsgR2v0mfuXh6YnOur%2FidXFZ5TT3h9t2XGSqkJk%2B%2F%2Fs%2FVbpkprL6kNEMDc%2FSkl1OAcwXBprYFGoj5Ht0fAeMrrmCPebNn9eZP8T&RelayState=%7B%22provider%22%3A%22okta%22%2C%22successRedirect%22%3A%22http%3A%2F%2Flocalhost%3A3000%2Fsso%22%2C%22failureRedirect%22%3A%22http%3A%2F%2Flocalhost%3A3000%2Flogin%3Ferror%3Dtrue%22%2C%22passive%22%3Afalse%7D
