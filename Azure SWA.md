# Azure Static Web App (Angular & React)

## **1. Create Angular sample app (recommended way)**

Run these commands:

```bash
npm install -g @angular/cli

ng new swa-angular-sample --routing=false --style=css
cd swa-angular-sample
``` 

This creates a basic starter Angular app.

----------

## Modify homepage text so you can confirm deployment is visible

Open:

`src/app/app.component.html` 

Replace content with:

```html
<div style="text-align:center;margin-top:50px;">
  <h1> Angular deployed on Azure Static Web Apps </h1>
  <p>If you see this, deployment and routing works!</p>
</div>

``` 

----------

# **2. Add Azure Static Web Apps configuration (public internet)**

Create file at root:

```
staticwebapp.config.json
```

Content:

```json
{
  "routes": [
    {
      "route": "/*",
      "allowedRoles": ["anonymous"]
    }
  ],
  "navigationFallback": {
    "rewrite": "/index.html"
  }
}

```

 This ensures SPA routing works on Azure.

## Frontdoor 

```json
{
  "routes": [
    {
      "route": "/*",
      "serve": "/index.html",
      "statusCode": 200
    }
  ],

  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": [
      "*.{css,scss,js,png,gif,ico,jpg,svg}"
    ]
  },

  "networking": {
    "allowedIpRanges": [
      "AzureFrontDoor.Backend"
    ]
  },

  "forwardingGateway": {
    "requiredHeaders": {
      "X-Azure-FDID": "123-456-4ba1-825f-ce6cc8170017"
    },
    "allowedForwardedHosts": [
      "intranet-apps.abc-companyt.com"
    ]
  }
}
```

----------

# **3. Build the app**

```bash
ng build --configuration production
```

Output will be created in:

```bash
dist/swa-angular-sample/
``` 

----------

# **4. Deploy to Azure Static Web Apps**

```bash
swa deploy --app-location dist/swa-angular-sample

or

swa deploy --app-location "dist/apps/sample" --env "production" --deployment-token "123-456-b890-48ca-90dc-890"
``` 

----------

# **Full working sample project structure**

```css
swa-angular-sample/
 ├── src/
 │   ├── app/
 │   │   ├── app.component.ts
 │   │   ├── app.component.html
 │   │   └── app.module.ts
 │   ├── index.html
 ├── angular.json
 ├── package.json
 ├── staticwebapp.config.json
``` 

----------

# If you want sample files (copy-paste)

### `app.component.ts`

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  title = 'swa-angular-sample';
}

```

### `app.component.html`

```html
<div style="text-align:center;margin-top:50px;">
  <h1> Angular deployed on Azure Static Web Apps</h1>
  <p>If you see this message, SWA deployment worked!</p>
</div>

```

### `package.json`

```json
{
  "name": "swa-angular-sample",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "watch": "ng build --watch --configuration development",
    "test": "ng test"
  },
  "dependencies": {
    "@angular/animations": "^16.2.0",
    "@angular/common": "^16.2.0",
    "@angular/compiler": "^16.2.0",
    "@angular/core": "^16.2.0",
    "@angular/forms": "^16.2.0",
    "@angular/platform-browser": "^16.2.0",
    "@angular/platform-browser-dynamic": "^16.2.0",
    "@angular/router": "^16.2.0",
    "rxjs": "~7.5.0",
    "tslib": "^2.4.0",
    "zone.js": "~0.13.0"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "^16.2.0",
    "@angular/cli": "^16.2.0",
    "@angular/compiler-cli": "^16.2.0",
    "typescript": "~5.1.0"
  }
}

``` 

### `staticwebapp.config.json`


# **Fix 4 — Your app has route roles enforced**

If your `staticwebapp.config.json` includes something like:

`"allowedRoles":  ["authenticated"]` 

or:

`"allowedRoles":  ["admin"]` 

That will block unauthenticated users.

Change config to:

``` json
{
  "routes": [
    {
      "route": "/*",
      "allowedRoles": ["anonymous"]
    }
  ],
  "navigationFallback": {
    "rewrite": "/index.html"
  }
}
```
Deploy again.

-----

## BUILD Stage (Node 16)

```groovy

stage('Select Node Version for Build') {
    steps {
        bat """
        echo Switching Node to 16.20.2 for Angular build...
        nvm use 16.20.2
        node -v
        npm -v
        """
    }
}
```

## DEPLOY Stage (Node 18)

```groovy

stage('Deploy to Azure SWA') {
    steps {
        bat """
        echo Switching Node to 18.18.2 for SWA deployment...
        nvm use 18.18.2
        node -v
        npm -v

        echo Deploying application...
        swa deploy ./dist ^
            --app-name seat-booking-swa-nonprod ^
            --deployment-token YOUR_DEPLOYMENT_TOKEN ^
            --env production
        """
    }
}
```

### Angular 15 + NGCC build is stable on:

1. Node 14
2. Node 16

### Angular 15 build is unstable on:

1. Node 18
(NGCC errors like you experienced)

### SWA CLI requires:

1. Node 18+

So the correct process is:

| Stage            | Node Version |
| ---------------- | ------------ |
| Install deps     | Node 16      |
| Build Angular    | Node 16      |
| Package artifact | Node 16      |
| Deploy to SWA    | **Node 18**  |

-----

## **2. Create React sample app (recommended way)**


# **1. Create a new React app**

Run:

```bash
npx create-react-app react-swa-sample cd react-swa-sample
``` 

----------

----------

# ✅ **2. Modify homepage text so you can verify deployment**

Open:

`src/App.js` 

Replace content with:

``` js
function App() {
  return (
    <div style={{ textAlign: "center", marginTop: 40 }}>
      <h1>✔ React app deployed to Azure Static Web Apps</h1>
      <p>If you see this message, deployment worked!</p>
    </div>
  );
}
export default App;

