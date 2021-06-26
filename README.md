# Introduction 
Docker Compose deployment of OAuth2-Proxy with Nginx-Proxy-Manager and Redis. Made Easy!!!

**NOTE: This setup assumes that you will be using Microsofts Azure Active Directory Services.**

**If you need to setup other providers, you can see how to configure the "oauth2-proxy.cfg" file [here](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider).**


# Requierments
1. Docker Engine
2. Docker Compose
3. Microsoft Azure Account - The Active Directory services can be used for free.

# Setup Guide
1. First you need to create an Application in your Active Directory. You can follow this [guide](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#azure-auth-provider) or [this one](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app). Keep note of the "client_id" & the "client_secret", you will need them to connect to the service. Also you will need the Tenant ID value.
2. After the applicaton is created now is a good time to add to it all the redirect URI for the services and applictions you will be hosting. Remmember, you can always come back and add more services down the line. You can learn more about the formats for the Redirect URI that Azure accepts [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/reply-url). The fast explanation is to add them in this format: "https://FQDN/oauth2/callback", where you replace the FQDN with the host name your service or application is hosted on.
3. Lets prepare the "oauth2-proxy.cfg" file.
Here are all the variables that you will need to edit (we will go one by one and explain what they do):
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
- **provider** - this is the actual provider of the 2FA authentication process. In this use case this will be set to "azure".
- **client_id** - this is the client_id value you get after creating the application in Azure Active Directory.
- **client_secret** - this one you will need to create after the application creation process. You can follow this guide [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app#add-a-client-secret).
- **oidc_issuer_url** - in our use case this will be set to "https://sts.windows.net/tenant_id/", where you will replace "tenant_id", with the actual tenant_id of your Azure Active Directory.
- **email_domains** - in this use case this paramater is not requiered, but it is still needed beacuse OAuth2_Proxy will not start with out it. You do not need to put any email address here, the authenticated useres will be pulled from the Azure Active Directory. If you need more useres to have access to your services, you just add them to your active directory and give them access to the application we created earlier.
- **cookie_name** - as the name says, this is just the name of the cookie that will be craeted on the clients browser. You can give it any name you like but I recommend you keep it as is in case you need to troubleshoot, it will be easier to see if the cookie is being generated properly.
- **cookie_secret** - this is the key that is used to encrypt the cookie. You will have to generate this yourself. You can do it online [here](https://www.allkeysgenerator.com/Random/Security-Encryption-Key-Generator.aspx). It is recommended you use 128-bit encryption or higher. **NOTE:Keep this as a TOP SECRET. If you leak this key anyone who gets their hands on it can decrypt your cookies and spoof legitimate sessions and gain access to your services or applications.**
- **cookie_domains** - this is were you will put the domain or hostname of the main server that your services will run on. For example if you have services named "service1" & "service2", hosted on a server with a FQDN of "srv.example.com, you will put here "example.com" as the value. **NOTE: This might be different if you are using a different provider.**
- **cookie_expire** - This is the time the cookie will be active. I have put this to 168 hours, which is 7 days. You can change this to suit your use case but I think one week is enough.
- **cookie_refresh** - This is the time one seassion will be kept open. Default is 1 hour, which I think is enough, but again, you are free to change this to suit your needs.
- **cookie_secure & cookie_httponly** - keep these as is. OAuth2-Proxy throws a fit when I change these. Still don't know why though. If a figure them out I will update the info here.
4. Now you can edit the variables in .env file. The variables are already set to work properly, but in case you have services that use one of the ports declared in the file you will need to change those to ports that are not being used. **NOTE: If you already have Redis or Nginx Proxy Manager running on your system you can comment out these services from the "docker-compose.yml". You don't need two instances of these applications for this to work. Keep in mind that you might need to change the enviornment variable "OAUTH2_PROXY_REDIS_CONNECTION_URL" to reflect your current Redis configuration.**
5. After everything is ready you can go ahead and start the docker containers with this command:
```shell
sudo docker-compose up -d
```
6. After everything is up and running, you will go to the Nginx Proxy Manager Dashboard. If this is your first time using NPM, the default login credentials are:
```code
email: admin@example.com
password: changeme
```
After you login you will be asked to change these.
7. Go ahead and create the proxy hosts for your services and applications. If you are having trouble just use Google. The Nginx Proxy Manager is a well documented project and you will find over a dozen tutorials. The use and setup of NPM is out of the scope of this project.
8. After you have everything setup the way you want it, it is time to add the connection to OAuth2-Proxy.
You will have to edit your proxy hosts one by one and in the "Advanced" tab add this snippet.
```nginx
  ### OAuth Snippet Start ----------------------------------------------------
  location /oauth2/ {
    proxy_pass       http://$server:4180;
    proxy_set_header Host                    $host;
    proxy_set_header X-Real-IP               $remote_addr;
    proxy_set_header X-Scheme                $scheme;
    proxy_set_header X-Auth-Request-Redirect $request_uri; 
  }
 
  location = /oauth2/auth {
    proxy_pass       http://$server:4180;
    proxy_set_header Host             $host;
    proxy_set_header X-Real-IP        $remote_addr;
    proxy_set_header X-Scheme         $scheme;
    # nginx auth_request includes headers but not body
    proxy_set_header Content-Length   "";
    proxy_pass_request_body           off;
 }
 
  location / {
    auth_request /oauth2/auth;
    error_page 401 = /oauth2/sign_in;
    # pass information via X-User and X-Email headers to backend,
    # requires running with --set-xauthrequest flag
    auth_request_set $user   $upstream_http_x_auth_request_user;
    auth_request_set $email  $upstream_http_x_auth_request_email;
    proxy_set_header X-User  $user;
    proxy_set_header X-Email $email;
 
    # if you enabled --cookie-refresh, this is needed for it to work with auth_request
    auth_request_set $auth_cookie $upstream_http_set_cookie;
    add_header Set-Cookie $auth_cookie;

    # Proxy!
    include conf.d/include/proxy.conf;
  }
  ### Oauth Snippet End ----------------------------------------------------------
  ```
