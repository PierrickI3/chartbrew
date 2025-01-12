---

canonicalUrl: https://docs.chartbrew.com/deployment/

meta: 
    - property: og:url
      content: https://docs.chartbrew.com/deployment/

---

# Deploy Chartbrew on your server

::: tip INFO
With time, this guide will be updated with more ways of deploying. Any PR is welcome!
:::

::: warning WARNING
**PostgreSQL** is not yet tested in production. If you try it out and it doesn't work, please [open an issue on GitHub](https://github.com/chartbrew/chartbrew/issues) and attach any info that can help debug the problem.
:::

## Building the app

Follow the [Getting Started](../#getting-started) guide to set up the project somewhere on your server.

Please note that you have to set all the production environmental variables in this case.

Keep in mind that the tools used in this guide are used as an example but they work really well. There might be multiple other tools that you can use, so feel free to do so if you wish.

## Serving the app

This guide is tried and tested with Ubuntu. The settings should be the same on all operating systems, but some of the files and commands may vary.

### Backend

The backend doesn't need to undergo a build process. You need the following things to make sure it can be served:

- MySQL running
- A database is created so that Chartbrew can use it (it must be empty if it wasn't used by Chartbrew before)
- `npm install` ran in the server folder or `npm run setup` in the project root folder
- Environmental variables are set for Production (check `.env-template` in the root folder to see which)

Once all these are checked, we can either run `npm run start`, or use an external tool like [`pm2`](https://pm2.keymetrics.io) to monitor and manage Node apps better.

```sh
npm install -g pm2

# starting the app in production
cd chartbrew/server
NODE_ENV=production pm2 start index.js --name "chartbrew-server"
```

You can also start [pm2 as a cluster](https://pm2.keymetrics.io/docs/usage/cluster-mode/) to take advantage of your server's multi-threading capabilities and 0-downtime reloads. If you wish to do this, replace the last command above with:

```sh
NODE_ENV=production pm2 start index.js --name "chartbrew-server" -i max
```

This will then run the server on the `port` specified in `server/settings.js`.

### Frontend

The Frontend app needs to be built and then served using [pm2](https://pm2.keymetrics.io) like above.

```sh
# build the app
cd chartbrew/client
mkdir dist && cd dist
vim build.sh
```

Copy the following in the `build.sh` file

```sh
cd ../ && npm run build && cp -rf build/* dist/
```

Make sure the script can be executed after you save the file:

```sh
chmod +x build.sh
```

Create the `pm2` configuration file:

```sh
vim app.config.json
```

```js
{
  apps : [
    {
      name: "cbc-client",
      script: "npx",
      interpreter: "none",
      args: "serve -s . -p 5100"
    }
  ]
}
```

Now you can start the front-end app with `pm2 start app.config.json`

Using the methods above, the app can be accessed on localhost. To serve it on a domain, continue reading on.

## Serving on a domain with Apache

As you already guessed, you will need Apache for this one. Below is a list of guides I found on the Internet on how to install it on different systems:

* [Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-18-04)
* [Fedora 30/29/28](https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-18-04)
* [Windows - using Xampp utility](https://www.apachefriends.org/download.html)
* [MacOS](https://tecadmin.net/install-apache-macos-homebrew/)

::: tip NOTE
This guide is made for Apache running on Ubuntu 16. It might be different on other operating systems and distros.
:::

Create a new Apache configuration file for the Chartbrew site.

```sh
sudo vim /etc/apache2/sites-available/chartbrew.conf
```

This configuration file will have everything necessary to serve the backend and frontend on your domains. Since both apps are already running on different ports, we will use a reverse proxy method to serve these on a domain.

```xml
# /etc/apache2/sites-available/chartbrew.conf

### FRONTEND
<VirtualHost *:80>
    # Important! use your own domain here
    ServerName  example.com  

    ProxyRequests off
    <Proxy *>
        Order deny,allow
        Allow from all
    </Proxy>
    <Location />
        ProxyPass http://localhost:5100/
        ProxyPassReverse http://localhost:5100/
    </Location>
    <Location ~ "/chart/*">
      Header always unset X-Frame-Options
    </Location>
    <Location ~ "/b/*">
      Header always unset X-Frame-Options
    </Location>
</VirtualHost>

### BACKEND
<VirtualHost *:80>
    # Important! use your own domain here
    ServerName  api.example.com

    ProxyRequests off
    <Proxy *>
        Order deny,allow
        Allow from all
    </Proxy>
    <Location />
        ProxyPass http://localhost:3210/
        ProxyPassReverse http://localhost:3210/
    </Location>
</VirtualHost>

```

Make sure you type your domain correctly and all the subdomains that you use are registered in your DNS configuration ([Cloudflare example](https://support.cloudflare.com/hc/en-us/articles/360019093151-Managing-DNS-records-in-Cloudflare)). Now activate the site and you will be able to access Chartbrew using your domain:

```sh
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2ensite chartbrew
sudo service apache2 reload
```

If you're making your site public, it's strongly recommended that you enable `https` on it. Some providers offer that automatically, but in case that's not happening have a look at [Certbot](https://certbot.eff.org/instructions) as it's super simple to set up and it's free.

### Troubleshooting the Chart Embedding feature

A problem that might arrise with embedding charts on other website is to do with the `X-Frame-Options` header being set as `deny` by the server. If your charts can't load on other sites because of this problem make sure you add the following configuration to the `VirtualHost` that serves your frontend:

```
<Location ~ "/chart/*">
  Header always unset X-Frame-Options
</Location>
```

This is already added in the example above, but if you create new virtual hosts for `https`, for example, don't forget to add it there as well.

## Run the application with Docker

::: warning WARNING
The setup is not yet updated for PostgreSQL. Please [open a PR](https://github.com/chartbrew/chartbrew/compare) or [a new issue](https://github.com/chartbrew/chartbrew/issues/new) if you have a solution or encounter any problems that can help us with the support.
:::

A [Chartbrew docker image](https://hub.docker.com/r/razvanilin/chartbrew) is built whenever a new version is released. Run the commands below to get started.

For `amd64` architecture:

```sh
docker pull razvanilin/chartbrew
```

For `arm64` architecture:

```sh
docker pull razvanilin/chartbrew:latest-arm64
```

Then:

```sh
docker run -p 3210:3210 -p 3000:3000 \
  -e CB_SECRET=<enter_a_secure_string> \
  -e CB_API_HOST=0.0.0.0 \
  -e CB_API_PORT=3210 \
  -e CB_DB_HOST=host.docker.internal \
  -e CB_DB_NAME=chartbrew \
  -e CB_DB_USERNAME=root \
  -e CB_DB_PASSWORD=password \
  -e REACT_APP_CLIENT_HOST=http://localhost:3000 \
  -e REACT_APP_API_HOST=http://localhost:3210 \
  razvanilin/chartbrew
```

Check `.env-template` in the repository for extra environmental variables to enable `One account` or emailing capabilities.

**Now let's analyse what is needed for the docker image to run properly**.

The `3210` port is used by the API and `3000` for the client app (UI). Feel free to map these to any other ports on your system (e.g `4523:3210`).

* `CB_SECRET` this string will be used to encrypt passwords and tokens. Use [a secure string](https://passwordsgenerator.net/) if you're planning to host the app publicly.

* `CB_API_HOST` needs to point to the home address of the system. Usually for a docker image this is `0.0.0.0`.

* `CB_DB_HOST` is the host of your database and determines how the application can reach it. `host.docker.internal` is used when you want the container to connect to a service on your host such as a database running on your server already. If your MySQL database is running on a different port than `3306`, then specify the port as well: `host.docker.internal:3307`.

* `CB_DB_NAME` the name of the database (make sure the database exists before running the image).

* `CB_DB_USERNAME` and `CB_DB_PASSWORD` are used for authentication with the DB.

* `REACT_APP_CLIENT_HOST` is the address of the client application and is used by the client to be aware of its own address (not as important)

* `REACT_APP_API_HOST` this is used for the client application to know where to make the API requests. This is the address of the API (backend).

If the setup fails in any way, please double-check that the environmental variables are set correctly. Check that both API and Client apps are running, and if you can't get it running, please [open a new issue](https://github.com/chartbrew/chartbrew/issues/new) with as much info as you can share (logs, vars).



