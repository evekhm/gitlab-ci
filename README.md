# GitLab CI

**IMPORTANT**: Do not rename the CI template `.gitlab-ci-build-template.yml` or change the visibility of this project. Doing so would break the CI pipelines of projects that "include:" the CI template.

## Why



## Usage

### GitLab CI
Simply use our template:

```yaml
include:
  - project: 'gcp-solutions/hcls/claims-modernization/gitlab-ci'
    file: '.gitlab-ci-build-template.yml'
```
