# Example Voting App

A simple distributed application running across multiple Docker containers.

## Pre requisites

- docker installed on the system
- WSL for window machines
- kubectl installed in the system

## Architecture

![Architecture diagram](architecture.excalidraw.png)

- A front-end web app in [Python](/vote) which lets you vote between two options
- A [Redis](https://hub.docker.com/_/redis/) which collects new votes
- A [.NET](/worker/) worker which consumes votes and stores them in…
- A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
- A [Node.js](/result) web app which shows the results of the voting in real time

## Kubernetes Architecture

All resources run in a single namespace, `voting-app`, split into three tiers.

- `vote` application has 2 replicas and is exposed over NodePort at 31000
- `result` application has 2 replicas and is exposed over NodePort at 31001
- The deployment for vote and result run front the `frontend` directory.
- `redis` is the back backend queue and `db` are deployed as `StatefulSet`s, with each of them using a `volumeClaimTemplate` so data survives pod restarts.
- `worker` has 1 replica that is responsible for writing the votes from `redis` to `db`.
- All cross-tier calls go through Service DNS names (`redis`, `db`), never pod IPs.
- `postgres-config` (non-sensitive) and `postgres-secret` (sensitive: DB user/password) are injected via `envFrom` into the postgres, `worker`, and `result` pods, so no credentials are hardcoded in the images.
- `postgres-networkpolicy.yaml` locks port 5432 down to only `worker` and `result`.

## Deployment Flow

First we need to install the minikube on the local system. As I am using WSl so, I will install it for linux

  ```bash
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
  sudo dpkg -i minikube_latest_amd64.deb
  
  ```

After this, start the docker desktop ( if on windows ) and launch the minikube

  ```bash
  minikube start --profile=example-voting-app --driver=docker --cpus=2 --memory=4096 --cni=calico
  ```

  The calico container network interface is used by minikube to apply the network policies as the native minikube network driver does not support using the custom network policies

After starting it, now install the argocd

  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  kubectl get pods -n argocd
  ```

Now ArgoCD is up, in the same terminal get the admin password and then forward the port to port 8000 to view the ArgoCD UI

  ```bash
  # get password
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  
  # forward port
  kubectl -n argocd port-forward svc/argocd-server 8000:443
  ```

Now push the code to the github so that the container registry has the images there for the application and then run the following command from the projects directory

  ```bash
  kubectl apply -f argocd/root-app.yaml
  ```

This will launch the 4 applications

- Namespace application that creates and manages the namespace
- The Backend application that deploys Redis and Worker
- The Database application that deploys the Postgres DB
- The Frontend application that deploys the Vote and Result Frontend

To view the applications use the following commands in two different terminals

  ```bash
  # for vote
  minikube service vote -n voting-app --url -p example-voting-app

  # for result
  minikube service result -n voting-app --url -p example-voting-app
  ```

## CI Workflow

This is how the CI workflow triggers, build and push the images and then trigger the CD

- A push to `main` branch touching `vote/`, `result/`, or `k8s/frontend/--` triggers `frontend-ci.yaml`
- A push to `main` branch touching `worker/`, `k8s/backend/--`, or `k8s/database/--` triggers `backend-ci.yaml`.
- Each workflow builds its service's Docker image and pushes it to Docker Hub as `votingapp:vote-latest`, `votingapp:worker-latest` and `votingapp:result-latest`.
- The workflow's deploy job then syncs the matching `k8s/-` manifests onto a `production` branch, rewriting the `image:` line to the tag it just pushed to `production`.
- ArgoCD watches `production` and auto-syncs four `Application`s (namespace, database, backend, frontend) via an app-of-apps root, ordered by `sync-wave` so the namespace and database come up before backend/frontend.
- Once synced, reach the application using the following commands
