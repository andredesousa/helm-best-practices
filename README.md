# Helm Best Practices

Helm is a package manager for Kubernetes applications, providing mechanisms to template and group Kubernetes manifests as versioned packages.
These tips are based on books, articles and professional experience.

## Table of Contents

1. [Follow Helm conventions](#follow-helm-conventions)
2. [Use Helm create command](#use-helm-create-command)
3. [Explore public charts](#explore-public-charts)
4. [Do not put everything in one chart](#do-not-put-everything-in-one-chart)
5. [Use subcharts to reuse code](#use-subcharts-to-reuse-code)
6. [Have a Git repository](#have-a-git-repository)
7. [Have a chart repository](#have-a-chart-repository)
8. [Test templates and charts](#test-templates-and-charts)
9. [Generate documentation for charts](#generate-documentation-for-charts)
10. [Sign and verify charts](#sign-and-verify-charts)
11. [Scan charts regularly](#scan-charts-regularly)
12. [Keep the deployments idempotent](#keep-the-deployments-idempotent)
13. [Rollbacks with version control](#rollbacks-with-version-control)
14. [Automatically roll deployments](#automatically-roll-deployments)
15. [Use template functions](#use-template-functions)
16. [Use lint, template, and debug commands](#use-lint-template-and-debug-commands)
17. [Use small-sized images](#use-small-sized-images)
18. [Avoid privileged containers](#avoid-privileged-containers)
19. [Limit container resources](#limit-container-resources)
20. [Include health/liveness checks](#include-healthliveness-checks)
21. [Include logging and monitoring tools](#include-logging-and-monitoring-tools)
22. [Use labels to find resources](#use-labels-to-find-resources)
23. [Separate values by environments](#separate-values-by-environments)
24. [Store secrets encrypted](#store-secrets-encrypted)
25. [Take advantage of Helm hooks](#take-advantage-of-helm-hooks)

## Follow Helm conventions

Code conventions are base rules that allow the creation of a uniform code base across an organization.
Following them does not only increase the uniformity and therefore the quality of the code.
The Helm documentation provides some [General Conventions](https://helm.sh/docs/chart_best_practices/conventions/) and [Best Practices](https://helm.sh/docs/chart_best_practices/).
These conventions include naming conventions, templates structure and organization, dependencies, labels and annotations, Pods and PodTemplates, custom resource definitions, and role-based access control.
[Developer's Guide to Writing a Good Helm Chart](https://boxunix.com/2022/02/05/developers-guide-to-writing-a-good-helm-chart/) is a recommended read.

## Use Helm create command

It is a common practice, who is new to Kubernetes, their first inclination is to copy some manifests from Kubernetes documentation and handcraft their Helm chart from scratch.
However, this approach is error-prone and cumbersome.
Instead, the command `helm create <my-chart>` will provide all the common Kubernetes manifests (*deployment.yaml*, *hpa.yaml*, *ingress.yaml*, *service.yaml*, and *serviceaccount.yaml*) as well as helper templates to circumvent resource naming constraints and labels/annotations.
Simply edit the *values.yaml* file to deploy your Docker image.

## Explore public charts

Perhaps Helm's biggest advantage is its large community.
When you get stuck, take some time to explore some public charts for inspiration or neat solutions to common tasks.
To search for charts, you can use the built-in [helm search](https://helm.sh/docs/helm/helm_search/) command:

```bash
helm search hub <chart>
```

This command will search for charts in the Artifact Hub or your own hub instance.
Other option, `helm search repo`, it searches repositories for a keyword in charts.

## Do not put everything in one chart

If you have multiple services that need to be deployed, it's better to create a separate chart for each service.
This will make it easier to manage your deployments and upgrades, and it will also make it easier to troubleshoot if something goes wrong.
It's also a good idea to create a shared chart for common resources, such as config maps and secrets.
This way, you can reuse these resources across all of your charts, and you won't have to duplicate them.

## Use subcharts to reuse code

Creating or using separate charts for the standalone applications and adding them to the parent charts is therefore recommended.
Subcharts are deployed with the main chart at the same time. Values for subcharts can be provided in the same *values.yaml* file for the main chart.
The folder structure should be in the following order:

```html
├── Chart.yaml
├── charts
|  ├── backend
|  |  ├── Chart.yaml
|  |  ├── templates
|  |  |  ├── deployment.yaml
|  |  |  ├── secrets.yaml
|  |  |  ├── ...*.yaml
|  |  |  └── service.yaml
|  |  └── values.yaml
|  ├── database
|  └── frontend
├── templates
└── values.yaml
```

## Have a Git repository

Having a Git repository will give you some benefits just by using it, such as easy rollbacks or the ability to track changes.
By sharing your code in a Git repository, you can easily collaborate with others on your team.
Also, maintaining the source code on Git repository is of utmost importance and is considered one of the best practices for CI/CD.
A revision/version control system is also important to track changes, identify differences, and maintaining an environment that eases the task to keep track of the chart releases.

## Have a chart repository

A [chart repository](https://helm.sh/docs/topics/chart_repository/) is an HTTP server that houses an *index.yaml* file and optionally some packaged charts.
In other words, a chart repository is simply a place where packaged charts can be stored and shared.
By sharing your Helm charts in a central repository, you can easily collaborate with others on your team, as well as share your work with the larger Helm community.
Once you've set up your Helm chart repository, you can push your charts to it using the [helm push](https://helm.sh/docs/helm/helm_push/) command.

## Test templates and charts

Helm charts consist of multiple resources that are to be deployed to the cluster.
It is essential to check that all the resources are created in the cluster with the correct values.
It is recommended to write tests for your charts and to run them after the installation.
For example, you can use the `helm test <release-name>` command to run tests.
Alternatively, [helm unittest](https://github.com/helm-unittest/helm-unittest) is a BDD-styled unit testing framework for Helm charts as a Helm plugin.

## Generate documentation for charts

As with any piece of software, documentation is key to usability.
Within a Helm chart, you can use inline comments, the *NOTES.txt*, and the *README* file.
The easiest way to generate and maintain docs is to use a Golang package named [helm-docs](https://github.com/norwoodj/helm-docs).
With *helm-docs* you can generate the *README* containing tables of values, versions, and description taken from *values.yaml* and *Chart.yaml*.
*helm-docs* can be integrated into [pre-commit](https://pre-commit.com/) along with linting.

## Sign and verify charts

[Sign your charts](https://helm.sh/docs/topics/provenance/) with `helm package –sign` and verify them with `helm install --verify`.
Asserting the integrity of the software components is the most common task when securing the software supply chain.
Integrity is established by comparing a chart to a provenance record.
Helm uses a PGP-based digital signature to create provenance records stored in provenance files (*.prov*), which are stored alongside a packaged chart.

## Scan charts regularly

The [CIS Kubernetes Benchmarks](https://www.cisecurity.org/benchmark/kubernetes) highlight numerous ways an object deployed to Kubernetes can be given too many permissions if specific configuration is omitted from the Kubernetes YAML defining the object.
It also makes recommendations against configuration options that may provide a larger surface area for attack.
Helm chart misconfigurations will look the same as that of a Kubernetes YAML manifest misconfiguration.
You can perform automated vulnerability scans for Helm Charts by integrating with a CI pipeline.
You can use tools like [Trivy](https://github.com/aquasecurity/trivy) or [Checkov](https://github.com/bridgecrewio/checkov) to find vulnerabilities/misconfigurations in Kubernetes YAML files.

## Keep the deployments idempotent

An idempotent operation is one you can apply many times without changing the result following the first run.
You can keep deployments idempotent by using `helm upgrade --install` command instead of `install` and `upgrade` separately.
It installs the charts if they are not already installed.
If they are already installed, it upgrades them.
Furthermore, you can use `--atomic` flag to rollback changes in the event of a failed operation during helm upgrade.
This ensures the Helm releases are not stuck in the failed state.

## Rollbacks with version control

There are scenarios where the test team could come across some issues which were not observed in the previous software release.
A probable reason could be a side-effect of a fix that is pushed in the release software which is under test.
In such cases, the developer who pushed that fix should be able to rollback his changes so that the release is not stalled, and he also gets some more time to re-look at his implementation.
Without version control system, such a seamless rollback is not possible.
The rollback is not limited to source code. It can be extended to documents, presentations, flow diagrams, etc.

## Automatically roll deployments

It is common to have ConfigMaps or Secrets mounted to containers.
Although the deployments and container images change with new releases, the ConfigMaps or Secrets do not change frequently.
The *sha256sum* function can be used to ensure a deployment's annotation section is updated if another file changes:

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

## Use template functions

Helm includes many [template functions](https://helm.sh/docs/chart_template_guide/function_list/) you can take advantage of in templates.
The functions are defined in the [Go template language](https://pkg.go.dev/text/template) and [Sprig template library](https://masterminds.github.io/sprig/).
Template functions are very useful when creating Helm charts.
They can improve templating, reduce code duplication, and can be used to validate values before deploying your applications to Kubernetes.
See the next example using the *upper* function:

```yaml
kind: ConfigMap
data:
  regions: {{ upper .Values.regions }}
```

## Use lint, template, and debug commands

Use of a linter to avoid common mistakes and establish best practice guidelines that engineers can follow in an automated way.
Use [helm lint](https://helm.sh/docs/helm/helm_lint/) command for verifying that your chart follows best practices.
Alternatively, [KubeLinter](https://github.com/stackrox/kube-linter) and [Kubeval](https://github.com/instrumenta/kubeval/) can be used to parse Kubernetes YAML files and Helm charts , and are often used locally as part of a development workflow as well as in CI pipelines.
The [helm template](https://helm.sh/docs/helm/helm_template/) command will render chart templates and display the output.
The `helm install --dry-run --debug` or `helm template --debug` commands are a way to have the server render your templates, then return the resulting manifest file.
The `helm get manifest` command is a good way to see what templates are installed on the server.
The `helm get values` command is used to retrieve the release values installed to the cluster.

## Use small-sized images

If your image is based on a full-blown OS distribution like Ubuntu or Centos, you will have a bunch of tools already packaged in the image.
The image size will be larger, but you don't need most of these tools in your application images.
In comparison by using smaller images with leaner OS distributions, which only bundle the necessary system, you need less storage space, minimize the attack surface, and ensure you create more secure images.
The best practice here would be to select an image with a specific version based on a leaner OS distribution like [Alpine](https://hub.docker.com/_/alpine).

## Avoid privileged containers

Ensuring that a container can perform only a very limited set of operations is vital for production deployments.
This is possible thanks to the use of non-root containers, which are executed by a user different from root.
You can restrict the container capabilities to the minimal required set using [securityContext.capabilities.drop](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container) option.
That way, in case your container is compromised, the range of action available to an attacker is limited.

## Limit container resources

By default, a container has no resource constraints and can use as much of a given resource as the host's kernel scheduler allows.
It's a good idea to limit the memory and CPU usage of your containers, especially if you're running multiple containers.
Beyond that, when a container is compromised, attackers may try to make use of the underlying host resources to perform malicious activity.
Set [resource requests and limits of Pods and containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) to minimize the impact of breaches for resource-intensive containers.

## Include health/liveness checks

[Health checks](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes) are a simple way to let the system know if an instance of your app is working or not working.
Many applications running for long periods of time eventually transition to broken states and cannot recover except by being restarted.
By default, Kubernetes starts to send traffic to a pod when all the containers inside the pod start and restarts containers when they crash.
While this can be "good enough" when you are starting out, you can make your deployments more robust by creating [custom health checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

## Include logging and monitoring tools

It is essential having our deployments properly monitored so we can early detect potential issues.
It also essential to have application usage, cost, and resource consumption metrics.
In order to gather this information, you would commonly deploy logging stacks like ELK ([ElasticSearch](https://www.elastic.co/elasticsearch/), [Logstash](https://www.elastic.co/logstash/), and [Kibana](https://www.elastic.co/kibana/)) and monitoring tools like [Prometheus](https://prometheus.io/).
When writing your chart, make sure that your deployment is able to work with these tools.
To do this, make sure all the containers log to `stdout/stderr` and Prometheus exporters are included.

## Use labels to find resources

Labels are crucial to Kubernetes' internal operations and the daily work of Kubernetes operators.
Almost every resource in Kubernetes offers labels for different purposes such as grouping, resource allocation, load balancing, or scheduling.
A single Helm command will allow you to install multiple resources.
But it's important to know where these resources originate.
Labels enable you to find your resources created by Helm releases quickly.

## Separate values by environments

Default values are specified for most settings in a *values.yaml* file and these values are passed into the chart.
You can maintain the values YAML files at the root folder namely *values-[env].yaml*.
For example, *values-dev.yaml* and *values-prod.yaml* for *dev* and *prod* environments, respectively.
Each value file is used depending on the environment for the deployment.
A [values file](https://helm.sh/docs/chart_template_guide/values_files/) if passed into `helm install` or `helm upgrade` with the `-f` flag.

## Store secrets encrypted

Sensitive data, such as keys or passwords, are stored as secrets in Kubernetes.
Base64 is an encoding algorithm, not an encryption one, and despite its name, Kubernetes secrets are not secret.
Some folks prefer to store encrypted secrets along with the code, while others prefer to keep them in a different location.
There are a few alternatives worth mentioning, including [helm-secrets](https://github.com/jkroepke/helm-secrets) plugin, [Hashicorp Vault](https://github.com/hashicorp/vault), [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets), [Mozilla's SOPS](https://github.com/mozilla/sops), and more.

## Take advantage of Helm hooks

Oftentimes a Helm chart developer may want to perform some auxiliary operations before installing the main service or upgrading the main service that are required for the main service to function in the correct manner.
[Helm hooks](https://helm.sh/docs/topics/charts_hooks/) are simple Kubernetes manifest templates identified by an annotation whose value will determine when the hook should be rendered.
Helm hooks allow you an opportunity to perform operations at strategic points in a release lifecycle.

## Bibliography

- [13 Best Practices for using Helm](https://codersociety.com/blog/articles/helm-best-practices)
- [Best Practices for Creating Production-Ready Helm charts](https://docs.bitnami.com/tutorials/production-ready-charts/)
- [Best practices for deploying to Kubernetes using Helm](https://itnext.io/best-practices-for-deploying-to-kubernetes-using-helm-73be1f3040d2)
- [Developer's Guide to Writing a Good Helm Chart](https://boxunix.com/2022/02/05/developers-guide-to-writing-a-good-helm-chart/)
- [Chart Development Tips and Tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/)
- [Helm 101 for Developers](https://levelup.gitconnected.com/helm-101-for-developers-1c28e734937e)
- [Helm Best Practices and Recommendations](https://kodekloud.com/blog/helm-best-practices/)
- [Helm Chart - Development Guide](https://medium.com/swlh/helm-chart-development-guide-bbc525d3b448)
- [Helm Chart Hooks Tutorial](https://rafay.co/the-kubernetes-current/helm-chart-hooks-tutorial/)
- [Helm Charts Best Practices](https://www.linkedin.com/pulse/helm-charts-best-practices-my-experience-suresh-palemoni/)
- [Helm security and best practices](https://sysdig.com/blog/how-to-secure-helm/)
- [Scan Helm charts for Kubernetes misconfigurations with Checkov](https://bridgecrew.io/blog/scan-helm-charts-for-kubernetes-misconfigurations-with-checkov/)
- [Steering Straight with Helm Charts Best Practices](https://jfrog.com/blog/helm-charts-best-practices/)
