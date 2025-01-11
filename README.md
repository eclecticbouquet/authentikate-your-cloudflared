# authentikate your cloudflared

![cloudflare + authentik](https://github.com/lynArk/authentikate-your-cloudflared/blob/main/img/cloudflare+authentik.jpg?raw=true)

A comprehensive guide to exposing your services with Cloudflare Tunnels, and protecting them behind Authentik authentication. Complete with Docker compose templates. Created as a reminder on how to do this for myself, because I struggled to do so the first time around. There is plenty of documentation out there on this topic, including officially from [Authentik](https://docs.goauthentik.io/integrations/services/cloudflare-access/) and [Cloudflare](https://developers.cloudflare.com/cloudflare-one/identity/idp-integration/generic-oidc/). But as a beginner, I found that what exists was not specific enough. So here are the steps I personally took, in excruciating detail. 

## Notes

- Following this method reveals your traffic in cleartext to Cloudflare. I chose to do so personally, but this is not the way for those looking to have the most privacy and independence from third parties. 
- I use code blocks like ```this``` frequently throughout the guide, for filenames, options, commands, etc. This is for the sake of making clear EXACTLY what needs to be inputted or selected. 
- When going through configuration steps, leave alone any settings that you see that I do not mention. For the purposes of this guide, those settings should remain default.

## Pre-requisites 

- custom domain with Cloudflare as your DNS provider 
- [FREE Cloudflare Zero Trust account](https://dash.cloudflare.com/sign-up/teams)
- access to a host machine with [Docker engine installed](https://docs.docker.com/engine/install/)

## Step-by-step Guide

### 1. Verify service functionality

Verify that your chosen service is accessible at ```http://<your-host's-ip>:<service's port>```. For example, I have a docker container running [BookStack](https://www.bookstackapp.com/), on port 6875. The internal IP address of my host system is ```192.168.1.69```. Therefore, I am going to verify the service's functionality at ```http://192.168.1.69:6875```. This is important to do before starting and along the way, to ensure that any errors you run into are not a result of the service itself. If you are also using Docker for your service, you can check if the container is running with ```sudo docker ps```. This will list all running Docker processes. 

### 2. Deploy Authentik with Docker

[Here is the official documentation for deploying Authentik with Docker](https://docs.goauthentik.io/docs/install-config/install/docker-compose), if you want to cross-reference with my own steps.

- a. Navigate to a folder that you want to store configuration and data in for your Authentik instance. This will be your authentik project folder. Mine is simply named ```authentik```.
- b. Run ```wget https://goauthentik.io/docker-compose.yml```. This will download the official template for deploying Authentik with Docker Compose.
- c. Open ```docker-compose.yml``` with you favorite command-line editor. I use nano for simplicity, so I would run ```sudo nano docker-compose.yml```.
- d. Edit ```docker-compose.yml```. In the [compose](https://github.com/lynArk/authentikate-your-cloudflared/tree/main/compose) folder of this repo, I have included a redacted version of my own configuration as an example. It may help you adjust your own configuration to your own needs. 
- e. Use this command to generate a secret key for Authentik: ```openssl rand -base64 60 | tr -d '\n'```. Copy the output to use momentarily. 
- f. Create a ```.env``` file in the root of your authentik project folder, to accompany your ```docker-compose.yml```. Check out my example ```.env``` file in the [compose](https://github.com/lynArk/authentikate-your-cloudflared/tree/main/compose) folder of this repo. Use the token we just created in place of ```<generated key>```.
- g. Let's test it out. In the root of your authentik project folder, run ```sudo docker compose up -d```. This will spin up the docker stack defined in your ```docker-compose.yml``` file. The ```-d``` flag will enable the stack to run as a "daemon", aka a background process. If you don't include the ```-d``` flag, your stack would shut down upon exiting your terminal. 
- h. Try and access the Authentik UI by going to ```http:/<your-host's-ip>:9000/if/flow/initial-setup/```. Remember to change ```9000``` to whatever custom port you've designated, if you did in fact designate a non-standard port.  
- i. Login with the default admin username, ```akadmin```. Set a strong password, and enter the UI.

#### Troubleshooting: 

If the UI is not accessible, there are a number of possible reasons why. Here are a few to investigate:

- Your ```docker-compose.yml``` is not configured properly. You need to check the logs for the Authentik server container. Find the name of the container that's using the ```ghcr.io/goauthentik/server:latest``` image. If your stack isn't running at all, run ```sudo docker logs <container_name>```. If your stack is running but the UI just isn't accessible, run the same command but add the ```-f``` flag. This will give you a live feed of the logs. 
- You're accessing the wrong URL. Let's breakdown the URL you're supposed to access for initial setup: ```http://<your-host's-ip>:9000/if/flow/initial-setup/```
	- In place of ```<your-host's-ip>```, there are a couple of things you could input. If your host has a web browser that you're working from, you can simply input ```localhost```. If you manage everything from another computer on your network, you need to input the internal IP address of your host. From your host, run ```ifconfig```. Next to ```inet```, you will find your internal IP. 
	- In place of ```9000```, you need to input whatever port you've defined in your ```docker-compose.yml``` file. Refer to [my compose example](https://github.com/lynArk/authentikate-your-cloudflared/blob/main/compose/authentik-compose.yml) for formatting.
	- Keep ```/if/flow/initial-setup/``` exactly as is. This specific path is needed to initially setup Authentik. You will access this URL without this path, only AFTER initial setup. 
- Your firewall settings are blocking traffic over the access port. 
 
### 3. Deploy cloudflared with Docker

Before we set up Authentik, we now need to set up a Cloudflare tunnel. The service that you want to expose as well as Authentik itself, can be securely exposed this way rather than opening up ports on your router. Cloudflare Tunnel's software, cloudflared, is free, can run as a Docker container, adds a layer of a security (though best security practices are still applicable), and also acts as a reverse proxy service!

- a. Open up the Cloudflare Zero Trust dash, and navigate to Networks > Tunnels > Create a tunnel > Select Cloudflared.
- b. Name your tunnel! I simply named mine after the host machine I would be running it on.
- c. Save tunnel.
- d. Select ```Docker``` as your environment. 
- e. On your host, navigate to a folder that you want to store your docker-compose.yml in for cloudflared. This will be your cloudflared project folder. 
- f. Cloudflare suggests using the ```docker run``` command to spin up cloudflared, but I systematically use Docker Compose across my whole system. So that's what I'll explain here. Run ```wget https://github.com/lynArk/authentikate-your-cloudflared/blob/main/compose/cloudflared.yml``` to download my example compose file. Pretty much the only necessary change you need to make, is editing ```<your token>``` by replacing it with the token Cloudflare gives you when you select Docker as your environment. Save changes.
- g. Rename your compose file by running ```mv cloudflared.yml docker-compose.yml```.
- h. Run ```sudo docker compose up -d```. Back on the Cloudflare site, you should almost immediately be able to see a running Connector for your newly created tunnel.
- i. Click Next!

### 4. Expose and secure Authentik

Upon clicking Next for your newly created and successfully connected Cloudflare tunnel, you should be met with the option to create a public hostname for your service. This is what maps services running on your host to a readable URL, accessible to the public. In this step, we are going to expose Authentik so that it's authorizations flows can be used remotely. Then we will secure the Authentik UI.

- a. Choose a Subdomain for Authentik. It may be best to choose a simple and easy to remember subdomain name, like ```auth```, so that you can simply access your Authentik UI at ```auth.exampledomain.com```.
- b. Select your personal domain from the drop down, or type it in manually. 
- c. Leave the Path blank.
- d. Select ```HTTP``` as the Service Type. Authentik will be accessible over HTTPS once exposed (because of Cloudflare provided certificates). But until it is exposed, Authentik is only available over HTTP within your local network. Thus why you are telling your Cloudflare tunnel to find Authentik at a HTTP address.
- e. Type in ```http://<your-host's-ip>:<authentik's port>``` for the Service URL. It should look something like ```192.168.1.69:9000```.
- f. Save tunnel. Completing this automatically creates a CNAME record in your Cloudflare DNS control panel. 
- g. Test that the mapping was successful. Ideally, you'd test this from a computer not connected to your local network, to test remote access. Enter the URL you chose into a browser, and see if it takes you to your Authentik UI. It may not work for a few minutes, while the DNS records are updating. 
- h. Now your Authentik UI is publicly available at a readable URL. We don't want it publicly available without some form of protection through authentication; that is the whole point of this guide after all. The Authentik admin account already has a strong password, so let's add another layer of protection with 2FA. Head to your Authentik UI at it's new URL, but append the ```/if/user/#/library``` path.
- i. Login as ```akadmin```.
- j. Go to Settings > MFA Devices > Enroll > TOTP Device. We're going to use a time-based one-time password (TOTP) authentication app. [Check out this guide for recommendations on 2FA apps.](https://www.privacyguides.org/en/multi-factor-authentication/)
- k. In your 2FA app, scan the QR code on your computer screen to finish setting up.
- l. Enter the code given to you by your app, into Authentik.
- m. Click Continue. 

### 5. Add a Cloudflare origin server certificate to Authentik

The last thing you need to do before creating your first Authentik Application/Provider, is add a Cloudflare certificate to Authentik. All of your Providers will use it, and it's a feature included in Cloudflare's free tier. 

- a. Head to the Cloudflare dashboard for your custom domain.
- b. Navigate to SSL/TLS > Origin Server > Create Certificate. Creating a certificate with the default settings will use a Cloudflare-generated key, encrypted with RSA. The certificate will protect your top level domain as well as any sub domains, and last for 15 years.
- c. Copy and save the Origin Certificate and Private Key in a _**secure place**_.
- d. Now login into your Authentik UI as ```akadmin```. Head to System > Certificates > Create.
- e. Make the name for your cert something simple like ```cloudflare``` so you know what it's for. 
- f. Input the text of the certificate you just created in Cloudflare, into the appropriate spots.
- g. Click Create. 

### 6. Create the Application and Provider for your service in Authentik

Authentik is ready to protect it's first application! Generally speaking in Authentik, you have 1 Application/Provider combination per 1 service that you are protecting. Let's setup your first one. 

- a. Still in the Authentik UI, click on the Applications tab, and then "Create With Wizard".
- b. Enter a name for this application. For simplicity, I recommend naming this after the service you're trying to expose.
- c. Again for simplicity, I recommend keeping the default value that populates for the Slug. 
- d. Set the policy engine mode to ```any```.
- e. Select ```OAuth2/OIDC``` as your Provider Type.
- f. Select ```default-provider-authorization-explicit-consent``` as your Authorization flow. Custom authorization flows can be created in the future for a more customized setup, but this guide is only detailing how to get started with Authentik + cloudflared. 
- g. Select ```Confidential``` as the Client type.
- h. Input ```https://<your Zero Trust team domain>.cloudflareaccess.com/cdn-cgi/access/callback``` as your Redirect URls/Origins . Your Zero Trust team domain may be different from your custom domain. You set it up upon initially making your Cloudflare Zero Trust account, and can find it in the Zero Trust dash under Settings > Custom Pages > Team domain. 
- i. Select the certificate you just inputted into Authentik for your Signing Key.
- j. Click Submit.

### 7. Add the Authentik Provider to your Cloudflare Zero Trust authentication settings

Next, we need to add the Authentik Provider you just created, as a login method for Cloudflare Zero Trust. You'll be copying values between Cloudflare and Authentik for this step. If this is not already the case, make sure you keep both the Cloudflare Zero Trust dash and the Authentik UI open in your browser. 

- a. In the Cloudflare Zero Trust dash, head to Settings > Authentication > Login methods > Add new > OpenID Connect. 
- b. Choose a name for this login method. For simplicity, I name my Authentik login methods ```authentik <app name>```. This name makes it very clear what the login method is for. 
- c. In the Authentik UI, go to Applications > Providers, and select the Provider that you just created with the Application/Provider creation wizard.
- d. Now, copy the following values and paste them in the appropriate places from the Authentik UI to the Cloudflare dash:
	- Client ID > App ID
	- Client Secret > Client secret
	- Authorize URL > Auth URL
	- Token URL > Token URL
	- JWKS URL > Certificate URL
- e. Click Save.

### 8. Expose and secure your service

Now that you have Authentik set up and ready for your first service, you can expose whatever service you want and immediately put it behind authentication. 

- a. In the Cloudflare Zero Trust dash, navigate to Networks > Tunnels > click the tunnel you created earlier > Edit > Public Hostnames > Add a public hostname. 
- b. To expose your chosen service via cloudflared, follow [steps 4a-4g](https://github.com/lynArk/authentikate-your-cloudflared?tab=readme-ov-file#4-expose-and-secure-authentik) in this guide. Of course, change values to reflect the service that you want to expose, rather than again entering the details for Authentik. I'll use my BookStack instance mentioned in [step 1](https://github.com/lynArk/authentikate-your-cloudflared?tab=readme-ov-file#1-verify-service-functionality) as an example. Internally on my network, BookStack is available at ```http://192.168.1.69:6875```. I've configured cloudflared to expose the service from that IP, to a public hostname known as ```bookstack.exampledomain.com```. 
- c. With your service now publicly available through a Cloudflare tunnel, let's now protect it with Authentik. Still in the Cloudflare Zero Trust dash, navigate to Access > Applications > Add an application > Self-hosted. 
- d. Chose an application name. I simply choose the name of the service that I'm hosting.
- e. Select a Session Duration, that defines how long a user will have access to your service before having to re-authenticate. The default 24 hours is a fine option.
- f. Enter the subdomain and domain that you configured in [step 8b](https://github.com/lynArk/authentikate-your-cloudflared?tab=readme-ov-file#8-expose-and-secure-your-service).
- g. Un-toggle ```Accept all available identity providers```.
- h. Select only the login method you defined in [step 7](https://github.com/lynArk/authentikate-your-cloudflared?tab=readme-ov-file#7-add-the-authentik-provider-to-your-cloudflare-zero-trust-authentication-settings).
- i. Toggle ```Allow users to skip identity provider selection when only one login method is available.```.
- j. Click Next.
- k. Choose a policy name. 
- l. Under the Configure rules section, choose ```Login Methods``` as your Selector. Choose the Authentik login method that we created for this application, as the Value.
- m. Click Next.
- n. Toggle ```Bypass options requests to origin```.
- o. Click Add application.
- p. Test your setup. Head to the URL for your service, and see if you are faced with the Authentik login page before you can access your service. 

#### 🎉🎊 Congrats! 🎊🎉
You are up and running with Authentik. Repeat steps 6-8 for any additional services you want to expose and protect.

## What now?

What I've described here, despite it's length, is something of a quick-start guide. In that, following the steps will get you only a basic setup. There is much more you can do with Authentik and Cloudflare Zero Trust to improve both your security and user experience. Here are a few things I suggest looking into next. 

### Creating non-admin users within Authentik
[Official Documentation](https://docs.goauthentik.io/docs/users-sources/user/)

You should create non-admin users within Authentik, for yourself and any clients. Better yet if all of these users also use 2FA for authentication. This is best practice, even if your only Authentik client is yourself.

### Customizing your Authentik installation
[Official Documentation](https://docs.goauthentik.io/docs/install-config/configuration/)

The Authentik installation that I walk through in this guide is a bit bare-bones. Check out the docs for greater customization!

### App integration
[Official Documentation](https://docs.goauthentik.io/integrations/)

Certain services have built-in integrations with Authentik that allow for a more seamless experience. It's worth looking into these specific integrations and setting them up. For services that do not have this built-in integration, there is sometimes still an option to disable the service's built-in authentication/login. This would prevent you and your users from having to double log in: one time through Authentik, and another time through the service itself. 

### Custom authorization flows
[Official Documentation](https://docs.goauthentik.io/docs/add-secure-apps/flows-stages/flow/#create-a-custom-flow)

Learn about Authentik authorization flows and creating a custom flow, in order to harness more granular control over your security.

### Monitoring and logging
[Official Authentik Documentation](https://docs.goauthentik.io/docs/sys-mgmt/events/) | [Official Cloudflare Documentation](https://developers.cloudflare.com/cloudflare-one/insights/logs/audit-logs/#per-request-audit-logs)

There are many ways to monitor your setup, at many steps of the process. At the Authentik level, you can monitor login attempts, in the  "Events" logging system within Authentik. It records details about every login attempt, including successful and failed logins, allowing you to view information like username, IP address, and timestamps for each attempt.

At the Cloudflare level, there are several ways to monitor traffic. One key way relevant to this guide's setup, is monitoring access attempts to your Zero Trust Applications. 

### Custom CSS
[Official Documentation](https://docs.goauthentik.io/docs/customize/interfaces/user/customization#custom-css)

You can customize your Authentik login portal with your own CSS! Here's a [cool example template](https://github.com/goauthentik/authentik/discussions/4831) to get you started!
 
### Cloudflare Domain WAF Custom Rules
[Official Documentation](https://developers.cloudflare.com/waf/)

You can use Cloudflare, domain-based, Web Application Firewall (WAF) Custom Rules to add a little more security to your publicly exposed services. You can make 5 of these rules with Cloudflare's free tier. 

As an example, I personally only expect my clients to access my services from within the US. Therefore, I have created a WAF Custom Rule that blocks all non-US traffic.

<img src="https://github.com/lynArk/authentikate-your-cloudflared/blob/main/img/non-us.png?raw=true" alt="example-waf-rule"/>

## [License](https://github.com/lynArk/authentikate-your-cloudflared/blob/main/LICENSE)

<img src="https://github.com/lynArk/authentikate-your-cloudflared/blob/main/img/mit-license.png?raw=true" alt="mit-license" height="150"/>