```

----------

----------

# ✅ **3. Add Azure Static Web App config**

Create a file:

```staticwebapp.config.json``` 

Content:

``` json
{
  "routes": [
    {
      "route": "/*",
      "rewrite": "/index.html"
    }
  ]
}

```

📌 This ensures correct routing for SPA refresh / deep links.

----------

----------

# ✅ **4. Build the app**

`npm run build` 

Output folder created:

```css
build/
   index.html
   static/
   ...

``` 

----------

----------

# ✅ **5. Deploy to Azure Static Web Apps**

You need the Azure SWA CLI installed:

```bash
npm install -g @azure/static-web-apps-cli
``` 

Now deploy:

```bash
swa deploy --app-location build
``` 

Azure will ask for deployment token if not provided.

### You get this token from:

Azure Portal → Static Web App → Deployment tokens → Copy token

Run deploy with token:

```bash
swa deploy --app-location build --deployment-token <your-token>
```

----------

----------

# Your React app is live!

The output will show URL like:

`Project deployed to https://your-site-id.azurestaticapps.net` 


## folder structure you deployed

```css
react-swa-sample/
├── build/              ⇐ deployed folder
├── src/
├── public/
├── staticwebapp.config.json
├── package.json
``` 

----------

----------

## Extra Good Practices

Force rebuild before deploy:

```bash
npm run build
``` 

If Jenkins or CI deploys it, ensure the config file is copied into `build` folder using script:

e.g.,

```bash
cp staticwebapp.config.json build/
``` 

Each deploy should use the production `build` folder.

------------

-----------------

## Jenkinsfile With Multi-Node

```groovy
pipeline {
    agent any

    parameters {
        choice(
            name: 'NODE_VERSION',
            choices: ['16.20.2', '18.18.2'],
            description: 'Node version'
        )
        string(
            name: 'BRANCH',
            defaultValue: 'cs-sprint-13-test',
            description: 'Git branch to build'
        )
    }

    environment {
        DIST_ROOT   = "${WORKSPACE}/dist"
        ENV_SRC     = "${WORKSPACE}/my-modernization-apps-deployment/booking-ui/dev"
        ENV_DEST    = "${WORKSPACE}/apps/sample-ui/src/assets/environments"
        DEPLOYMENT_TOKEN = "12344-21122121-b890-48ca-90dc-890"
    }

    stages {

        /* ---------------------------------------------------------
         * CHECKOUT SOURCE CODE
         * --------------------------------------------------------- */
        stage('Git Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: "*/${params.BRANCH}"]],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'my-app-ssh-key',
                        url: 'git@github.com:abc-companyt/booking-ui.git'
                    ]]
                )
            }
        }

        stage('Checkout my-modernization-apps-deployment') {
            steps {
                dir('my-modernization-apps-deployment') {
                    checkout scmGit(
                        branches: [[name: '*/master']],
                        extensions: [],
                        userRemoteConfigs: [[
                            credentialsId: 'my-app-ssh-key',
                            url: 'git@github.com:abc-companyt/my-modernization-apps-deployment.git'
                        ]]
                    )
                }
            }
        }

        /* ---------------------------------------------------------
         * SELECT NODE VERSION
         * --------------------------------------------------------- */
        stage('Select Node Version') {
            steps {
                bat """
                    echo Switching to Node ${params.NODE_VERSION}...
                    nvm use ${params.NODE_VERSION}
                    node -v
                    npm -v
                """
            }
        }

        /* ---------------------------------------------------------
         * COPY DEV CONFIG FILE
         * --------------------------------------------------------- */
        stage('Copy config file') {
            steps {
                bat """
                echo Copying environment config files...
                xcopy /s /y "${ENV_SRC}" "${ENV_DEST}"
                """
            }
        }

        /* ---------------------------------------------------------
         * BUILD DEV VERSION
         * --------------------------------------------------------- */
        stage('Build Dev') {
            steps {
                bat 'npm install --force'
                bat 'npm install -g nx'
                bat 'npm install -g @azure/static-web-apps-cli'
                bat 'nx --version'
                bat 'nx build sample-ui --configuration=production --skip-nx-cache --no-cloud --verbose'
            }
        }

        /* ---------------------------------------------------------
		 * DEPLOY TO SWA (DEV)
		 * --------------------------------------------------------- */
		stage('Deploy to Azure SWA (DEV)') {
			steps {
				bat """
					echo Switching Node to 18.18.2 for SWA deploy...
					nvm use 18.18.2
					node -v
					swa deploy --app-location "dist/apps/sample-ui" --env "production" --deployment-token "123-456-b890-48ca-90dc-890"
				"""
			}
		}

    }
}


```

-----
