# Introduction 
Docker Compose deployment of OAuth2-Proxy with Nginx-Proxy-Manager and Redis. Made Easy!!!

**NOTE: This setup assumes that you will be using Microsofts Azure Active Directory Services.**

**If you need to setup other providers, you can see how to configure the "oauth2-proxy.cfg" file [here](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider).**


# Requierments
1. Docker Engine
2. Docker Compose
3. Microsoft Azure Account - The Active Directory services can be used for free.

# Setup Guide
1. First you need to create an Application in your Active Directory. You can follow this [guide](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#azure-auth-provider) or [this one](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app).

Keep note of the "client_id" & the "client_secret", you will need them to connect to the service.
Also you will need the Tenant ID value.
2. After the applicaton is added now is a good time to add to it all the redirect URI for the services and applictions you will be hosting. Remmember, you can always come back and add more services down the line.

You can learn more about the formats for the Redirect URI that Azure accepts [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/reply-url).
3. Lets prepare the "oauth2-proxy.cfg" file.
    - Here are all the variables that you will need to edit (we will go one by one and explain what they do):
    ```code
    provider = ""
    client_id = ""
    client_secret = ""
    oidc_issuer_url = ""
    email_domains = "*"
    cookie_name = "_oauth2_proxy"
    cookie_secret = ""
    cookie_domains = ""
    cookie_expire = "168h"
    cookie_refresh = "1h"
    cookie_secure = true
    cookie_httponly = true
    ```

4. Now you can edit the variables in .env file. The variables are already set to work properly, but in case you have services that use one of the ports declared in the file you will need to change those to ports that are not being used.

**NOTE: If you already have Redis or Nginx Proxy Manager running on your system you can comment out these services from the "docker-compose.yml". You don't need two instances of these applications for this to work. Keep in mind that you might need to change the enviornment variable "OAUTH2_PROXY_REDIS_CONNECTION_URL" to reflect your current Redis configuration.**

