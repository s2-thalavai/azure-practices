# Azure Static Web App

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

## **2. Create React sample app (recommended way)**
