# Kong EE on Heroku

This is a guide on how to deploy Kong Enterprise Edition on Heroku. You will require a [Private Space](https://devcenter.heroku.com/articles/private-spaces) for this as there are multiple services running on multiple ports.

## Setup

### **Step 1:** Checkout this repo.
    
    git clone https://github.com/travega/heroku-kong-ee.git

### **Step 2:** Create a private space. 

See here for a [list of region identifiers](https://devcenter.heroku.com/articles/private-spaces#regions)
    
    heroku spaces:create <my-space-name> --team <my-team-name> --region <space-location>

### **Step 3:** Create your apps.
    
**NGINX:**

    heroku create <my-NGINX-app-name> -r nginx --space <my-space-name>
    
**Kong EE:**

    heroku create <my-KONG-app-name> -r kong --space <my-space-name>

### **Step 4:** Create a Postgre DB for Kong.

You now have 2 app placeholders for NGINX and Kong. We will create a Postgres instance for the Kong App.

    heroku addons:create heroku-postgresql:private-0 -r kong

>PLEAESE NOTE: Some Heroku Addon tiers incur a cost that will come our of your Addon Credit allocation. Please speak with your Heroku Admin before creating any paid addons.

### **Step 5:** Add required envirement variables to the Kong app.

```bash
heroku config:add -r kong DATABASE_URL=<PG_CONNECTION_STRING>     # This will be added automatically in 'Step 4' above
heroku config:add -r kong KONG_ADMIN_LISTEN="0.0.0.0:8001"        # The port on which the admin service is running in Kong
heroku config:add -r kong KONG_DATABASE=postgres                  # We will use 'postgres' here
heroku config:add -r kong KONG_LICENSE_DATA=<JSON_LICENSE_STRING> # Copy and paste from licesnse.json file
heroku config:add -r kong KONG_PG_DATABASE=<POSTGRES_DB>          # This will be after the '/' in the DATABASE_URL
heroku config:add -r kong KONG_PG_HOST=<POSTGRES_HOST>            # The AWS URL string without the port number appended
heroku config:add -r kong KONG_PG_PASSWORD=<POSTGRES_PWD>         # The string preceding '@' in the DATABASE_URL
heroku config:add -r kong KONG_PG_USER=<POSTGRES_USR>             # The string following 'postgres://' and before the ':'
heroku config:add -r kong KONG_PORTAL=on                          # Enable the Kong Portal
heroku config:add -r kong KONG_VITALS=on                          # Health and performance metrics
```

## Deploy

### **Step 1:** Deploy the Kong EE app

We will be deploying Kong EE as a Docker image to a Docker conatiner in Heroku. First we need to set the Heroku stack to run our Docker container.

    heroku container:login -r kong

Now we need to push our container which is defined in the `Docker.kongee` file.

    heroku container:push --recursive -r kong

Finally, we release our container.

    heroku container:release kongee -r kong

### **Step 2:** Deploy the NGINX service

First we will need to add the NGINX buildpack.

    heroku buildpacks:add https://github.com/heroku/heroku-buildpack-nginx -r nginx

For this service all we need to do is a standard Heroku push.

    git push nginx master

### **Step 3:** Configure NGINX

In the `config/nginx.conf.erb` at the bottom is the config for the upstream Kong EE services. 

Replace the `<my-KONG-app-name>` references with the name you gave your app in **Setup** > **Step 3: Create your apps** > **Kong EE** above.

```nginx
upstream upstream_proxy_api {
    server kongee.<my-KONG-app-name>.app.localspace:8000;
}

upstream upstream_admin_api {
    server kongee.<my-KONG-app-name>.app.localspace:8001;
}

upstream upstream_admin_ui {
    server kongee.<my-KONG-app-name>.app.localspace:8002;
}

upstream upstream_dev_portal {
    server kongee.<my-KONG-app-name>.app.localspace:8003;
}
```

> Note: You may need to change your location config if you are getting 404s trying to load css and js assets from the UIa.

E.g. Change this config:
```nginx
location / {
      set $upstream upstream_proxy_api;
      proxy_pass http://$upstream/;
      proxy_set_header Host $http_host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Scheme $scheme;
    }
```
