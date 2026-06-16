# Azure Hands-On Lab: Next.js Web App + Python FastAPI API + Easy Auth

## Objective

Deploy:
- Next.js web application on Azure App Service
- Python FastAPI Web API on Azure App Service
- Microsoft Entra ID authentication using Easy Auth
- API authorization using exposed scopes

## Architecture

```text
Browser
  |
  | Sign in with Entra ID
  v
Next.js App Service
  |
  | Calls Python API
  v
Python FastAPI App Service
  |
  | Easy Auth validates caller
  v
Protected API endpoint
```

## Resources

- Resource Group
- App Service (Next.js)
- App Service (Python FastAPI)
- Microsoft Entra ID App Registration (Web)
- Microsoft Entra ID App Registration (API)

## Lab Steps

### Step 1 – Create Resource Group

Resource Group:
`rg-python-nextjs-easyauth-lab`

Region:
`Central US`

### Step 2 – Create Python App Service

Name:
`api-easyauth-lab-<unique>`

Runtime:
`Python 3.12`

OS:
`Linux`

### Step 3 – Create Next.js App Service

Name:
`web-easyauth-lab-<unique>`

Runtime:
`Node 20 LTS`

OS:
`Linux`

### Step 4 – Create API App Registration

Name:
`lab-python-api-reg`

Copy:
- Client ID
- Tenant ID

### Step 5 – Expose API Scope

Application ID URI:

```text
api://<API_CLIENT_ID>
```

Scope:

```text
access_as_user
```

Final scope:

```text
api://<API_CLIENT_ID>/access_as_user
```

### Step 6 – Create Web App Registration

Name:

```text
lab-nextjs-web-reg
```

Redirect URI:

```text
https://web-easyauth-lab-<unique>.azurewebsites.net/.auth/login/aad/callback
```

### Step 7 – Create Client Secret

Create a client secret and save the value.

### Step 8 – Grant API Permission

Add delegated permission:

```text
access_as_user
```

Grant admin consent.

### Step 9 – Configure Easy Auth on Next.js

Authentication settings:

```text
Identity Provider: Microsoft
Require Authentication: Yes
Token Store: Enabled
```

### Step 10 – Configure Easy Auth on Python API

Authentication settings:

```text
Identity Provider: Microsoft
Require Authentication: Yes
Token Store: Enabled
```

Allowed token audiences:

```text
api://<API_CLIENT_ID>
<API_CLIENT_ID>
```

---

# FastAPI Starter Code

## main.py

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "FastAPI running on Azure"}

@app.get("/api/me")
def me():
    return {"message": "Protected endpoint"}
```

## requirements.txt

```text
fastapi
uvicorn
gunicorn
```

### Startup Command

```bash
gunicorn -w 2 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8000 main:app
```

---

# Next.js Starter Code

## app/page.tsx

```tsx
export default function Home() {
  return (
    <main>
      <h1>Next.js + FastAPI + Easy Auth</h1>
      <a href="/.auth/me">View Auth Info</a>
    </main>
  );
}
```

### Environment Variable

```text
NEXT_PUBLIC_API_BASE_URL=https://api-easyauth-lab.azurewebsites.net
```

---

# Deploy Python API

```bash
zip -r api.zip .

az webapp deploy \
  --resource-group rg-python-nextjs-easyauth-lab \
  --name api-easyauth-lab-<unique> \
  --src-path api.zip \
  --type zip
```

# Deploy Next.js

```bash
npm run build

zip -r web.zip .

az webapp deploy \
  --resource-group rg-python-nextjs-easyauth-lab \
  --name web-easyauth-lab-<unique> \
  --src-path web.zip \
  --type zip
```

# Validation

1. Browse to Next.js App Service.
2. Sign in using Entra ID.
3. Verify /.auth/me.
4. Verify API endpoint access.
5. Confirm Easy Auth headers are populated.

## Outcome

- Next.js deployed to Azure App Service
- FastAPI deployed to Azure App Service
- Easy Auth enabled
- Entra ID integrated
- API scope authorization configured
- Secure communication between frontend and API
