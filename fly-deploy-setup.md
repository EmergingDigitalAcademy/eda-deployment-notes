# Fly Deploy Setup

We can deploy apps to fly.io and take advantage of their free tier. By default the Free Tier allows a user to run 3 apps (using 256mb micro virtual machines).

![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/18b9869b-44c8-4c91-be31-0ff0728b88f7)

Care must be taken to ensure that:
  - Apps are deployed to a single machine. By default Fly will spin up TWO machines per app for high availability.
  - Apps are deployed to a micro virtual machine with 256mb of RAM. This ensures that hosting costs will be < $5/mo

On your dashboard, always ensure that there are no more than 3 machines listed and all are 256mb. This way your hosting fees will always be $0.

## Initial Setup

1. [Create an account](https://fly.io/app/sign-up) on fly.io
2. Login and add your Credit Card to activate your account. There is $0 charge unless you break the rules above.
3. Install the Command Line Tools by [following the instructions](https://fly.io/docs/hands-on/install-flyctl/) listed on the website.
4. Authenticate on your laptop, and you're good to go: `fly auth login` and `fly auth whoami` to verify.

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

For projects that are using a database, ensure that your `pool.js` can account for external database configuration via an environment
variable called `DATABASE_URL`. You are responsible for making this environment variable using `fly secrets` in step 4.

pool.js
``` javascript
const pg = require('pg');
let pool;

// Example of what DATABASE_URL could look like (you get this from your db provider)
// DATABASE_URL=postgresql://jDoe354:secretPw123@some.db.com/db_name?sslmode=require
if (process.env.DATABASE_URL) {
    pool = new pg.Pool({
        connectionString: process.env.DATABASE_URL,
        ssl: { rejectUnauthorized: false }
    });
} else {
    pool = new pg.Pool({
        host: 'localhost',
        port: 5432,
        database: 'my_database',
    });
}
pool.on('connect', () => console.log(`Connected to database'));
pool.on(error, (err) => console.error(`Error connecting to database: `, err));

module.exports = pool;
```

## Step 2: Project Setup

To deploy your project, you will need two files *in the root directory of your app*: 
  - `fly.toml` which will tell Fly what kind of infrastructure your app needs (vm size, extra services, etc). This file also configures
    the virtual network your app will run on, including what PORT will be opened for network traffic and how to set up `process.env.PORT` for your app to use. The only thing you need to edit is your app name, which will tell fly to make your app available at `yourappname.fly.dev`
  - `Dockerfile` which tells fly how to run your app:
    - Specifies that `nodejs` should be installed
    - Specifies that `npm install` should be ran to install package dependencies
    - Specifies that `npm start` should be the last command to run to boot the app up.
  - For apps that **use React**, we also need to add a line to ensure `npm run build` will properly run before `npm start`

You can generate these files with `fly launch` or use the files below below so that you don't have to worry about bad defaults.

This file opens port 8080. ***IMPORTANT***: Update line 5 to an appropriate app name (see comment below).
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

Dockerfile (for base app, no react)
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
#RUN npm run build 2>&1

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

Dockerfile (for react apps)
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
RUN npm run build

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

## Step 3: Deploy Your App

The first time you deploy your app, fly will configure a fixed number of machines. These can be manually changed later, but it's nice 
to get it right the first time. As mentioned before, fly by default wills pin up TWO machines per app which is convenient for redundancy because
if one machines goes down, the other will pick up the slack. However, this will take up 2 of your 3 free machines.

To deploy for the first time, and disable high availability, run the following command. For future deploys, you can run the 
same command or just `fly deploy`. 

```
fly deploy --ha=false
```

You can use `fly logs` to see your server console output and in general monitor the server activity.

After a successful deploy, open your URL and see what it looks like! Also view your dashboard and ensure that you only have 1 machine
launched for your app. **Please note that fly does create and tear down a temporary 'builder' machine that assists in the deploy process**
You may see this app for a period of time BUT: a) it is a special machine type that is free and b) it destroys itself after a few minutes

There are several issues that could go wrong:
  - If your `server.js` is not using `process.env.PORT`, it may try to listen on `5000` which likely won't be correct
  - If your `fly.toml` file is missing the PORT environment variable, your server may not be able to listen on the right port
  - If your database is not set up properly, or you are missing `process.env.DATABASE_URL`, the app won't be able to run properly.

## Step 4: Cloud Database Setup (if applicable)

To get a database set up in the cloud, there are a few steps. First ensure that you have an account at Neon, which is where we will host our small databases for free.
  1. Login and create a new database, ensuring `pooled` is enabled for the connections.
  2. Create your tables and seed any necessary data. You can use the build-in web interface on neon.tech or connect via postico by creating a new server connection and pasting the hostname, username, password, port, and database name.
  3. Copy the connection string into a new environment variable secret called `DATABASE_URL` on your fly account so that your app can find it:
     ```
     $ fly secrets set DATABASE_URL=postgresql://jDoe354:secretPw123@some.db.com/db_name?sslmode=require
     ```
     and ensure that the secret is available and there are no typos :)
     ```
     $ fly secrets list
     ```

That's it! Monitory your logs with `fly logs` to see if your app is able to read the environment variable and connect properly. To debug, you can drop a `console.log(`Connection String:`, process.env.DATABASE_URL);` to your `pool.js` to verify that the environment variable is set up properly.
