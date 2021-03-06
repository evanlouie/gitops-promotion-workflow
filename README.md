# GitOps Environment Promotion Workflow

This is a living document outlining a potential solution/workflow for promoting new releases through environment gates
in the context of GitOps.

| Version | Date       | Description |
| ------- | ---------- | ----------- |
| 1.0.0   | 2019-03-18 | Initial     |

## Preamble

### Core Ideas & Concepts

#### GitOps & Flux

GitOps has become one of the defacto ways of maintaining the state of ones cluster. The idea is simple, Git by default
gives users a means to version control documents and when combined with vendors such as GitHub, one can even add branch level
guards and pre/post-triggers for actions such as merging. Adding to this, by utilizing tools such as 
[Flux](https://github.com/weaveworks/flux), we can point our cluster to a Git repository of Kubernetes manifests representing
the defacto state of what should be on the cluster; and as of [v1.11.0](https://github.com/weaveworks/flux/releases/tag/1.11.0), what
[should no longer be there](https://github.com/weaveworks/flux/blob/1.11.0/site/garbagecollection.md).

#### Fabrikate

Taking the ideas of Helm and adding an extra layer of configuration on top of it, [Fabrikate](https://github.com/Microsoft/fabrikate/)
allows users to compose and nest Helm charts and vanilla K8s manifests together into what are called High Level Definitions.
Fabrikate can materialize these HLDs into standard K8s manifests which we can then `apply` to our cluster.

## The Problem

In real life scenarios, the standard GitOps approach needs guidance and best practices to make it applicable to real world
development pipelines. More often then not, we will have multiple clusters and environments within those clusters each representing
different deployments and states of our system.

Moving forward, we will focus on the scenario where we have 3 environments we wish to promote our changes through:

- `dev`
- `stage`
- `prod`

We can go under the assumption that our CI pipeline has setup guards to only allow merging from `dev` to `stage` and `stage` to `prod`.

With this example in mind, we need a means for development teams to promote there changes from `dev` to `stage` to `prod` via
the GitOps workflow and Fabrikate. 

One potential solution is set image-tag config in the cluster level HLD, however this quickly becomes unwieldy as the cluster HLD config
will now have to be editable by every developer on every team which has an application on the cluster. You could also 
have your CI pipeline programmatically keep track of and inject image-tag numbers into the config specific for target environments. 
However this will require extra tracking of state for the CI pipeline and make reversions or cancellations of workflows
difficult (requiring the tracking of changes and subsequent undoing of any changes to the manifest repository).

## The Solution

### Rationale

The proposed workflow aims to solve the problem outlined while:

- Minimizing the amount of state which the CI system has to keep track of.
  - The proposed workflow holds **no** state in the CI system. Using only GitHub triggers.
- Materialization of HLD's (whether they be cluster or application level) should be idempotent; and safe to call n+1 times.
  - The materialzation of the cluster HLD is a pure idempotent function, using its own config and `subcomponents` as args; no runtime variables needed.
- Reduce the overhead for DevOps teams; assistance from a DevOps teams should not be required by a dev team to deploy a new release of an application.
  - The proposed workflow, once setup in the application repositories, requires no DevOps assistance or monitoring.
  - From the developer prospective, they only need to interact with the GitHub (no interaction with CI tooling).

### State of the System

- A clusters manifests are generated via a single Fabrikate HLD which is comprised of many application HLDs.
- A clusters manifests repository has a branch for every environment the the cluster can have; in this case 3: `dev`, `stage`, and `prod`.
- A clusters Flux deployment points to a branch of the  clusters manifest repository.
- Every application deployed to a cluster contains their own HLD and is hosted on their own Git repository.
- Every application repository has at minimum 4 branches: `master`, `dev`, `stage`, `prod`.
- Every application repository contains a directory which contains their Fabrikate component definition.
- Every application repository has guards on protected branches to only allow merging from white-listed branches (ie. `dev` to `stage` and `stage` to `prod`)

### The Workflow

The clusters manifests will be generated via a single HLD which will encompass all applications which should be present on the cluster.

**Note:** This cluster HLD can also differ per branch.

On `dev`:

```yaml
name: "my-cluster"
subcomponents:
  # DevOps tools
  - name: "cloud-native"
    source: "https://github.com/timfpark/fabrikate-cloud-native"
    method: "git"
  # Applications
  - name: "cat-application"
    source: "https://github.com/some/git-repository-for-cat"
    method: "git"
    branch: dev # Tells Fabrikate to checkout the `dev` branch
    path: 'hld' # Tells Fabrikate where in the repository the component definition is
  - name: "dog-application"
    source: "https://github.com/some/git-repository-for-dog"
    method: "git"
    branch: dev
    path: 'hld'
```

On `stage`:

```yaml
name: "my-cluster"
subcomponents:
  # DevOps tools
  - name: "cloud-native"
    source: "https://github.com/timfpark/fabrikate-cloud-native"
    method: "git"
  # Application
  - name: "cat-application"
    source: "https://github.com/some/git-repository-for-cat"
    method: "git"
    branch: stage # Different branch of application HLD
    path: 'hld' 
  - name: "dog-application"
    source: "https://github.com/some/git-repository-for-dog"
    method: "git"
    branch: stage
    path: 'hld'
```

My `cat-application` and `dog-application` will have their own per-environment Fabrikate component definitions.

`./hld/component.yaml`:

```yaml
name: cat-application
generator: helm
path: ./cat-chart
```

`./hld/config/common.yaml`:

```yaml
config:
  image:
    repository: some-repo/cat-app
    tag: v1.1.0
```

In this example, `./` is the root directory of the application code, `./hld` contains our Fabrikate component, 
and `./hld/cat-chart` contains a helm chart for the `cat-application`.

By placing the the `cat-application`'s Fabrikate component and Helm chart into the application git repository itself, we 
have enabled DevOps teams to hand off the ability for development teams to manage for themselves what is deployed
to the cluster. As well as given the application HLD's a coupling too the code they are deploying and their own git
git system to utilize for versioning.

#### Developer Experience/Workflow

Workflow Sequence Diagram: 
![alt text][sequence-diagram]

[sequence-diagram]: sequence-diagram.png "Promotion Workflow Sequence Diagram"
