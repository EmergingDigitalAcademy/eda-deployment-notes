# Deploying your App to the cloud (fly.io)
## For apps with a database, with or without react

We can deploy apps to fly.io and take advantage of their free tier. By default the Free Tier allows a user to run 3 apps (using 256mb micro virtual machines).

![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/18b9869b-44c8-4c91-be31-0ff0728b88f7)

Care must be taken to ensure that:
  - Apps are deployed to a single machine. By default Fly will spin up TWO machines per app for high availability.
  - Apps are deployed to a micro virtual machine with 256mb of RAM. This ensures that hosting costs will be < $5/mo
  - Apps are deployed *without* a database machine, since we're using the free tier of neon.tech and do not need fly to provision us a postgres machine (which counts against our free tier)

On your dashboard, always ensure that there are no more than 3 machines listed and all are 256mb. This way your hosting fees will always be $0.

### Summary of Steps:
  1. Ensure [`server.js`](#step-1-make-your-app-deploy-ready), [`package.json`](#step-1-make-your-app-deploy-ready), and [`pool.js`](#action-update-your-pooljs-to-account-for-a-cloud-database) are good to go.
  2. Copy [`fly.toml`](#action-update-flytoml-with-an-appropriate-app-name), [`Dockerfile`](#action-create-your-dockerfile), [`.dockerignore`](#action-create-your-dockerignore-file) into your project
  3. Change line 5 in `fly.toml` to a unique app namespace, like your initials and project name `app = "githubhandle-projectname"` or `app = "booherbg-todo-app"`
  4. [`fly launch --vm-size=shared-cpu-1x`](#action-create-the-cloud-app)
     - 'Y' to copy config
     - 'Y' to tweak - set postgres to `none` (***IMPORTANT***)
     - 'Y' for .dockerfile)
  6. With React: [`npm run build && fly deploy --ha=false`](#step-3-deploy-your-app)
  7. Without React: [`npm run build && fly deploy --ha=false`](#step-3-deploy-your-app)
  8. [Create database tables](#step-4-cloud-database-setup-if-applicable) in your project at neon.tech dashboard
  9. [Copy connection string](#action-create-your-database_url-environment-variable), then `fly secrets set DATABASE_URL=...` in your project

To deploy code changes after : `fly deploy` (no react) or `npm run build && fly deploy` (with react)

## Step 0: Initial Account Setup

1. [Create an account on fly.io](https://fly.io/app/sign-up) on fly.io
2. Login and add your Credit Card to activate your account. There is $0 charge unless you break the rules above.
3. Install the Command Line Tools by [following the instructions](https://fly.io/docs/hands-on/install-flyctl/) listed on the website.
4. Authenticate on your laptop, and you're good to go: `fly auth login` and `fly auth whoami` to verify.

For apps with a database, create your neon.tech account:
1. [Create an account on neon.tech](https://console.neon.tech/signup)
2. Verify your email, etc. 
3. Login and create a project, you can just call it `eda-apps`. That should be all you need.
4. You can create a database now or later. All databases will be created under the default project and can be deleted at any time.

## Step 1: Make Your App Deploy Ready

The following needs to be accounted for in your project to make your app "deploy ready".

Ensure that `server.js` accounts for a PORT environment variable. This is a very typical way to let your server run on port 5000
on your machine, or allow for that default to be overriden by providing an alternative port as an environment variable:
``` javascript
const PORT = process.env.PORT || 5000;
```

Double check that express is configured to serve our a `build/` folder:
```
app.use(express.static('build'));
```

Ensure that `package.json` is properly configured for `npm start`. The `"deploy"` script
is optional but allows you to run `npm run deploy` may come in handy later as it combines two commands.

``` json
...
  "scripts": {
    "start": "node server/server.js",  (note: make sure vite is not referenced here)
    "deploy": "npm run build && fly deploy --ha=false"
  }
...
````

### Action: Update your `pool.js` to account for a cloud database

For projects that are using a database, ensure that your `pool.js` can account for external database configuration via an environment
variable called `DATABASE_URL`. You are responsible for making this environment variable using `fly secrets` in step 4.

Here's a `pool.js` you can use. It looks for the `process.env.DATABASE_URL` environment variable, and uses it if found. You don't need to edit this file, it's good to go.
pool.js
``` javascript
const pg = require('pg');
let pool;

// Example of what DATABASE_URL could look like (you get this from your db provider)
// DATABASE_URL=postgresql://jDoe354:secretPw123@some.db.com/db_name?sslmode=require
if (process.env.DATABASE_URL) {
  console.log(`Using cloud database config (DATABASE_URL found)`);
  pool = new pg.Pool({
    connectionString: process.env.DATABASE_URL,
    ssl: { rejectUnauthorized: false }
  });
} else {
  console.log(`Using local database config (no DATABASE_URL found)`);

  pool = new pg.Pool({
    host: 'localhost',
    port: 5432,
    database: 'my-database-name',
  });
}
pool.on('connect', () => console.log(`Connected to database`));
pool.on('error', (err) => console.error(`Error connecting to database:`, err));

module.exports = pool;
```

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

### Action: Create your Dockerfile

Create a `Dockerfile` in your root project directory. This file contains the instructions
that fly will use to run our app (installs NodeJS, installs our node packages, runs the project, etc).

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

### Action: Create a .dockerignore file

This ensures the `build/` folder is not ignored

.dockerignore
```
# flyctl launch added from .gitignore
# See https://help.github.com/ignore-files/ for more about ignoring files.
# cypress Snapshots
cypress/screenshots

# dependencies
node_modules

# testing
coverage

# production

# misc
**/.DS_Store
**/.env.local
**/.env.development.local
**/.env.test.local
**/.env.production.local

**/npm-debug.log*
**/yarn-debug.log*
**/yarn-error.log*

**/.eslintcache
fly.toml
```

### Action: Create the Cloud App

Now that we have our `fly.toml` config file and `Dockerfile`, we can provision the resources on fly's platform.

Note that as of right now, there is no way to ask fly to NOT create a separate dedicated cloud database machine for us. Ensure that fly will skip this step by following the instructions below.

Run this command, and verify that everything looks correct:
  - `app name` is updated to be specific to your project
  - `vm size` is 256mb shared-cpu-1x
  - `database` is set to `none`
```
# Answer 'Y' to copying the configuration into the new app
# Answer 'Y' to needing to tweaking the settings
# Set the 'postgres' provisioning to 'none' -- we do not need to provision a database
# Answer 'Y' to creating a .dockerignore file
fly launch --vm-size=shared-cpu-1x
```
![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/ac6c9d59-abdf-42ae-8b9b-e087a32a2c6e)

Verify that your `fly.toml` and `Dockerfile` look OK and have not been overwritten with new defaults. Check the reference file above to ensure the ports are still 8080 and your settings look OK.

## Step 3: Deploy Your App

The first time you deploy your app, fly will configure a fixed number of machines that can be manually changed later, but it's nice to get it right the first time. As mentioned before, fly by default wills pin up TWO machines per app which is convenient for redundancy because if one machines goes down, the other will pick up the slack. However, this will take up 2 of your 3 free machines and is not necessary for small projects.

To deploy for the first time, and disable high availability, run the following command. You should run `npm run build` before every deploy so that you ensure that your react app is up to date.

```
#Without React:
fly deploy --ha=false

#With React:
npm run build && fly deploy --ha=false
```

If you forget `--ha=false`, you may end up with 2 machines for the app. To fix this just destroy one of them:

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
  - If your database is not set up properly, or you are missing `process.env.DATABASE_URL`, the app won't be able to run properly.

## Step 4: Cloud Database Setup (if applicable)

To get a database set up in the cloud, there are a few steps. Make sure you followed the initial steps to create an account on neon.tech
  1. On the neon.tech website, create a new database. Check the box for 'pooled' connections. This text in the box will be needed later (this is your connection string that will go in process.env.DATABASE_URL). You can retrieve it later too.
     ![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/2aae1b25-1b77-40f1-9485-e5eb97a6a93f)

  3. On the neon.tech dashboard, click 'SQL Editor'. Ensure your database is selected, and run your `CREATE TABLE` statements along with any `INSERT` statements you need for initial seeded data. You can highlight individual queries to run. You can also use postico for this step.
     ![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/123f581a-d0c1-4bd1-a902-89de85d34b32)

  4. Click the 'Tables' tab and verify that your tables and relevant data are there.
     ![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/51198d89-255c-4c2d-876a-0791e763a9f0)

### Action: Create your `DATABASE_URL` environment variable 
On the dashboard, select your database and copy the connection string.
     ![image](https://github.com/EmergingDigitalAcademy/eda-deployment-notes/assets/159698/7f44b3b2-059c-4424-9144-42ea718d241d)

Back in your project, create a new environment variable secret called `DATABASE_URL` and paste in your connection string. Take care
     to ensure that there are no typos, extra spaces, etc. This is the link between your project and your database. Without this
     step, `process.env.DATABASE_URL` will not exist.
     
   ```
   $ fly secrets set DATABASE_URL="postgresql://jDoe354:secretPw123@some.db.com/db_name?sslmode=require"
   ```

   Double check to be sure that the secret is available and there are no typos :)
   ```
   $ fly secrets list
   ```

That's it! Monitory your logs with `fly logs` to see if your app is able to read the environment variable and connect properly. To debug, you can drop a `console.log(`Connection String:`, process.env.DATABASE_URL);` to your `pool.js` to verify that the environment variable is set up properly.

To deploy database changes, simply store any `ALTER TABLE` statements as timestamped files in your project (like in a `database_changes` folder) and make sure they all get ran on your production database when you deploy, so that your production database is always up to date. Doing it this way ensures that you do not need to drop your database and re-create it every time you deploy (which destroys all data).

To start over, use the `DROP DATABASE` command: `DROP DATABASE TODO_LIST`
