# SAST, DAST, and code scan template

Reusable GitHub Actions template for application security testing. It includes a small Node.js sample app so the workflows can run end to end, plus three scan pipelines you can copy into another repository.

This is the same project as [OkiriGabriel/SAST-DAST-workflow](https://github.com/OkiriGabriel/SAST-DAST-workflow).

| Pipeline | Workflow | What it does |
| --- | --- | --- |
| **SAST** | [`.github/workflows/sast.yml`](.github/workflows/sast.yml) | [CodeQL](https://codeql.github.com/docs/) static analysis of source |
| **DAST** | [`.github/workflows/dast.yml`](.github/workflows/dast.yml) | [OWASP ZAP](https://www.zaproxy.org/) baseline scan of a running app |
| **Code scan** | [`.github/workflows/code-scan.yml`](.github/workflows/code-scan.yml) | [Trivy](https://trivy.dev/) filesystem, IaC, and image scans, plus GitHub dependency review and `npm audit` |

Findings upload to **Security → Code scanning**. HTML/SARIF reports are also stored as workflow artifacts.

## Use this as a template

1. On GitHub, open **Settings → General → Template repository** and enable it.
2. Create a new repo from the template, or copy the three workflow files and the config files below.
3. Enable **Code scanning**, **Dependabot alerts**, **Dependabot security updates**, and **Secret scanning** under **Settings → Code security**.
4. Protect `main` so pull requests must pass **SAST**, **DAST**, and **Code scan**.

Required companion files:

```text
.github/workflows/sast.yml
.github/workflows/dast.yml
.github/workflows/code-scan.yml
.github/codeql/codeql-config.yml
.github/dependabot.yml
.zap/rules.tsv
.trivyignore
Dockerfile          # needed for the image scan
```

### Adapt SAST to another language

Edit the matrix in `.github/workflows/sast.yml`:

```yaml
- language: python          # or java-kotlin, go, csharp, ruby, ...
  build-mode: none          # compiled languages: autobuild or manual
```

Query packs live in [`.github/codeql/codeql-config.yml`](.github/codeql/codeql-config.yml).

### Adapt DAST to your app

The default job installs this repo, starts `npm start`, waits for `/health`, then scans `http://localhost:3000`.

To scan a deployed URL instead, change the ZAP `target` in [`.github/workflows/dast.yml`](.github/workflows/dast.yml) and remove the local install/start steps.

Tune ignored ZAP alerts in [`.zap/rules.tsv`](.zap/rules.tsv).

### Adapt code scan to your image

The image job builds `./Dockerfile` and tags it as `<repository-name>:<sha>`. Suppress documented CVEs in [`.trivyignore`](.trivyignore). The job fails the build when Trivy finds an unfixed **CRITICAL** vulnerability.

On pull requests, [dependency review](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review) fails on **HIGH** or worse new dependency advisories.

## Run the sample app

```bash
cp .env.example .env
npm ci
npm start
```

```bash
curl http://localhost:3000
curl http://localhost:3000/health
```

```bash
docker build -t secure-webapp-lab:latest .
docker run --rm -p 3000:3000 secure-webapp-lab:latest
```

## Local scans (optional)

```bash
# CodeQL is the CI SAST engine. Semgrep is a useful local stand-in:
# semgrep scan --config auto

docker build -t secure-webapp-lab:latest .
trivy image --severity CRITICAL,HIGH secure-webapp-lab:latest
trivy fs --scanners vuln,secret,misconfig .
trivy config .
```

## When each workflow runs

| Workflow | Pull request | Push to `main` | Weekly schedule | Manual |
| --- | --- | --- | --- | --- |
| SAST | yes | yes | Monday 00:00 UTC | `workflow_dispatch` |
| DAST | yes | | Monday 04:00 UTC | `workflow_dispatch` |
| Code scan | yes | yes | Monday 03:00 UTC | `workflow_dispatch` |

Each workflow also supports `workflow_call`, so another repository can reuse it:

```yaml
jobs:
  sast:
    uses: OkiriGabriel/SAST-DAST-workflow/.github/workflows/sast.yml@main
  dast:
    uses: OkiriGabriel/SAST-DAST-workflow/.github/workflows/dast.yml@main
  code-scan:
    uses: OkiriGabriel/SAST-DAST-workflow/.github/workflows/code-scan.yml@main
```

The called repository must allow access from the caller under **Settings → Actions → General → Access**.

## Sample app security baseline

`server.js` ships with Helmet CSP, HSTS, Permissions-Policy, and cache-control headers, plus `/health`, `/robots.txt`, and `/sitemap.xml`. That gives ZAP a clean baseline you can tighten or replace with your own app.

## Replacing the older workflow names

Earlier copies of this repo used `codeql-analysis.yml`, `zap-scan.yml`, `trivy-scan.yml`, `docker-scan.yml`, and `dependency-check.yml`. Those jobs are covered by the three template workflows above. Delete the old files after you adopt this layout so scans do not run twice.

## Resources

- [GitHub code scanning](https://docs.github.com/en/code-security/code-scanning)
- [CodeQL](https://codeql.github.com/docs/)
- [OWASP ZAP](https://www.zaproxy.org/)
- [Trivy](https://trivy.dev/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
