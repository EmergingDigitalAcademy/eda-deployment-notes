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
```
const PORT = process.env.PORT || 5000;
```

Ensure that `package.json` is properly configured for `npm start`:
```
...
  "scripts": {
    "start": "node server/server.js"
  }
...
````

For projects that are using a database, ensure that your `pool.js` can account for external database configuration via an environment
variable called `DATABASE_URL`. You are responsible for making this environment variable using `fly secrets` in step 4.

pool.js
```
const pg = require('pg');
let pool;

// When our app is deployed to the internet 
// we'll use the DATABASE_URL environment variable
// to set the connection info: web address, username/password, db name
// eg: 
//  DATABASE_URL=postgresql://jDoe354:secretPw123@some.db.com/prime_app
if (process.env.DATABASE_URL) {
    pool = new pg.Pool({
        connectionString: process.env.DATABASE_URL,
        ssl: {
            rejectUnauthorized: false
        }
    });
}
// When we're running this app on our own computer
// we'll connect to the postgres database that is 
// also running on our computer (localhost)
else {
    let databaseName = 'prime_feedback'
    
    if (process.env.NODE_ENV === 'test') {
      databaseName = 'prime_testing'
    }

    pool = new pg.Pool({
        host: 'localhost',
        port: 5432,
        database: databaseName, 
    });
}

module.exports = pool;
```

## Project Steup: Step 2

To deploy your project, you will need two files *in the root directory of your app*: 
  - `fly.toml` which will tell Fly what kind of infrastructure your app needs (vm size, extra services, etc). This file also configures
    the virtual network your app will run on, including what PORT will be opened for network traffic and how to set up `process.env.PORT` for your app to use.
  - `Dockerfile` which tells fly how to run your app:
    - Specifies that `nodejs` should be installed
    - Specifies that `npm install` should be ran to install package dependencies
    - Specifies that `npm start` should be the last command to run to boot the app up.
  - For apps that **use React**, we also need to add a line to ensure `npm run build` will properly run before `npm start`
  - For apps taht **use a database**, we also need to ensure that an environment variable (like `process.env.DATABASE_URL`) is wired up as
  - a projec secret so it can be accessible to your `pool.js` file.

You can generate these files with `fly launch` or use the files below below so that you don't have to worry about bad defaults.

fly.toml (use this for all node apps; opens port 8080):
```

```

Dockerfile (for base app, no react)
```
```

Dockerfile (for react apps)
```
```

## Deploy your app: Step 2

The first time you deploy your app, fly will configure a fixed number of machines. These can be manually changed later, but it's nice 
to get it right the first time. As mentioned before, fly by default wills pin up TWO machines per app which is convenient for redundancy because
if one machines goes down, the other will pick up the slack. However, this will take up 2 of your 3 free machines.

To deploy for the first time, and disable high availability, run the following command. For future deploys, you can run the 
same command or just `fly deploy`. 

```
fly deploy --ha=false
```

After a successful deploy, open your URL and see what it looks like! Also view your dashboard and ensure that you only have 1 machine
launched for your app. **Please note that fly does create and tear down a temporary 'builder' machine that assists in the deploy process**
You may see this app for a period of time BUT: a) it is a special machine type that is free and b) it destroys itself after a few minutes

There are several issues that could go wrong:
  - If your `server.js` is not using `process.env.PORT`, it may try to listen on `5000` which likely won't be correct
  - If your `fly.toml` file is missing the PORT environment variable, your server may not be able to listen on the right port

