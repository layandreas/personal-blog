+++
author = "Andreas Lay"
title = "Securing any App with Oauth2Proxy"
date = "2023-11-23"
description = "How to secure apps behind a load balancer with SSO authentication"
tags = ["oauth2proxy", "nginx", "sso", "authentication"]
categories = ["Platform", "Authorisation"]
ShowToc = true
TocOpen = true
+++

## Introduction

When you deploy applications you might want to protect them behind a login. If you're deploying multiple  applications it might not be feasible to add authentication for each deployment separately. Here I'll show how to set up a load balancer with `nginx` and `oauth2proxy` using `Keycloak` to secure any app.

## Run the example

You can find a working example of running a load balancer with authentication in [this repository](https://github.com/layandreas/oauth-proxy-example). You can use `docker compose` to run the example:

```bash
git clone https://github.com/layandreas/oauth-proxy-example.git
cd oauth-proxy-example
docker compose up
```

**Note: Depending on your system you will need to change the oauth2-proxy binary in the Dockerfile. It is set as oauth2-proxy-v7.4.0.linux-arm64.tar.gz which will work with ARM Macs.**


You can login with the user `test` and password `test`:

![Login with authentication flow](/personal-blog/auth_flow.gif)

## Setup Explained

### Streamlit

Our container runs two independent [Streamlit](https://streamlit.io/) apps. We want to secure those apps however Streamlit does not offer native support for authentication. We launch these apps in `supervisor.conf`:

```
[program:app1]
command=streamlit run app1.py --server.port 8501 --server.enableCORS false  --server.enableXsrfProtection false --server.baseUrlPath=/app1
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
priority=2

[program:app2]
command=streamlit run app2.py --server.port 8502 --server.enableCORS false  --server.enableXsrfProtection false --server.baseUrlPath=/app2
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
priority=2
```

### Keycloak

We use [Keycloak](https://www.keycloak.org/) as our identity and access management (IAM) provider:

- Our users live here. Specifically Keycloak supports multi-tenancy through so-called `realms`. You create a  realm on your Keycloak instance and create users within this realm
- You can optionally also add external identity providers like for example [Google](https://cloud.google.com/architecture/identity/keycloak-single-sign-on)
- You can create groups and roles in your realm and assign those to users


We launch Keycloak in `supervisor.conf`:

```
[program:keycloak]
command=./keycloak/bin/kc.sh start-dev
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
priority=0
```

You can access the admin console through [http://localhost:8080/admin](http://localhost:8080/admin) (username: `admin`, password: `admin`). Select your realm and create users, roles, groups, oauth clients, etc. in it.

### oauth2proxy

We are using [oauth2proxy](https://github.com/oauth2-proxy/oauth2-proxy) as our identity aware proxy. It takes care of the whole [OAuth 2.0 flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/which-oauth-2-0-flow-should-i-use) for us.

- It will redirect to your auth provider (in our case Keycloak)
- On successful login it will store the Keycloak [access token](https://jwt.io/) in an encrypted cookie
- If the cookie is present in your request, it will decrypt and validate the access token. If the cookie is not present or the token has expired / is invalid, you will be redirected to the login page


We launch oauth2proxy in `supervisor.conf`:

```
[program:proxy]
# Keycloak runs a build step after startup that takes a while
# therefore wait here a little bit before starting the proxy
command=bash -c 'sleep 15 && /oauth2-proxy-v7.4.0.linux-arm64/oauth2-proxy --http-address=0.0.0.0:4180 --cookie-secure=false --scope=openid profile email groups'
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
priority=1
# The proxy startup will fail if Keycloak hasn't finished building yet,
# in this case we increase the number of retries here
startretries=10
```

We configure the proxy through environment variables, see `docker-compose.yml`:

```yaml
version: "0.1"
services:
  secured-app: 
    build: .
    environment:
      OAUTH2_PROXY_UPSTREAMS: http://localhost:8501/ 
      OAUTH2_PROXY_PROVIDER: keycloak-oidc 
      OAUTH2_PROXY_OIDC_ISSUER_URL: http://localhost:8080/realms/myrealm
      OAUTH2_PROXY_CLIENT_ID: myclient
      OAUTH2_PROXY_CLIENT_SECRET: npXBg26U4NjbhC4lms42xvikvXaHNDlR
      OAUTH2_PROXY_PASS_ACCESS_TOKEN: true 
      OAUTH2_PROXY_EMAIL_DOMAINS: '*' 
      OAUTH2_PROXY_REDIRECT_URL: http://localhost/oauth2/callback 
      OAUTH2_PROXY_COOKIE_SECRET: Ie1OKaRV4-CpoTSTw7WuKSg3iUENTY6YO7yPEytUBk4=
      OAUTH2_PROXY_REVERSE_PROXY: true
      OAUTH2_PROXY_COOKIES_SECURE: false
      OAUTH2_PROXY_COOKIES_SAMESITE: false
      OAUTH2_PROXY_COOKIE_DOMAINS: "localhost"
      OAUTH2_PROXY_SET_XAUTHREQUEST: true
      OAUTH2_PROXY_SET_AUTHORIZATION_HEADER: true
      OAUTH2_PROXY_PASS_HOST_HEADER: true
      OAUTH2_PROXY_CUSTOM_SIGN_IN_LOGO: logo.jpeg
      OAUTH2_PROXY_CUSTOM_TEMPLATES_DIR: custom_templates
```

### nginx

We could use oauth2proxy on its own to secure a single app, however we want to protect multiple backend applications. Here comes [nginx](https://www.nginx.com/) into play. We use nginx as a reverse proxy, i.e. a single point of entry for all requests to our backend apps. nginx is configured through the configuration file `nginx/nginx.conf`.

- We serve our apps under the routes `app1` and `app2`. Access is only allowed after successful authentication (see `auth_request` directive):

```
location /app1 {
    # Anyone with valid account can access
    auth_request /oauth2/auth; 
    include /usr/local/bin/nginx_conf_streamlit_with_auth.conf;
    proxy_pass http://host.docker.internal:8501/app1;
}

location /app2 {
    # Only users with group2 can access
    auth_request /oauth2/auth_group2;
    include /usr/local/bin/nginx_conf_streamlit_with_auth.conf;
    proxy_pass http://host.docker.internal:8502/app2;
}
```

- We set the oauth2proxy endpoints in the config file as well. Note that oauth2proxy can check if the authenticated user is part of groups (which are defined in Keycloak). These endpoints are used for the authentication request:

```
# Anyone with an account can access the backend when using this
# endpoint
location = /oauth2/auth {
    proxy_pass       http://host.docker.internal:4180;
    proxy_set_header Host             $host;
    proxy_set_header X-Real-IP        $remote_addr;
    proxy_set_header X-Scheme         $scheme;
    # nginx auth_request includes headers but not body
    proxy_set_header Content-Length   "";
    proxy_pass_request_body           off;
}

# Anyone with group2 can access the backend when using this
# endpoint
# We need this hacky workaround with different endpoints per group
# as nginx will escape the query string when passing it to the auth server
# with auth_request. Thus we need proxy_pass, see:
# https://github.com/oauth2-proxy/oauth2-proxy/issues/2057#issuecomment-1602546342
location = /oauth2/auth_group1 {
    proxy_pass       http://host.docker.internal:4180/oauth2/auth?allowed_groups=group1;
    proxy_set_header Host             $host;
    proxy_set_header X-Real-IP        $remote_addr;
    proxy_set_header X-Scheme         $scheme;
    # nginx auth_request includes headers but not body
    proxy_set_header Content-Length   "";
    proxy_pass_request_body           off;
}

location = /oauth2/auth_group2 {
    proxy_pass       http://host.docker.internal:4180/oauth2/auth?allowed_groups=group2;
    proxy_set_header Host             $host;
    proxy_set_header X-Real-IP        $remote_addr;
    proxy_set_header X-Scheme         $scheme;
    # nginx auth_request includes headers but not body
    proxy_set_header Content-Length   "";
    proxy_pass_request_body           off;
}
```

## Alternatives

This is everything you need to secure any backend application behind a secure login. If you do not want to use oauth2proxy, [Pomerium](https://www.pomerium.com/) is an alternative worth checking out. You could also replace nginx with [caddy](https://caddyserver.com/docs/quick-starts/reverse-proxy).
