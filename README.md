**Securing Microservices on Kubernetes**. 
This chapter is all about moving from a system that simply *runs* to one that is locked down against attackers who want to steal data, run botnets, or corrupt your system.

## 1. The Concepts Explained in Simplest Terms

* **User Accounts vs. Service Accounts:** * *User Accounts* are for **humans** (like you logging into the dashboard via GitHub OAuth2).
* *Service Accounts* are for **software** (like your Spring Boot app needing permission to talk to Kubernetes to launch pods).


* **RBAC (Role-Based Access Control):** This is a checklist of who can do what. Instead of giving your Spring Boot app total control over the entire cluster, you give it a specific "Role" that *only* allows it to create Deployments in the `default` namespace.

* **Secrets Management:** Never hardcode database passwords or GitHub tokens in your code or YAML files. A Kubernetes `Secret` is an encrypted digital vault inside the cluster that securely holds this data and injects it into your containers only when they run.

---

## 2. Project: Secure Secret & Service Account

We will create a micro-project where a secure application needs to read a database password from a Kubernetes Vault (**Secret**) using a strictly limited identity (**Service Account** and **RBAC**).

### Step 1: Create a Project Folder

Open your Command Prompt (`cmd`) and create a fresh directory for this lab:

```cmd
mkdir C:\Users\USER\Desktop\K8s-Security-Lab
cd C:\Users\USER\Desktop\K8s-Security-Lab

```

### Step 2: Create the Secure Vault (The Secret)

We will store a database password securely inside Kubernetes so no human can see it in plain text in the deployment configurations.

In your terminal, run this command to create the secret directly:

```cmd
kubectl create secret generic db-vault --from-literal=password=SuperSecretMMS4Password

```

*To verify it exists, run:* `kubectl get secrets`

### Step 3: Define the Limited Identity & Security Rules (RBAC)

We need to create a `ServiceAccount` (the identity) and a `Role` (the rule allowing it to read secrets), then bind them together.

Create a file named `security-rules.yaml` in your folder. You can use Notepad from the command prompt to create it:

```cmd
notepad security-rules.yaml

```

Paste the following configuration inside and save it:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: previewer-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: previewer-rbac-binding
subjects:
- kind: ServiceAccount
  name: previewer-sa
  namespace: default
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io

```

Apply this file to your cluster using `cmd`:

```cmd
kubectl apply -f security-rules.yaml

```

### Step 4: Deploy the Secure Application Pod

Now we will launch a test pod that uses our limited `ServiceAccount` and securely mounts the password from the `db-vault` secret straight into its environment variables.

Create a file named `secure-pod.yaml`:

```cmd
notepad secure-pod.yaml

```

Paste this configuration, save, and close:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app-pod
  namespace: default
spec:
  serviceAccountName: previewer-sa
  containers:
  - name: app-container
    image: alpine
    command: ["sh", "-c", "echo 'Application Started!'; echo 'My Secure DB Password is: ' $DB_PASSWORD; sleep 3600"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-vault
          key: password

```

Deploy the pod using your terminal:

```cmd
kubectl apply -f secure-pod.yaml

```

---

## 3. The Hero Moment: Verification

Let's verify that your security configuration successfully passed the secret data into the runtime environment without ever exposing it in your deployment files.

1. Check if the pod is running successfully:
```cmd
kubectl get pods

```


2. Read the internal logs of the container to watch it securely pull the environment variable from the secret manager:
```cmd
kubectl logs secure-app-pod

```



**Expected Output:**

```text
Application Started!
My Secure DB Password is: SuperSecretMMS4Password

```

### Why --- Classroom:

You just demonstrated a bulletproof production architecture from scratch:

1. The password is encrypted in a Kubernetes **Secret**.
2. The pod runs on a restricted **Service Account** with minimal **RBAC** clearance.
3. If an attacker breaches the cluster, they can't access other namespaces because your rules locked down the resource visibility.


GitHub Setup

Step 1: Add Your Secret to GitHub's Vault
Just like Kubernetes has Secrets, GitHub has a secure vault to store sensitive data so it never leaks into your public code files.

Go to your repository on GitHub.

Click on Settings (the gear icon at the top).

On the left sidebar, scroll down to Secrets and variables and click Actions.

Click the green New repository secret button.

Set the Name to: DB_PASSWORD

Set the Secret value to: SuperSecretMMS4Password

Click Add secret.

Step 2: Create Your Kubernetes Resource Files
Inside your repository, make sure you have your two configuration files saved exactly as we defined them earlier. You can commit them to the main directory of your repo:

security-rules.yaml (Defines your ServiceAccount and RBAC roles)

secure-pod.yaml (Defines the application container)

Step 3: Create the Automated Workflow
GitHub Actions automatically looks for files inside a special hidden folder structure named .github/workflows/.

Inside your repository, create a folder named .github and a subfolder named workflows.

Create a file inside that subfolder named k8s-security-pipeline.yml.

Paste the following automated script inside it:
YAML
name: Kubernetes Security Deployment Pipeline

# Trigger this workflow every time someone pushes code to the main branch
on:
  push:
    branches: [ "main" ]

jobs:
  deploy-and-test:
    runs-on: ubuntu-latest

    steps:
    # Step 1: Pull the code from your repository into the runner environment
    - name: Checkout Code
      uses: actions/checkout@v4

    # Step 2: Spin up a miniature local Kubernetes cluster (Minikube) inside the GitHub server
    - name: Start Local Kubernetes Cluster
      uses: medyagh/setup-minikube@master

    # Step 3: Securely generate the Kubernetes secret using the value hidden in GitHub Settings
    - name: Create Kubernetes Secret
      run: |
        kubectl create secret generic db-vault --from-literal=password="${{ secrets.DB_PASSWORD }}"

    # Step 4: Apply your Service Account identity rules (RBAC)
    - name: Apply RBAC and Service Accounts
      run: |
        kubectl apply -f security-rules.yaml

    # Step 5: Deploy your secure application container
    - name: Deploy Secure Pod
      run: |
        kubectl apply -f secure-pod.yaml

    # Step 6: Wait briefly for the pod to boot, then verify the logs to prove success
    - name: Verify Secure Log Output
      run: |
        echo "Waiting for pod to initialize..."
        kubectl wait --for=condition=Ready pod/secure-app-pod --timeout=60s
        echo "--- RUNNING CONTAINER LOGS ---"
        kubectl logs secure-app-pod

Step 4: Watch the Automation Run.
Once you commit and push that .yml file to GitHub, the automation engine instantly fires up:

Click on the Actions tab at the top of your GitHub repository page.

You will see a live workflow running named "Kubernetes Security Deployment Pipeline". Click on it.

Click into the deploy-and-test job box to watch the real-time command terminal stream directly in your browser.

When it hits the final step ("Verify Secure Log Output"), you will see it print the application startup logs confirming that your automated environment securely injected your hidden password secret directly into the running instance container!

