# Securing your CI/CD Pipeline (Gitlab version)

### The goal of this project is to provide a fast and effortless way of integrating the security tools and stages that a modern CI/CD Pipeline and agile SDLC demands, thus effectively "shifting security left".

## Table of Contents

[[_TOC_]]

## Supported Stages and Tools

| STAGE | DESCRIPTION | TOOLS |
|-|-|-|
| Secrets Scanning | The `secrets-scanning` stage scans your codebase for passwords, private keys, credentials and sensitive information in general in order to prevent their deployment. | [git-secrets](https://github.com/awslabs/git-secrets): "Prevents you from committing passwords and other sensitive information to a git repository." <hr/> [truffleHog](https://github.com/dxa4481/truffleHog): "Searches through git repositories for high entropy strings and secrets." |
| Software Composition Analysis (SCA) | At the `sca` stage all project dependencies get scanned for known vulnerabilities. This is a crucial security check, especially when a big part of modern project's code comes from its dependencies. | [yarn audit](https://classic.yarnpkg.com/en/docs/cli/audit/) *(Node.js)*: "Checks for known security issues with the installed packages. The output is a list of known issues. You must be online to perform the audit." <hr/> [npm audit](https://docs.npmjs.com/cli/audit) *(Node.js)*: Similar to its yarn counterpart, but for npm. <hr/> [Safety](https://pypi.org/project/safety/) *(Python)*: "Safety checks your installed dependencies for known security vulnerabilities." <hr/> [SensioLabs Security Checker](https://github.com/sensiolabs/security-checker) *(PHP)*: "The SensioLabs Security Checker is a command line tool that checks if your application uses dependencies with known security vulnerabilities." <hr/> [bundler audit](https://github.com/rubysec/bundler-audit) *(Ruby)*: Similar to Node.js yarn and npm audit, but for Ruby. <hr/> [OWASP Dependency-Check](https://github.com/jeremylong/DependencyCheck): "Dependency-Check is a Software Composition Analysis (SCA) tool that attempts to detect publicly disclosed vulnerabilities contained within a project's dependencies." |
| Static Application Security Testing (SAST) | The `sast` stage analyses the project code and tries to find security vulnerabilities such as, but not limited to, hardcoded credentials, weak or inexisting input/ouput sanitization or known vulnerable code patterns. | [NodeJSScan](https://github.com/ajinabraham/nodejsscan) *(Node.js)*: As the name suggests, it is a static security code scanner for Node.js applications. <hr/> [bandit](https://pypi.org/project/bandit/) *(Python)*: "Bandit is a tool designed to find common security issues in Python code. To do this Bandit processes each file, builds an AST from it, and runs appropriate plugins against the AST nodes." <hr/> [Veracode](https://www.veracode.com/): "Veracode Static Analysis is a Static Application Security Testing (SAST) solution that enables you to quickly identify and remediate application security findings." It supports major frameworks and programming languages. |
| Container Scanning | By placing it after the a `build` stage, the `container-scanning` stage scans the newly-created docker image and reports both known security flaws and bad practices (e.g. container user is root). | [Dockle](https://github.com/goodwithtech/dockle): "Container Image Linter for Security". Compares docker image metadata against a set of "checkpoints" and report if it follows best-practices. <hr/> [Trivy](https://github.com/aquasecurity/trivy): "A Simple and Comprehensive Vulnerability Scanner for Containers and other Artifacts, Suitable for CI. Trivy detects vulnerabilities of OS packages (Alpine, RHEL, CentOS, etc.) and application dependencies (Bundler, Composer, npm, yarn, etc.)." |
| Dynamic Application Security Testing (DAST) | The `dast` stage aims to perform a set of black-box security tests against the target, just-deployed, application. This stage can be parameterized to perform either simple scans - as security headers or ssl/tls ciphers checks - or more complex scan by using, for example, the OWASP ZAProxy integration. | [Kubesec](https://kubesec.io/): "Security risk analysis for Kubernetes resources". Similar to Dockle in the sense that it checks for security best-practices, but used against Kubernetes resources (manifests). (Although, technically, Kubesec can't be considered a DAST tool, it was the author's decision to add it to this stage for now, as it runs as a post-deployment stage.) <hr/> [testssl](https://github.com/drwetter/testssl.sh): "testssl.sh is a free command line tool which checks a server's service on any port for the support of TLS/SSL ciphers, protocols as well as some cryptographic flaws." <hr/> [Qualys ssllabs-scan](https://github.com/ssllabs/ssllabs-scan): Similar to testssl.sh. **However, it requires the target host to be reachable via the Internet.** <hr/> [security-headers](https://github.com/koenbuyens/securityheaders): "The script (and burp plugin) validates whether the headers pertaining to security are present and if present, whether they have been configured securely. It implements checks identified by http://securityheaders.io/, https://csp.withgoogle.com, OWASPs cheat sheets and Original research." <hr/> [OWASP ZAProxy](https://www.zaproxy.org/): "The world's most widely used web app scanner." One of OWASP's flagship tools, Zed Attack Proxy can be used either as a manual tool by Penetration Testers, or in an automated fashion as a DAST tool (which is the way it's integrated here). |

### Specifically Supported Languages

> Languages that are supported by their own scanning tool (e.g. bandit for Python) and not only by general purpose scanners (as Veracode or OWASP Dependency Check).

* Javascript/Node.js
* Python
* PHP
* Ruby

## Table of Contents

[[_TOC_]]

## Quick Start

Simply use the `.gitlab-ci.yml` file below.

```yaml
variables:
  GIT_SSL_NO_VERIFY: "1"
  DOCKER_IMAGE_URL: ${DOCKER_REGISTRY}/images/${CI_PROJECT_NAMESPACE}-${CI_PROJECT_NAME}
  PIPELINE_TYPE: node

stages:
  - secrets-scanning
  - sca
  - sast
  - build
  - container-scanning
  - deploy
  - dast

include:
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-secrets-scanning-template.yml'
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-sca-template.yml'
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-sast-template.yml'
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-container-scanning-template.yml'
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-dast-template.yml'

.container-scanning:
  variables:
    DIND_IMAGE: docker:stable-dind
    DOCKER_HOST: tcp://localhost:2375
    DOCKER_DRIVER: overlay2

.dast:
  before_script:
    - echo "this should be replaced with a proper pod k8s manifest content!" > sec_dast_pod_manifest.yml
  variables:
    SEC_DAST_TARGET_URL: "https://thefriendlypirate.com"
    NO_INTERNET: "true"
    USE_KUBESEC: "true"
    USE_ZAPROXY: "true"

build:
  stage: build
  image: docker:stable
  services:
    - docker:stable-dind
  variables:
    DOCKER_HOST: tcp://localhost:2375
    DOCKER_DRIVER: overlay2
  script:
    - echo "build..."

deploy:
  stage: deploy
  image: bitnami/kubectl
  script:
  - echo "deploy..."

```

## Secrets Scanning

### Supported Tools

* git-secrets
* trufllehog

### Where to Start?

Just add the following snippets to your pipeline configuration file.

```yaml
stages:
  - ...
  - secrets-scanning
  - ...

include:
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-secrets-scanning-template.yml'
```

### Configuration Variables

| VARIABLE | DESCRIPTION | DEFAULT |
|-|-|-|
| `CI_PROJECT_DIR` | Project's root directory. | *`GITLAB DEFAULT`* |
| `SEC_TRUFFLEHOG_IMAGE` | Docker Image used for Trufflehog. | `thefriendlypirate/trufflehog:latest` |
| `SEC_GITSECRETS_IMAGE` | Docker Image used for git-secrets. | `thefriendlypirate/git-secrets:latest` |
| `TRUFFLEHOG_EXCLUSION_PATHS_FILE` | Path to the file that holds all the paths which should be excluded from trufllehog's scan. |  |

## Dependency Scanning / Software Composition Analysis (SCA)

### Supported Tools

* yarn audit
* npm audit
* Safety
* SensioLabs Security Checker
* bundler audit
* OWASP Dependency Check

### Where to Start?

Just add the following snippets to your pipeline configuration file.

```yaml
stages:
  - ...
  - sca
  - ...

include:
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-sca-template.yml'
```

### Configuration Variables

| VARIABLE | DESCRIPTION | DEFAULT |
|-|-|-|
| `CI_PROJECT_DIR` | Project's root directory. | *`GITLAB DEFAULT`* |
| `PIPELINE_TYPE` | This variable drives which tools/scanners should be run. Example: if set to `node`, it will trigger yarn/npm audit. Supported values: `node`, `python`, `php`, `ruby`. |  |
| `SEC_NODE_IMAGE` | Docker Image used for yarn/npm audit. | `thefriendlypirate/node:latest` |
| `SEC_SAFETY_IMAGE` | Docker Image used for safety. | `thefriendlypirate/safety:latest` |
| `SEC_SYMFONYSECURITYCHECK_IMAGE` | Docker Image used for symfony security:check. | `thefriendlypirate/symfony-securitycheck:latest` |
| `SEC_BUNDLERAUDIT_IMAGE` | Docker Image used for bundler audit. | `thefriendlypirate/bundler-audit:latest` |
| `SEC_DEPENDENCYCHECK_IMAGE` | Docker Image used for dependency-check. | `thefriendlypirate/dependency-check:latest` |

## Static Application Security Testing (SAST)

### Supported Tools

* Veracode
* NodeJsScan
* bandit

### Where to Start?

Just add the following snippets to your pipeline configuration file.

```yaml
stages:
  - ...
  - sast
  - ...

include:
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-sast-template.yml'

.sast:
  before_script:
    - zip -r0 ${CI_PROJECT_NAMESPACE}-${CI_PROJECT_NAME}.zip ${CI_PROJECT_DIR} || true
  variables:
    VERACODE_PACKAGE: "${CI_PROJECT_NAMESPACE}-${CI_PROJECT_NAME}.zip"
    USE_VERACODE: "true"
    VERACODE_ID: ""
    VERACODE_KEY: ""
    VERACODE_SCANVERSION: ""
    VERACODE_APPNAME: ""
    VERACODE_SANDBOXNAME: ""
```

### Configuration Variables

| VARIABLE | DESCRIPTION | DEFAULT |
|-|-|-|
| `CI_PROJECT_DIR` | Project's root directory. | *`GITLAB DEFAULT`* |
| `PIPELINE_TYPE` | This variable drives which tools/scanners should be run. Example: if set to `node`, it will trigger NodeJsScan. Supported values: `node`, `python`. |  |
| `SEC_NODEJSSCAN_IMAGE` | Docker Image used for NodeJsScan. | `thefriendlypirate/nodejsscancli:latest` |
| `SEC_BANDIT_IMAGE` | Docker Image used for bandit. | `thefriendlypirate/bandit:latest` |
| `SEC_VERACODE_IMAGE` | Docker Image used for Veracode. | `thefriendlypirate/veracode-scanner:latest` |
| `USE_VERACODE` | Enable Veracode scanner. | `false` |
| `VERACODE_PACKAGE` | Filepath to the already prepared project package (*.zip, *.war, ...). |  |
| `VERACODE_ID` | Veracode id for API auth. |  |
| `VERACODE_KEY` | Veracode key for API auth. |  |
| `VERACODE_APPNAME` | Veracode Application Profile name. |  |
| `VERACODE_SANDBOXNAME` | Veracode sandbox name (if sandboxes being used). |  |
| `VERACODE_SCANVERSION` | Veracode mandatory `version` parameter / scan name. |  |

## Container Scanning

### Supported Tools

* Dockle
* Trivy

Just add the following snippets to your pipeline configuration file.

```yaml
stages:
  - ...
  - container-scanning
  - ...

include:
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-container-scanning-template.yml'

.container-scanning:
  variables:
    DOCKER_HOST: tcp://localhost:2375
    DOCKER_DRIVER: overlay2
```

### Configuration Variables

| VARIABLE | DESCRIPTION | DEFAULT |
|-|-|-|
| `DIND_IMAGE` | Docker in Docker image. | `docker:stable-dind` |
| `SEC_DOCKLE_IMAGE` | Docker Image used for Dockle. | `thefriendlypirate/dockle:latest` |
| `SEC_TRIVY_IMAGE` | Docker Image used for Trivy. | `thefriendlypirate/trivy:latest` |
| `DOCKER_IMAGE_URL` | Target Docker Image URL in the form `<DOCKER_REGISTRY>/<PATH_TO_IMAGE>:<TAG>` (mandatory). |  |
| `DOCKER_REGISTRY` | Private Docker Registry host. |  |
| `DOCKER_REGISTRY_USER` | Docker Registry Username. |  |
| `DOCKER_REGISTRY_PASSWORD` | Docker Registry Password. |  |
| `TRIVY_TARGET_IMAGE_URL` | Target Docker Image URL in the form `<DOCKER_REGISTRY>/<PATH_TO_IMAGE>:<TAG>`. If not present, falls back to DOCKER_IMAGE_URL var. |  |
| `TRIVY_USERNAME` | Docker Registry username. If not present, falls back to DOCKER_REGISTRY_USER var. |  |
| `TRIVY_PASSWORD` | Docker Registry password. If not present, falls back to DOCKER_REGISTRY_PASSWORD var. |  |
| `TRIVY_INSECURE` | Set to `true` to disable certificate verification. | `false` |
| `DOCKLE_USERNAME` | Docker Registry username. If not present, falls back to DOCKER_REGISTRY_USER var. |  |
| `DOCKLE_PASSWORD` | Docker Registry password. If not present, falls back to DOCKER_REGISTRY_PASSWORD var. |  |
| `DOCKLE_TARGET_IMAGE_URL` | Target Docker Image URL in the form `<PATH_TO_IMAGE>` (without Docker Registry part; workaround for Dockle only). If not present, falls back to DOCKER_IMAGE_URL var. |  |
| `DOCKLE_INSECURE` | Set to `true` to disable certificate verification. | `false` |
| `DOCKLE_USE_PRIVATE_REGISTRY` |  | `false` |

## Dynamic Application Security Testing (DAST)

### Supported Tools

* Kubesec
* OWASP ZAProxy
* Qualys ssllabs-scan
* testssl
* security-headers

### Where to Start?

Just add the following snippets to your pipeline configuration file.

```yaml
stages:
  - ...
  - dast
  - ...

include:
  - project: 'joao.mauricio/security-integration-gitlabci'
    file: '.security-dast-template.yml'

.dast:
  before_script:
    - kubectl -n ${NAMESPACE} get pod $(kubectl -n ${NAMESPACE} get pods --selector=xxx=yyy --no-headers | cut -d" " -f1 | head -n 1) -o yaml > sec_dast_pod_manifest.yml || true
```

### Configuration Variables

| VARIABLE | DESCRIPTION | DEFAULT |
|-|-|-|
| `SEC_DAST_TARGET_URL` | URL of the target (mandatory). |  |
| `USE_KUBESEC` | Enable Kubesec. | `false` |
| `USE_ZAPROXY` | Enable ZAProxy. | `true` |
| `sec_dast_pod_manifest.yml` *(file)* | File holding a pod manifest, to be used by kubesec. |  |
| `NO_INTERNET` | Set to `true` if the target URL isn't reachable from the Internet. | `false` |
| `SEC_KUBELINT_IMAGE` | Docker Image used for kubesec. | `thefriendlypirate/kubesec:latest` |
| `SEC_SSLLABSSCAN_IMAGE` | Docker Image used for ssllabs-scan. | `thefriendlypirate/ssllabs-scan:latest` |
| `SEC_TESTSSL_IMAGE` | Docker Image used for testssl. | `thefriendlypirate/testssl:latest` |
| `SEC_SECURITYHEADERS_IMAGE` | Docker Image used for security-headers. | `thefriendlypirate/securityheaders:latest` |
| `SEC_ZAPROXY_IMAGE` | Docker Image used for ZAProxy. | `thefriendlypirate/zaproxy:latest` |
| `ZAPROXY_FULL_SCAN` | Flag indicator whether a ZAProxy Full Scan should be performed (instead of the default Baseline Scan). | `false` |

## Author

[@joao.mauricio](https://gitlab.com/joao.mauricio) (João Maurício)
