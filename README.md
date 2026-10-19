# End-to-End DevOps CI/CD and GitOps Project

## Overview

This project demonstrates an automated CI/CD and GitOps workflow for deploying a portfolio website to a local Kubernetes cluster.

Jenkins builds the application’s Docker image and pushes versioned images to Docker Hub. It then updates the image tag in a separate GitOps repository. Argo CD detects the manifest change and synchronizes the desired state with Kubernetes.

## Technology Stack

- **Source Control:** Git and GitHub
- **CI/CD:** Jenkins
- **Containerization:** Docker
- **Container Registry:** Docker Hub
- **Orchestration:** Kubernetes (Rancher Desktop/K3s)
- **GitOps:** Argo CD
- **Ingress:** Traefik

## Architecture and Pipeline Flow

1. Developer pushes application changes to GitHub.
2. Jenkins checks out the source code.
3. Jenkins validates the required application files.
4. Jenkins builds a Docker image and assigns a build-number tag.
5. Jenkins pushes the versioned image and `latest` tag to Docker Hub.
6. Jenkins checks out the GitOps repository and updates the Kubernetes Deployment image tag.
7. Jenkins commits and pushes the manifest change.
8. Argo CD detects the GitOps change and synchronizes the application.
9. Kubernetes updates the Deployment and manages the ReplicaSets and Pods.
10. Traefik routes HTTP traffic to the portfolio application.

## Repositories

- **Application source:** https://github.com/AravindaKumar-0/devops-portfolio
- **GitOps manifests:** https://github.com/AravindaKumar-0/devops-portfolio-gitops

## Key Features

- Automated Docker image build and registry push through Jenkins.
- Versioned Docker images using Jenkins build numbers.
- Automated Kubernetes manifest updates.
- Argo CD synchronization with automated sync, pruning, and self-healing enabled.
- Kubernetes replica scaling managed through GitOps.
- Local Kubernetes deployment without requiring a billable AWS cluster.

## Kubernetes Deployment

The GitOps repository contains the Kubernetes manifests:

- `k8s/deployment.yaml`
- `k8s/service.yaml`
- `k8s/ingress.yaml`

Argo CD monitors the GitOps repository and reconciles the cluster with the desired configuration.

## Accessing the Application Locally

The portfolio website is accessed through Traefik using the local hostname `portfolio.local`.

The local setup requires the appropriate hosts-file entry and a running port-forward to the Traefik service:

```bash
kubectl port-forward -n kube-system svc/traefik 8000:80
```

Then open:

http://portfolio.local:8000/

Keep the port-forward terminal running while accessing the website.

## Challenges and Lessons Learned

- **CI versus GitOps:** Application image changes go through Jenkins, while Kubernetes manifest changes can be synchronized directly by Argo CD.
- **Image versioning:** Unique build-number tags make application deployments easier to track.
- **GitOps reconciliation:** Argo CD compares the Git repository with the live cluster and applies the desired configuration.
- **Kubernetes scaling:** Changing the Deployment replica count allows Kubernetes to create or remove Pods to match the desired state.
- **Networking:** Local access depends on the correct hostname configuration, Ingress routing, and port-forwarding setup.

## Future Improvements

- Verify and document successful GitHub webhook delivery.
- Add Prometheus and Grafana monitoring.
- Add automated application tests and deployment notifications.
- Improve deployment security and secret management.
