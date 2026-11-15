# Contributing

Thank you for improving this Argo CD example.

## Before opening a pull request

1. Keep changes focused and explain the operational impact.
2. Do not commit credentials, kubeconfig files, generated passwords, or cloud
   account identifiers.
3. Validate Kubernetes manifests with `kubectl` or `kubeconform`.
4. Validate Compose changes with `docker compose config`.
5. Update the README when commands, prerequisites, or resource names change.

## Pull requests

Use the pull request template and include the validation commands you ran.
Changes must pass the GitHub Actions workflows before merging.
