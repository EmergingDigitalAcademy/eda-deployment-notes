# Deploying your App to the cloud (fly.io)
## For apps with no database, with or without react

We can deploy apps to fly.io and take advantage of their free tier. By default the Free Tier allows a user to run 3 apps (using 256mb micro virtual machines).

![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/18b9869b-44c8-4c91-be31-0ff0728b88f7)

Care must be taken to ensure that:
  - Apps are deployed to a single machine. By default Fly will spin up TWO machines per app for high availability.
  - Apps are deployed to a micro virtual machine with 256mb of RAM. This ensures that hosting costs will be < $5/mo

On your dashboard, always ensure that there are no more than 3 machines listed and all are 256mb. This way your hosting fees will always be $0.

### Summary of Steps:
  1. Ensure `server.js` and `package.json` are good to go.
  2. Copy `fly.toml` and `Dockerfile` into your project
  3. Change line 5 in `fly.toml`: `app = "yourinitials-projectname"`
  4. `fly launch --ha=false` ('Y' to copy config, 'N' to tweak, 'Y' to .dockerignore)
  5. `fly scale count 1` to confirm only 1 machine
  6. `fly deploy` (for all future deploys)

## Step 0: Initial Account Setup

1. [Create an account on fly.io](https://fly.io/app/sign-up) on fly.io
2. Login and add your Credit Card to activate your account. There is $0 charge unless you break the rules above.
3. Install the Command Line Tools by [following the instructions](https://fly.io/docs/hands-on/install-flyctl/) listed on the website.
4. Authenticate on your laptop, and you're good to go: `fly auth login` and `fly auth whoami` to verify.

For projects that are using a database, see [./fly-deploy-postgres.md](./fly-deploy-postgres.md)

## Step 1: Make Your App Deploy Ready

The following needs to be accounted for in your project to make your app "deploy ready".

Ensure that `server.js` accounts for a PORT environment variable. This is a very typical way to let your server run on port 5000
on your machine, or allow for that default to be overriden by providing an alternative port as an environment variable:
``` javascript
const PORT = process.env.PORT || 5000;
```

Ensure that `package.json` is properly configured for `npm start`:
``` json
...
  "scripts": {
    "start": "node server/server.js"
  }
...
````

For projects that are using a database, see [./fly-deploy-postgres.md](./fly-deploy-postgres.md)

## Step 2: Cloud Project Setup

To deploy your project, you will need two files *in the root directory of your app*: 
  - `fly.toml` which will tell Fly what kind of infrastructure your app needs (vm size, extra services, etc). This file also configures
    the virtual network your app will run on, including what PORT will be opened for network traffic and how to set up `process.env.PORT` for your app to use. The only thing you need to edit is your app name, which will tell fly to make your app available at `yourappname.fly.dev`
  - `Dockerfile` which tells fly how to run your app:
    - Specifies that `nodejs` should be installed
    - Specifies that `npm install` should be ran to install package dependencies
    - Specifies that `npm start` should be the last command to run to boot the app up.
  - For apps that **use React**, we also need to add a line to ensure `npm run build` will properly run before `npm start`

### Action: Update `fly.toml` with an appropriate app name
***IMPORTANT***: Update line 5 to an appropriate app name (see comment below).
fly.toml
```
# fly.toml app configuration file generated
# See https://fly.io/docs/reference/configuration/ for information about how to use this file.

#TODO: Change this to your project name and github handle, like app = "booherbg-todo-app"
app = ""
primary_region = "ord"

[build]

[env]
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 0
  processes = ["app"]

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory_mb = 256

```

Dockerfile (for base app with or without react)
```
# syntax = docker/dockerfile:1

# Adjust NODE_VERSION as desired
ARG NODE_VERSION=20.6.0
FROM node:${NODE_VERSION}-slim as base

LABEL fly_launch_runtime="Node.js"

# Node.js app lives here
WORKDIR /app

# Set production environment
ENV NODE_ENV="production"


# Throw-away build stage to reduce size of final image
FROM base as build

# Install packages needed to build node modules
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential node-gyp pkg-config python-is-python3

# Install node modules
COPY --link package-lock.json package.json ./
RUN npm ci

# Run the build stage (mostly for react)
RUN if [ -f src/index.js ]; then npm run build; fi

# Copy application code
COPY --link . .

# Final stage for app image
FROM base

# Copy built application
COPY --from=build /app /app

# Start the server by default, this can be overwritten at runtime
EXPOSE 8080
CMD [ "npm", "run", "start" ]
```

### Action: Create the Cloud App (deploy)

Now that we have our fly config file and Dockerfile, we can provision the resources on fly's platform.

The first time you launch your app, fly will deploy for you. As mentioned before, fly by default wills pin up TWO machines per app which is convenient for redundancy because if one machines goes down, the other will pick up the slack. However, this will take up 2 of your 3 free machines and is not necessary for small projects. Run `fly scale count 1` to scale down to 1 machine if needed.

Verify that your `fly.toml` and `Dockerfile` look OK and have not been overwritten with new defaults. Check the reference file above to ensure the ports are still 8080 and your settings look OK.

Run these commands, and verify that everything looks correct (including the VM size of 256mb)
```
# Answer 'Y' to copying the configuration into the new app
# Answer 'N' to needing to tweaking the settings
# Answer 'Y' to creating a .dockerignore file
fly launch --ha=false
```
![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/94e8f142-26ba-41a3-b68b-2ba96af67525)

That's it! Fly will deploy your app for you the first time. Use `fly open` to launch your browser, and `fly logs` to debug.

## Step 3: How to deploy your app

Your app is probably deployed already. To deploy again:

```
fly deploy
```

It's a good idea to make sure you only have 1 machine. You can do this manually (below) or just run `fly scale count 1` to force the app to have only 1 machine.

```
fly machines list                 (to verify there are two machines)
```

To scale down to 1 machine:
```
fly scale count 1
```

Alternatively, you can pick the machine to destroy:
```
fly machines list -q                 (to verify there are two machines)
fly destroy --force the_machine_id   (just pick one of the machine IDs to destroy)
```
![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/3b733ca9-5229-4ef7-809b-28676128d905)

You can use `fly logs` to see your server console output and in general monitor the server activity.

After a successful deploy, open your URL and see what it looks like! Also check out your dashboard and ensure that you only have 1 machine
launched for your app. **Please note that fly does create and tear down a temporary 'builder' machine that assists in the deploy process**
You may see this app for a period of time BUT: a) it is a special machine type that is free and b) it destroys itself after a few minutes

There are several issues that could go wrong:
  - If your `server.js` is not using `process.env.PORT`, it may try to listen on `5000` which likely won't be correct
  - If your `fly.toml` file is missing the PORT environment variable, your server may not be able to listen on the right port
