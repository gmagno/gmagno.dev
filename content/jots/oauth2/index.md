---
date: "2022-06-01"
draft: false
showpagemeta: true
title: "OAuth2 authorization code flow"
---

> A few notes on oauth2 **authorization code** flow.

<!--more-->

{{< figure src="/jots/oauth2/the-new-york-public-library-76LwGQkt-gg-unsplash.jpg" caption="Photo by [The New York Public Library](https://unsplash.com/photos/76LwGQkt-gg)" link="https://unsplash.com/photos/76LwGQkt-gg" >}}

OAuth2 is an authorization delegation protocol. Meaning: to grant some third party granular access to user resources on their behalf without impersonating them or sharing secrets/passwords.
The [RFC6749](https://www.ietf.org/rfc/rfc6749.txt) and [RFC6750(https://www.ietf.org/rfc/rfc6750.txt)] cover the protocol flows in detail:

- Authorization Code Grant
- Implicit Grant
- Resource Owner Password Credentials Grant
- Client Credentials Grant

In these notes the [authorization code flow](https://www.rfc-editor.org/rfc/rfc6749#page-24) is covered.

> **ℹ️ Note:**
> Authentication has to do with identity whereas authorization is about permission. OAuth2 is a framework for "permissions" delegation, but it may be extended to support authentication. Check [OpenID Connect](https://openid.net/) for more on that.

### The Problem

In essence, the issue addressed by OAuth2 is as follows: A user desires to grant a third-party application access to specific data or resources they possess on a "service provider" without ever disclosing their credentials.

{{< figure src="/jots/oauth2/use_case_light.png" class="fig_light" caption="User access to service provider through third-party app" >}}
{{< figure src="/jots/oauth2/use_case_dark.png" class="fig_dark" caption="User access to service provider through third-party app" >}}

Three main entities, relevant for understanding the flow, can be identified from the problem statement:

- the user: `Resource Owner`
- the third party application: `Client`
- service provider: `Resource/Authorization Server` (these two don't necessarily run in the same server)

### The Flow

The flow goes, roughly, like this:

{{< figure src="/jots/oauth2/flow_light.png" class="fig_light" caption="OAuth2 authorization code flow" >}}
{{< figure src="/jots/oauth2/flow_dark.png" class="fig_dark" caption="OAuth2 authorization code flow" >}}

1- the user triggers login or attempts to access a protected resource through the `Client`

2- the `Client` responds redirecting the `User-Agent` (browser) to the `Authorization Server`, sending, among other thigs:

- a `Client` identifier
- the scope: list of permissions the `Client` needs to fullfil the user request
- a redirection uri: the endpoint where the user will be redirected back to after authentication (step 3)

3- the user is shown an authentication form and asked to consent with the permissions being asked by the `Client`.

4- the user authenticates and consents with the permissions being asked by the `Client`

5- user is redirected back to the `Client` (to the redirection uri mentioned in step 2). The request includes a short lived `code` that is used to retrieve the access token (step 6). This is the code that gives name to the flow.

6- the `Client` is now able to authenticate and talk directly to the `Authorization Server`. The `Client` authenticates with the `Authorization Server` and "exchanges" the `code` by an access token.

7- the `Authorization Server` responds with an access token, that, from this point on, is used by the `Client` to request data/resources on behalf of the user.

### Proof of Concept

For example purposes, follows some code using FastAPI to depict the flow described above, using Github as the authorization server.

The app implements two endpoints:

- /api/v1/oauth/login
- /api/v1/oauth/code

which relate to steps 1. and 5. from the flow description above, respectively.

```python
from fastapi import APIRouter
from fastapi.responses import RedirectResponse
from httpx import AsyncClient, Response, codes

from oauth2flow.schemas import Token

GITHUB_OAUTH_CLIENT_ID = "client_id goes here"
GITHUB_OAUTH_CLIENT_SECRET = "client_secret goes here"
GITHUB_OAUTH_SCOPE = "read:user user:email"
GITHUB_OAUTH_CODE_REDIRECT_URI = "https://oauth2flow.loca.lt/api/v1/oauth/code"
GITHUB_OAUTH_REDIRECT_URI = "https://github.com/login/oauth/authorize"
GITHUB_OAUTH_ALLOW_SIGNUP = True
GITHUB_OAUTH_ACCESS_TOKEN_URL = "https://github.com/login/oauth/access_token"

oauth_router = APIRouter()


@oauth_router.get("/login", include_in_schema=True)
async def login() -> RedirectResponse:
    github_authorize_url: str = (
        f"{GITHUB_OAUTH_REDIRECT_URI}"
        f"?client_id={GITHUB_OAUTH_CLIENT_ID}"
        f"&redirect_uri={GITHUB_OAUTH_CODE_REDIRECT_URI}"
        f"&scope={GITHUB_OAUTH_SCOPE}"
        f"&allow_signup={GITHUB_OAUTH_ALLOW_SIGNUP}"
    )
    return RedirectResponse(github_authorize_url)


@oauth_router.get("/code", include_in_schema=True)
async def code(code: str, state: str) -> RedirectResponse:
    async with AsyncClient(base_url=GITHUB_OAUTH_ACCESS_TOKEN_URL) as ac:
        response: Response = await ac.post(
            "",
            params={
                "client_id": GITHUB_OAUTH_CLIENT_ID,
                "client_secret": GITHUB_OAUTH_CLIENT_SECRET,
                "code": code,
                "redirect_uri": GITHUB_OAUTH_CODE_REDIRECT_URI,
            },
            headers={
                "Accept": "application/json",
            },
        )
    if response.status_code != codes.OK:
        return None

    response_payload = response.json()
    token = Token.parse_obj(response_payload)
    print(token)  # <-- `token.access_token` holds the actual token
    return RedirectResponse("https://oauth2flow.loca.lt/api/docs")
```

For the full implementation check the repo: [https://github.com/gmagno/oauth2flow](https://github.com/gmagno/oauth2flow)
