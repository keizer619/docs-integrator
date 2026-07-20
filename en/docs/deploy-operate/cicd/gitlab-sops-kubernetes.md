---
title: GitLab CI/CD with SOPS Secrets
description: Decrypt a SOPS-encrypted Config.toml in a GitLab pipeline and deploy code-to-cloud Kubernetes artifacts.
---

# GitLab CI/CD with SOPS Secrets

Build a GitLab pipeline that decrypts a [SOPS](https://github.com/getsops/sops)-encrypted `Config.toml`, builds a WSO2 Integrator (Ballerina) project with the code-to-cloud (c2c) extension, pushes the container image to the GitLab registry, and deploys the generated Kubernetes artifacts.

This is the CI/CD version of [Encrypt Config.toml with SOPS](../secure/sops-config-secrets.md) — complete that tutorial first. You should already have a project containing:

```
greeter/
├── .gitignore          # excludes Config.toml, age.key, target/
├── .sops.yaml          # age public key + creation rules
├── Ballerina.toml      # distribution 2201.13.4, cloud = "k8s"
├── Cloud.toml          # [[cloud.config.secrets]] file = "Config.toml"
├── Config.toml.enc     # SOPS-encrypted configuration (committed)
└── main.bal
```

## Overview

The pipeline has four stages:

1. **test** — decrypt `Config.toml.enc`, run `bal test`.
2. **build** — decrypt `Config.toml.enc`, run `bal build`; c2c generates the Dockerfile and Kubernetes YAML, embedding the decrypted configuration as a Kubernetes `Secret`.
3. **docker** — build the image from the generated Dockerfile and push it to the GitLab container registry.
4. **deploy** — apply the generated Kubernetes artifacts with `kubectl`.

The age **private** key never touches the repository — it lives in a GitLab CI/CD variable and is used only to decrypt inside pipeline jobs.

## Prerequisites

- A GitLab project with the source code above
- GitLab Runner available (shared or project-specific, `amd64`)
- GitLab container registry enabled for the project
- A Kubernetes cluster connected via the [GitLab agent for Kubernetes](https://docs.gitlab.com/ee/user/clusters/agent/)

## Step 1: Adjust Cloud.toml for CI

Two changes compared to the local tutorial:

- Point `repository` at your GitLab container registry so the generated `Deployment` references the image the pipeline pushes.
- Set `buildImage = false` so `bal build` only generates the Dockerfile and Kubernetes YAML — the image is built in a dedicated Docker stage, keeping the Ballerina job free of Docker-in-Docker.

```toml
[container.image]
repository = "registry.gitlab.com/mygroup/myproject"   # your $CI_REGISTRY_IMAGE
name = "greeter"
tag = "0.1.0"

[settings]
buildImage = false

# Mount Config.toml as a Kubernetes Secret (not a ConfigMap).
[[cloud.config.secrets]]
file = "Config.toml"

[cloud.deployment]
min_memory = "256Mi"
max_memory = "512Mi"
min_cpu = "200m"
max_cpu = "500m"

[cloud.deployment.probes.readiness]
port = 9090
path = "/healthz"

[cloud.deployment.probes.liveness]
port = 9090
path = "/healthz"
```

With `buildImage = false`, `bal build` still produces everything the later stages need:

- `target/docker/greeter/` — Dockerfile plus the application and runtime JARs
- `target/kubernetes/greeter/greeter.yaml` — Service, Secret, Deployment, and HPA

## Step 2: Store the age private key in GitLab

1. In your GitLab project, go to **Settings > CI/CD > Variables** and select **Add variable**.
2. Set:
    - **Type**: `File`
    - **Key**: `SOPS_AGE_KEY_FILE`
    - **Value**: the full content of your `age.key` file (the `AGE-SECRET-KEY-...` line, comments included)
    - **Flags**: enable **Protect variable** so it is only available on protected branches
3. Select **Add variable**.

Because the variable type is `File`, GitLab writes the key to a temporary file and exposes its **path** in the `SOPS_AGE_KEY_FILE` environment variable — which is exactly the variable SOPS reads to find an age key. No extra wiring is needed.

## Step 3: Create the pipeline

Create a `.gitlab-ci.yml` at the root of your repository:

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - docker
  - deploy

variables:
  SOPS_VERSION: "3.12.1"
  IMAGE: "$CI_REGISTRY_IMAGE/greeter:0.1.0"   # must match Cloud.toml [container.image]

# Download the sops static binary and decrypt Config.toml.enc.
# SOPS finds the age key via the SOPS_AGE_KEY_FILE file variable (Step 2).
.decrypt-config: &decrypt-config
  - wget -q -O sops "https://github.com/getsops/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64"
  - chmod +x sops
  - ./sops --decrypt Config.toml.enc > Config.toml

test:
  stage: test
  image: ballerina/ballerina:2201.13.4
  script:
    - *decrypt-config
    - bal test

build:
  stage: build
  image: ballerina/ballerina:2201.13.4
  script:
    - *decrypt-config
    - bal build
  artifacts:
    paths:
      - target/docker/greeter/
      - target/kubernetes/greeter/
    expire_in: 1 hour

docker:
  stage: docker
  image: docker:27
  services:
    - docker:27-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$IMAGE" target/docker/greeter
    - docker push "$IMAGE"

deploy:
  stage: deploy
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  script:
    - kubectl config use-context mygroup/myproject:my-agent   # your GitLab agent context
    - kubectl apply -f target/kubernetes/greeter/
  environment: production
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

## How the stages work

### test and build

Both jobs run on the official `ballerina/ballerina:2201.13.4` image. The shared `.decrypt-config` anchor downloads the sops static binary (the image is Alpine-based and runs as a non-root user, so a self-contained binary in the project directory avoids needing a package manager) and decrypts `Config.toml.enc` into `Config.toml`.

Decryption must happen **before** `bal build`: c2c reads `Config.toml` at build time and embeds its content — base64-encoded — into the Kubernetes `Secret` in `target/kubernetes/greeter/greeter.yaml`.

The build job publishes only the two directories the later stages need. The plaintext `Config.toml` is deliberately **not** listed under `artifacts:paths`.

### docker

Runs on the standard `docker:27` image with a Docker-in-Docker service, builds the image from the c2c-generated Dockerfile in `target/docker/greeter/`, and pushes it to the GitLab container registry using the built-in `$CI_REGISTRY_*` credentials. No secrets are baked into the image — the configuration reaches the pod only through the Kubernetes `Secret`.

### deploy

Applies the generated artifacts through the GitLab agent for Kubernetes. Replace `mygroup/myproject:my-agent` with your agent context (`kubectl config get-contexts` inside a job lists what is available). The job is `manual` and restricted to the default branch so a human gates production deployments.

If your cluster cannot pull from a private GitLab registry yet, create a pull secret once and attach it to the default service account, or use a [deploy token](https://docs.gitlab.com/ee/user/project/deploy_tokens/).

## Step 4: Run the pipeline

Commit and push:

```bash
git add .gitlab-ci.yml Cloud.toml
git commit -m "Add SOPS-enabled GitLab pipeline"
git push
```

The pipeline runs test → build → docker automatically; trigger **deploy** manually from the pipeline view. Verify the rollout:

```bash
kubectl get secret config-secret
kubectl get pods -l app=greeter
```

## Security notes

- **`greeter.yaml` contains the secret.** The generated Kubernetes YAML embeds the decrypted `Config.toml` base64-encoded, and it is passed between stages as a job artifact. Keep `expire_in` short, and restrict who can download artifacts (**Settings > CI/CD > Job token permissions** and project visibility settings).
- **Protect the key variable.** With **Protect variable** enabled, `SOPS_AGE_KEY_FILE` is only injected on protected branches — pipelines from forks or feature branches cannot decrypt.
- **Job logs are safe by default.** SOPS never prints decrypted content, and the pipeline never `cat`s `Config.toml`. Keep it that way — do not echo configuration values for debugging.
- **Rotate by re-encrypting.** Rotation is a Git commit: update the value locally, `sops --encrypt Config.toml > Config.toml.enc`, push, and re-run the pipeline. See [Rotating secrets](../secure/sops-config-secrets.md#rotating-secrets).

## What's next

- [Encrypt Config.toml with SOPS](../secure/sops-config-secrets.md) — the local version of this workflow
- [GitLab CI/CD](gitlab.md) — basic build and test pipeline
- [GitHub Actions](github-actions.md) — CI/CD with GitHub-hosted runners
