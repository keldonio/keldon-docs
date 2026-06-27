# Contributing to Keldon

Thanks for your interest in contributing to Keldon. This guide explains how to participate.

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md). By participating, you agree to abide by its terms. Report unacceptable behavior to **conduct@keldon.io**.

## Contributor License Agreement

By submitting a pull request, you agree to the terms of our [Contributor License Agreement](https://gist.github.com/elhimad/380e7ad3f200955c56d5bb74ddeff2e0).

The first time you open a pull request, the [CLA Assistant](https://cla-assistant.io) bot will comment with a link to sign electronically via GitHub. The signature covers all your future contributions to any Keldon repository. You only need to sign once.

If you cannot sign the CLA (e.g., your employer owns IP rights to your contribution), please open an issue first so we can discuss alternatives.

## Before You Start

Open an issue before submitting a significant pull request. Discussing the change first avoids wasted work if the direction doesn't fit.

For typo fixes, doc improvements, or small bug fixes, go straight to a pull request.

## How to Contribute

1. **Fork the repository** and clone your fork locally.
2. **Create a branch** with a descriptive name: `feat/your-feature`, `fix/bug-description`, `docs/section-name`.
3. **Make your changes** following the conventions described below.
4. **Test your changes** (see the repo-specific testing section below).
5. **Commit** with a clear message. We prefer [Conventional Commits](https://www.conventionalcommits.org/) format: `feat:`, `fix:`, `docs:`, `chore:`, etc.
6. **Open a pull request** against `main`. Fill in the PR template.
7. **Sign the CLA** when the bot prompts you (first PR only).
8. **Iterate on feedback**. A maintainer will review and may request changes.

## Pull Request Guidelines

- One feature or fix per PR. Smaller PRs get reviewed faster.
- Include tests where applicable.
- Update documentation if your change affects user-facing behavior.
- Make sure CI passes before requesting review.
- Keep commit history clean — squash trivial commits before merge.

## Reporting Issues

- **Bugs**: Use the [bug report template](.github/ISSUE_TEMPLATE/bug_report.md). Include reproduction steps, expected vs actual behavior, and environment details.
- **Feature requests**: Use the [feature request template](.github/ISSUE_TEMPLATE/feature_request.md). Explain the use case and motivation, not just the implementation.

## Reporting Security Issues

**Do not file public issues for security vulnerabilities.** Email **security@keldon.io** with details. We will respond within 72 hours.

## Development Setup

> Repo-specific setup instructions go here. Examples below — adapt per repo.

### For the operator (`keldon-operator`)

- Go 1.26 or later
- A Kubernetes cluster for testing (k3s, kind, or k3d works)
- `cert-manager` installed (see operator README)

```bash
git clone https://github.com/<your-username>/keldon-operator
cd keldon-operator
make build
make test
```

To deploy your local build against a cluster:

```bash
make docker-build IMG=localhost/keldon-operator:dev
kind load docker-image localhost/keldon-operator:dev   # if using kind

# Install via the public chart with your image override:
helm repo add keldon https://charts.keldon.io
helm install keldon-operator keldon/keldon-operator \
  --namespace keldon-system --create-namespace \
  --set image.repository=localhost/keldon-operator \
  --set image.tag=dev
```

### For docs (`keldon-docs`)

Docs are sourced into the website at build time. To preview locally, clone the website repo alongside this one and run `hugo serve`.

### For the Helm chart (`keldon-charts`)

```bash
helm lint charts/keldon-operator
helm template charts/keldon-operator
```

Chart version in `Chart.yaml` must be bumped for any change to be picked up by the publish workflow.

## License

By contributing, you agree that your contributions will be licensed under the [Apache License 2.0](LICENSE), the same license that covers the project.
