# Kubernetes and OIDC
(For notes on OAuth2 and OIDC, see notes on [oauth2-and-oidc](../../authentication/oauth2-and-oidc.md))

## Authentication
We'll look at how to authenticate with the Kube API server using OIDC.

![oidc-flow-diagram](./imgs/oidc-flow-diagram.png)

1. User logs in to the Identity Provider
1. The IDP provides:
    1. Access Token
    1. ID Token (JWT)
    1. Refresh Token
1. User calls `kubectl`
1. Token info is sent along within the Auth: Bearer header
1. Kube API server verifies that the JWT token was signed by the IDP
1. Kube API server verifies the token isn't expired
1. Kube API server verifies the user is authorised to access the resource requested in the `kubectl` command.
1. Result returned from API to `kubectl` client
1. Return returned to user

- Kube API doesn't speak to the identity provider when this transaction happens. This is possible because the identity provider signed the token with their private key. 
- The API server has the public key (because this is something that is retrieved once in a while, seperate to this example flow).
- Using the public key of the identity provider, the API server is able to verify that the token was indeed signed by the **trusted** identity provider.

## Pods Using IAM Roles
This is covered in the [AWS Technical Docs](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html) and [Best Practices Docs](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#kubernetes-service-accounts)

Pods are able to gain the access that an IAM role grants by using service accounts. This feature is called "IAM Roles for Service Accounts" (IRSA). It works by leveraging a Kubernetes feature known as Service Account Token Volume Projection. 

AWS IAM supports federated identities using OIDC. This allows us to authenticate AWS API calls with a supported identity provider and recieve a valid OIDC JWT. This token can then be passed to the AWS STS `AssumeRoleWithWebIdentity` API operation and recieve IAM temporary role credentials.

EKS hosts a public OIDC discovery endpoint per cluster (eg: `https://oidc.eks.us-east-1.amazonaws.com/id/<guid-assigned-per-cluster>/.well-known/openid-configuration`) which contains the signing keys for the `ProjectedServiceAccountToken` JWTs, so external systems like IAM can validate and accept the OIDC tokens issued by Kubernetes.

## Technical Docs Summary
1. Service Accounts are associated with IAM roles by the `eks.amazonaws.com/role-arn` annotation in the Service Account spec.
1. The [Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook) on the cluster watches for pods that have this annotation and applies the following environment variables to them:
    1. `AWS_ROLE_ARN=<role-arn>`
    1. `AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
1. The Kubernetes API server will also call the public OIDC discovery endpoint for the cluster on startup. The public OIDC provider endpoint will look something like `https://oidc.eks.us-east-1.amazonaws.com/id/<guid-assigned-per-cluster>/.well-known/openid-configuration`.
1. The endpoint cryptographically signs the OIDC token, issued by Kubernetes
1. The resulting token is mounted as a volume.

## Best Practices Docs Summary
1. API server calls the Public OIDC discovery endpoint
1. The endpoint cryptographically signs the OIDC token issued by Kubernetes
1. The resulting token is mounted as a volume on the pod
1. This signed token allows the Pod to call the AWS APIs associated IAM role.
1. When the AWS API is invoked, the AWS SDKs call `sts:AssumeRoleWithWebIdentity`
1. After validating the token's signature, IAM exchanges the Kubernetes issued token for a temporary AWS Role Credential.
1. The [Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook) is a mutating webhook which runs as part of the EKS control plane and injects the following environment variables to pods which are using a service account which have the `eks.amazonaws.com/role-arn` annotation:
    1. `AWS_ROLE_ARN=<role-arn>`
    1. `AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
1. The kubelet will automatically rotate the projected token when it is older than 80% of its TTL or after 24 hours. The AWS SDKs are responsible for reloading the token when it rotates.

## Deep Dive Summary
This is good summation of what happens. The notes below are taken from [this blog](https://blog.mikesir87.io/2020/09/eks-pod-identity-webhook-deep-dive/)

As mentioned above, the mutating webhook adds the token env vars and projects a token volume to the pod based on the service account annotation. The env vars are used by the AWS SDKs to automatically assume the specified role. In order to work an OIDC provider is configured in IAM to trust the ServiceAccount tokens. 

### 1. OIDC Workflow
- The OIDC provider has a public discovery endpoint (eg `https://oidc.eks.us-east-1.amazonaws.com/id/<guid-assigned-per-cluster>/.well-known/openid-configuration`)
- The `<jwks_uri>` endpoint of the above, provides a JSON Web Key Set. This is a collection of public keys being used to sign tokens from the issuer.
- The OIDC provider is configured to ensure the token receiver doesn't trust just anyone.

1. A Client wants to authenticate with an API. It has a JWT token that identifies itself which has been signed by the trusted provider.
1. The API recieves the JWT and determines if it has the public key identified by the `kid` (Key ID) attribute in the JWT header. If it doesn't recognise the key, it:
    1. Fetches the OIDC config at the discovery endpoint
    1. Uses the `<jwks>` endpoint to get the provider's published public keys (the JWKS).
1. Once the API has the providers public key, it validates the signature of the token, expiration, audience claims (who it's intended for, aka Client ID), etc.
1. Once validated, the API can assure the claims in the token are valid

### 2. Creating an OIDC Provider
EKS will create an OIDC Provider URL when the cluster is created. This will have the public discovery endpoints and the jwks config but you'll still need to create an identity provider within the IAM console which specifies the `sts.amazonaws.com` audience.

### 3. OIDC with AWS IAM
Once we have the provider in IAM, we can configure AWS IAM to trust the tokens which have been generated by it. We want to allow the JWTs for a service account to obtain STS tokens for a particular role. We do this by creating a trust relationship on the desired role. 

The below example is authorising an IAM role to be assumed by a client that has a JWT token with a `sub` claim of `system:serviceaccount:mynamespace:hello-world-app` which is the service account called `hello-world-app` in the `mynamespace` namespace.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::1234567890123:oidc-provider/my-oidc-provider.example.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "my-oidc-provider.example.com:sub": [
            "system:serviceaccount:default:hello-world-app"
          ]
        }
      }
    }
  ]
}
```

To express this as a sequence diagram:

![iam-oidc-seq](./imgs/iam-oidc-seq-diag.png)

### 4. Service Account Token/JWT
This section will answer _how_ the service account gets it's JWT. 

- Every pod by default as a service account JWT **but** the default service account has no `exp` (expiry) claim. Having a trusted JWT with no expiry would be a bad. This is the problem that the mutating webhook solves.
- The webhook leverages the [Service Account Token Volume Projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) feature, which provides a pod with a newly created JWT that contains a specified audience and expiration. 
- The cluster will automatically rotate and update this token for as long as the pod is running
- In order to use this feature, the Kube API server needs to be configured with the following flags:
    - `--service-account-issuer`: The issuer name for the cluster (this is typically a full URI for the issuer)
    - `--service-account-signing-key-file`: A private key to be used with signing the JWTs
    - `--service-account-api-audiences`: A list of audiences allowed to be specified in projected volumes. These also serve as defaults if no specific audience is indicated in mount config.
- The webhook also injects a couple of environment variables:
    1. `AWS_ROLE_ARN=<role-arn>`
    1. `AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
- If we looked at this JWT, it'd look something like this:

```json
{
  "aud": ["aws-iam"],
  "exp": 1600956419,
  "iat": 1600870019,
  "iss": "https://my-oidc-provider.example.com",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "test-iam-pod",
      "uid": "0b65077b-336d-442c-8c47-09ac8bed4b26"
    },
    "serviceaccount": {
      "name": "hello-world-app",
      "uid": "635ee15d-8b81-499e-bde0-093a3b0612ec"
    }
  },
  "nbf": 1600870019,
  "sub": "system:serviceaccount:mynamespace:hello-world-app"
}
```
Note that it contains an `exp` and the name of the Service Account assigned to the pod

AWS SDKs (since around the end of 2019) have been configured to automatically use the configuration specified in the `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`. So if the webhook has injected these into the pod (and the token passes the verify steps) then the pod will be able to assume the IAM role with a set of temporary credentials.
