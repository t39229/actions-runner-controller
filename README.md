# actions-runner-controller
actions-runner-controller

Step 1: Provision K3s on devops-int
1.Create 3 instances ubuntu 22.04 (devops-init, devops-init-2, scopioboxVimg)

2.Run script ssh_add_keys_to_nodes.sh (contains loop that scans keys, adds them to known hosts, copies the autorized key for remote host, remove password auth for sudo at remote host and blocks passwordauth)

3. Install last version of ansible on devops-init
   
4.Create config folder contains inventory.ini, k3s.yml

5. Populate inventory.ini hostanme=devops-init-2
   
7. Create ansible playbook (k3s,yaml)
   
9. Run ansible-playbook -i inventory.ini k3s.yml
    
11. After the playbook runs successfully, you'll have a kubeconfig.yml file in the same directory.  You need to tell kubectl to use this file to connect to your K3s cluster:
    
export KUBECONFIG=$PWD/kubeconfig.yml

13. Verify k3s master node is running
14. 
kubectl get nodes

Step 2: Patch & Sing SecurityBoot Machine via Github Action + Helm

Task 1: Deploy GitHub Runner via Helm:

1. Create namespace ci-runners
2. Add the action-runner-controller Helm repository:
   
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller && helm repo update

4. Install the action-runner-controller using Helm:
   
   helm install actions-runner-controller actions-runner-controller/actions-runner-controller \
  --namespace ci-runners \
  --set authSecret.create=true \
  --set authSecret.github_token=<YOUR_GITHUB_token>
  
5.  Create a YAML file (e.g., runner-deployment.yaml) to define your runner deployment:
   apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runnerdeploy
spec:
  replicas: 1
  template:
    spec:
      repository: <YOUR_GITHUB_USERNAME>/<YOUR_REPOSITORY_NAME>

6. Apply the runner deployment:
   kubectl apply -f runner-deployment.yaml

7. Verify that the runners are created and registered:
   
 ➜  nir777ert777 kubectl get runners
 
NAME                              ENTERPRISE   ORGANIZATION   REPOSITORY                         GROUP   LABELS   STATUS    MESSAGE   WF REPO   WF RUN   AGE

scopio-runnerdeploy-hmj26-z776r                               t39229/actions-runner-controller                    Running                                32m

9. Check your GitHub repository settings to confirm that the runners are connected.
    
   <img width="783" height="233" alt="image" src="https://github.com/user-attachments/assets/d713e111-118d-47f9-bba3-b088615017ff" />

Task 2: Create GitHub Workflow:

name: Secure Patching

on:
  push:
    branches: [ main ]  # Trigger on pushes to the main branch
  workflow_dispatch:  # Allow manual triggering

jobs:
  patching:
    runs-on: [self-hosted, linux]

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: SSH and Patch
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: 12.10.10.4
          username: nir777ert777
          key: ${{ secrets.SSHPRIVATEKEY }}       # SSH Private key stored in GitHub secrets
          script: |
            set -e  # Exit immediately if any command fails
            # Update package lists and upgrade installed packages
            sudo apt update && sudo apt upgrade -y
            # Sign the kernel modules
            sudo python3 /opt/secureboot/scripts/sign-kernel-modules.py
            # Check Secure Boot state
            mokutil --sb-state
      - name: Print Result
        run: |
          echo "Patching and signing completed (or failed)."
   

2. Helm values.yaml
githubTokenSecret:
  create: true
  name: github-token
  stringData:
    github_token: <yout_token>
# Use the created secret as the token value
githubToken:
  valueFrom:
    secretKeyRef:
      name: github-token # The name of the secret
      key: github_token  # The key in the secret

runnerGroup: default
metrics:
  enabled: false

# Horizontal Runner Autoscaler Configuration
horizontalRunnerAutoscaler:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  scaleTargetRef:
    apiVersion: actions.summerwind.dev/v1alpha1
    kind: RunnerDeployment
    name: scopio-runner-deployment # Name of the runner deployment
# Runner Deployment Configurations
runnerDeployment:
  enabled: true
  name: scopio-runner-deployment
  replicas: 1
  template:
    spec:
      labels:
        self-hosted: "true"
        os: "linux" # Example Label make sure it matches with Github Actions labels


Screenshot of successful GitHub Action run.
    <img width="1662" height="666" alt="image" src="https://github.com/user-attachments/assets/c2dda2f2-7702-4a57-a714-e95ea9734480" />

kubectl get po -n ci-runners

NAME                                         READY   STATUS    RESTARTS   AGE

actions-runner-controller-559597c7d5-f8x79   2/2     Running   0          4h14m

    




