 Here are all the commands to run on the new system:

  Prerequisites

  # Ensure kind cluster is running
  kind create cluster --name kind   # skip if already exists
  kubectl cluster-info

  Step 1 — Login and pull the runner image

  echo "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" | docker login ghcr.io -u vinodkrishnavarma --password-stdin
  docker pull ghcr.io/vinodkrishnavarma/actions_cache:main

  Step 2 — Load image into kind (avoids pull issues inside cluster)

  kind load docker-image ghcr.io/vinodkrishnavarma/actions_cache:main --name kind

  Step 3 — Install ARC controller

  helm install arc \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
    --namespace arc-systems \
    --create-namespace \
    --wait

  Step 4 — Create namespace and GitHub token secret

  kubectl create namespace arc-runners

  kubectl create secret generic gh-pat-secret \
    --namespace arc-runners \
    --from-literal=github_token=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

  Step 5 — Install the runner scale set

  helm install arc-runner-set \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
    --namespace arc-runners \
    --set githubConfigUrl="https://github.com/vinodkrishnavarma/action_cache_test" \
    --set githubConfigSecret=gh-pat-secret \
    --set template.spec.containers[0].name=runner \
    --set template.spec.containers[0].image=ghcr.io/vinodkrishnavarma/actions_cache:main \
    --set template.spec.containers[0].imagePullPolicy=Never \
    --set minRunners=0 \
    --set maxRunners=3 \
    --wait

  Step 6 — Verify listener is connected

  kubectl get pods -n arc-systems
  kubectl logs -n arc-systems -l app.kubernetes.io/component=runner-scale-set-listener --tail=10

  Step 7 — Trigger the test workflow

  # Either push to the repo or trigger manually:
  gh run trigger  # from the repo, or go to GitHub UI → Actions → Test Action Cache → Run workflow

  Step 8 — Watch the runner pick up the job

  kubectl get pods -n arc-runners -w

  ▎ Note: On the new system without proxy, the runner should be able to connect to GitHub directly and the workflow should complete successfully — proving the action archive cache works.