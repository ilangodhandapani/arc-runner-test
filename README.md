To test self-hosted runners running via Actions Runner Controller (ARC) on AKS
Step 1: Create the 2-Node AKS Standard Cluster

We chose Standard_D2s_v5 (2 vCPUs, 8 GB RAM per node) so two nodes equal 4 vCPUs total, staying within your regional vCPU quota without needing Spot VMs or cert-manager.

# 1. Create the AKS Standard cluster (2 nodes x 2 vCPUs = 4 vCPUs)
az aks create \
  --resource-group ilango \
  --name my-aks-standard \
  --node-count 2 \
  --node-vm-size Standard_D2s_v5 \
  --generate-ssh-keys

# 2. Fetch kubeconfig credentials and set kubectl context
az aks get-credentials \
  --resource-group ilango \
  --name my-aks-standard \
  --overwrite-existing

# 3. Verify both nodes are Ready and untainted
kubectl get nodes -o wide


Step 2: Install the ARC Controller (gha-runner-scale-set-controller)

ARC v2 uses OCI Helm charts directly from GitHub Container Registry (ghcr.io) and does not require cert-manager.
bash
# 1. Install the ARC controller into the arc-systems namespace
helm install arc \
  --namespace arc-systems \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

# 2. Verify the controller pod is 1/1 Running
kubectl get pods -n arc-systems

Step 3: Install & Configure the ARC Runner Scale Set (gha-runner-scale-set)

Deploy the runner scale set pointing to your GitHub repository and Personal Access Token (repo scope for a repository, or admin:org for an organization).
bash
# 1. Install (or upgrade) the runner scale set in the arc-runners namespace
helm upgrade --install arc-runner-set \
  --namespace arc-runners \
  --create-namespace \
  --set githubConfigUrl="https://github.com" \
  --set githubConfigSecret.github_token="<YOUR_GITHUB_PAT>" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set

# 2. Verify both Helm releases are deployed
helm list -A

Step 4: Verify the GitHub Listener Pod & Connection

Once configured with a valid URL and PAT, the ARC controller automatically spawns a listener pod inside arc-systems that polls GitHub for queued jobs.
bash
# 1. Confirm both the controller and listener pods are 1/1 Running
kubectl get pods -n arc-systems

# 2. Check listener logs to confirm connection to GitHub ("Getting next message")
kubectl logs -n arc-systems -l ://github.com --tail=20


Step 5: Configure & Trigger the Test GitHub Actions Workflow

In your repository (ilangodhandapani/arc-runner-test), add .github/workflows/test-arc-runner.yml:
yaml
name: Test AKS ARC Runner

on:
  workflow_dispatch:

jobs:
  basic-check:
    runs-on: arc-runner-set
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run basic shell commands
        run: |
          echo "Hello from AKS ARC self-hosted runner!"
          echo "Runner Name: $RUNNER_NAME"
          echo "Hostname: $(hostname)"
          echo "Current Date: $(date)"
          ls -la

1. Trigger the workflow from GitHub → Actions → Test AKS ARC Runner → Run workflow.
2. Watch AKS dynamically spin up an ephemeral runner pod in arc-runners and delete it once the job completes:bash
kubectl get pods -n arc-runners -w


