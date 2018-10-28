OpenShift CLI for CI/CD
=======================

This [Openshift command line tool](https://docs.openshift.com/enterprise/3.0/cli_reference/get_started_cli.html) docker
image ships `oc` and includes `gettext` so you can use `envsubst` to substitute
environment variables in your CI/CD pipeline, for example using in
[Jenkins](https://jenkins.io/) or a job in [GitLab CI .gitlab-ci.yml file](https://docs.gitlab.com/ce/ci/yaml/README.html#gitlab-ci-yml).

Examples
--------

Why should I use `envsubst`? You should never put secrets into your version
control, so you might want to keep them in secret variables in your CI/CD
system. You can use `envsubst` to substituted them correctly in your templates.

A CI/CD system usually sets a bunch of variables to the build environment.
Below I'll show an example of [GitLab CI variables set in the build environment](https://docs.gitlab.com/ce/ci/variables/#predefined-variables-environment-variables).

- `CI_PROJECT_NAME`: my_awesome_project
- `CI_BUILD_REF_SLUG`: f1234d

Your `app.yaml` could look similar the one below:

    $ cat app.yaml
    ...
    - apiVersion: v1
      kind: DeploymentConfig
      metadata:
        labels:
          app: ${CI_PROJECT_NAME}
        name: sample
      spec:
        replicas: 1
        selector:
          app: ${CI_PROJECT_NAME}
          deployment: ${CI_BUILD_REF_SLUG}
    ...

After `cat app.yaml | envsubst > app.yaml` you'll notice the variables have
been replaced with their actual values:

    $ cat app.yaml
    ...
    - apiVersion: v1
      kind: DeploymentConfig
      metadata:
        labels:
          app: my_awesome_project
        name: sample
      spec:
        replicas: 1
        selector:
          app: my_awesome_project
          deployment: f1234d
    ...

GitLab CI example
-----------------

Below a sample job in an `.gitlab-ci.yml` file, please note that OpenShift does
not allow `_` in project names:

    deploy:
      image: widerin/openshift-cli
      stage: deploy
      script:
        - oc login "$OPENSHIFT_SERVER" --token="$OPENSHIFT_TOKEN"
        - cat app.yaml | envsubst > app.yaml
        - oc replace -f app.yaml -n ${CI_PROJECT_NAME/_/}

```dockerfile
FROM frolvlad/alpine-glibc:latest

MAINTAINER Daniel Widerin <daniel@widerin.net>

ENV OC_VERSION=v3.11.0 \
    OC_TAG_SHA=0cbc58b \
    BUILD_DEPS='tar gzip' \
    RUN_DEPS='curl ca-certificates gettext'

RUN apk --no-cache add $BUILD_DEPS $RUN_DEPS && \
    curl -sLo /tmp/oc.tar.gz https://github.com/openshift/origin/releases/download/${OC_VERSION}/openshift-origin-client-tools-${OC_VERSION}-${OC_TAG_SHA}-linux-64bit.tar.gz && \
    tar xzvf /tmp/oc.tar.gz -C /tmp/ && \
    mv /tmp/openshift-origin-client-tools-${OC_VERSION}-${OC_TAG_SHA}-linux-64bit/oc /usr/local/bin/ && \
    rm -rf /tmp/oc.tar.gz /tmp/openshift-origin-client-tools-${OC_VERSION}-${OC_TAG_SHA}-linux-64bit && \
    apk del $BUILD_DEPS

CMD ["/usr/local/bin/oc"]
```
