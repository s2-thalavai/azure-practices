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

# **2. Add Azure Static Web Apps configuration**

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

swa deploy --deployment-token 1234-12122121 --app-location dist/swa-angular-sample

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

`Project deployed to https://your-site-id.azurestaticapps.net 🚀` 


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

    /*****************************************
     * PARAMETERS
     *****************************************/
    parameters {
        choice(
            name: 'NODE_VERSION',
            choices: ['14.21.3', '16.20.2', '18.18.2', '20.19.0'],
            description: 'Select Node version for this build'
        )

        string(
            name: 'BRANCH',
            defaultValue: 'master',
            description: 'Branch to checkout and build'
        )
    }

    stages {

        /*****************************************
         * CHECKOUT BOOKING-UI
         *****************************************/
        stage('Checkout booking-ui') {
            steps {
                checkout scmGit(
                    branches: [[name: "*/${BRANCH}"]],
                    userRemoteConfigs: [[
                        credentialsId: 'is-apps-ssh-key',
                        url: 'git@github.com:---Limited/booking-ui.git'
                    ]]
                )
            }
        }

        /*****************************************
         * CHECKOUT CONFIG REPO
         *****************************************/
        stage('Checkout Config Repo') {
            steps {
                dir('IS-modernization-apps-deployment') {
                    checkout scmGit(
                        branches: [[name: '*/master']],
                        userRemoteConfigs: [[
                            credentialsId: 'is-apps-ssh-key',
                            url: 'git@github.com:---Limited/-modernization-apps-deployment.git'
                        ]]
                    )
                }
            }
        }

        /*****************************************
         * SELECT NODE VERSION USING NVM
         *****************************************/
        stage('Select Node Version') {
            steps {
                bat """
                echo Available Node Versions:
                nvm list

                echo Switching to Node ${NODE_VERSION}...
                nvm use ${NODE_VERSION}

                echo Active Node Version:
                node -v
                """
            }
        }

        /*****************************************
         * VALIDATE NODE COMPATIBILITY
         *****************************************/
        stage('Validate Node Version') {
            steps {
                script {
                    def version = bat(script: "node -v", returnStdout: true).trim()
                    echo "Detected Node Version: ${version}"

                    if (BRANCH == "prod" && !version.startsWith("v18")) {
                        error "Production builds require Node 18.x — found ${version}"
                    }
                }
            }
        }

        /*****************************************
         * COPY ENV CONFIGURATION FILES
         *****************************************/
        stage('Copy Config File') {
            steps {
                bat """
                xcopy /s /y ^
                "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\booking-ui-prod\\-modernization-apps-deployment\\booking-ui\\prod" ^
                "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\booking-ui-prod\\apps\\booking\\src\\assets\\environments"
                """
            }
        }

        /*****************************************
         * INSTALL DEPENDENCIES & BUILD BOOKING UI
         *****************************************/
        stage('Install & Build') {
            steps {
                bat """
                echo Installing dependencies...
                npm install --legacy-peer-deps --force

                npm install -g nx
                npm install -g @azure/static-web-apps-cli

                echo Nx Version:
                nx --version

                echo Building booking...
                nx build booking --aot --output-hashing=all --configuration=production --skip-nx-cache --no-cloud
                """
            }
        }

        /*****************************************
         * PREPARE OUTPUT DIRECTORY
         *****************************************/
        stage('Flatten Nx dist output') {
            steps {
                bat """
                echo Flattening build output...
                xcopy /s /y ^
                "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\booking-ui-prod\\dist\\apps\\booking" ^
                "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\booking-ui-prod\\dist"

                rmdir /s /q "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\booking-ui-prod\\dist\\apps"
                """
            }
        }

        /*****************************************
         * DEPLOY TO AZURE STATIC WEB APP
         *****************************************/
        stage('Deploy to Azure SWA') {
            steps {
                bat """
                swa deploy "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\booking-ui-prod\\dist" ^
                    --app-name static-webapp ^
                    --deployment-token ----- ^
                    --env production
                """
            }
        }

        /*****************************************
         * OPTIONAL SECURITY / QUALITY STEPS
         *****************************************/

        /*
        stage('Snyk Security Test') {
            steps {
                snykSecurity organisation: 'soma-aravinda',
                             severity: 'critical',
                             snykInstallation: 'Snyk_latest',
                             snykTokenId: 'org-snyk-api-token',
                             targetFile: 'package.json'
            }
        }

        stage('Run Jest Tests') {
            steps {
                bat 'cd apps\\booking && npx jest --coverage'
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    bat 'npm run sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        */

    } // stages

    /*****************************************
     * POST ACTIONS
     *****************************************/
    post {
        success {
            echo "Build & Deployment Successful!"
        }
        failure {
            echo "Build Failed!"
        }
    }
}

```

-----
