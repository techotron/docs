Dumping comments from a Slack thread

The pattern you’re referring to is most directly encapsulated by OpenID Connect, often shortened to OIDC, an extension of OAuth 2.0. The best place to learn about OAuth is from the spec, and for that I recommend start with version 2.1. While it is not yet finalized, OAuth 2.1 prunes many of the less secure flows (such as the implicit flows) and incorporates extensions like PKCE which improve the overall security by making the protocol significant more resistant to replay attacks and other tampering.

You can read about the authorization code flow on the official website and you can also read about the OIDC extension protocol on the OpenID website. Here’s a basic outline of the protocol:

    User visits your website/service

    You generate a redirect that includes, among other things, your applications client id (previously register with the users identity provider) and any scopes you’re requesting (including the openid scope of using OIDC)

    The user is redirected to the authorization server/identity provider. Here they login and are shown the scopes being requested by your application.

    Once they approve your application, they will be redirected back to your website/service (at a URL configured during application registration) that includes a very short lived token as a query string parameter

    Your website/service exchanges that short lived token with the authorization server by making a second request, supplying the temporary token (among other information, such as the PKCE challenge)

    If completed successfully, the server will return an auth_token and optionally an id_token (if using OIDC) and optionally a refresh_token (if offline_access or other refresh scope was requested). These tokens are passed (typically as bearer tokens) to the relevant resource servers.

A couple important notes about this flow:

    Do not implement it yourself. Use a library if possible.

    Storing the users tokens securely is your responsibility. Best practice is to store them server-side and use a session cookie, but since you said you didn’t want to do that, your only options are probably localStorage of some kind. Note that this does open the user to various forms of XSS attacks. There is no safe mechanism for storing credentials in the users browser

    auth_token may or may not be JWTs but, per the specification, this is irrelevant to you. They should be considered opaque from the standpoint of your application, and only inspected via the token introspective endpoint of the authorization server.

    id_token MUST be JWTs per the specification, and MUST be validated before use. All relevant claims MUST be valid, or else the whole token should be discarded. That includes iss, sub, iat, nbf, and exp, along with the signature. Extra caution must be used to prevent various signing verification attacks such as the alg=none attack.

As for using tokens across separate services, this is typically frowned upon, particularly in the case of id_tokens. The iss and sub claims MUST be validated before use, and the sub claim in particular is likely to vary from service to service.

I’m happy to answer any questions you may have, but this is a 20k ft view of how OAuth and OIDC are typically used to accomplish the authentication and authorization flow you’re describing.
