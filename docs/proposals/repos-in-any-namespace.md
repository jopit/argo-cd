---
title: Repositories in any namespace
authors:
  - "@jopit" # Authors' github accounts here.
sponsors:
  - TBD        # List all interested parties here.
reviewers:
  - "@alexmt"
  - TBD
approvers:
  - "@alexmt"
  - TBD

creation-date: 2024-01-26
last-updated: 2024-01-26
---

# Allow repositories to be specified in other namespaces

Allow users to specify repositories and repository credential templates in
non-control-plane namespaces to improve the self-service model of Argo CD.

## Summary

Argo CD currently only supports specifying repositories and repository
credential templates in the namespace where Argo CD is running (the
control-plane namespace). This means configuring the repositories and/or
credential templates needed by a user's applications requires an administrator.
In organizations which are making use of muti-tenancy (where a single Argo CD
instance is shared between multiple users), this increases the burden on
administrators and increases the friction experienced by users as they create
and manage their applications.

Adding the ability for users to create and manage the repositories and
credential templates for their application in the same namespace as the
application itself will improve the user experience and relieve administrators
of the need to manage these secrets.

## Motivation

As a user of Argo CD, I can create and manage applications in my own
non-control-plane namespace, but I cannot create and manage the secrets defining
the repositories and credential templates that these applications rely on. I
need to request someone with access to Argo CD's control-plane namespace to
create and manage these secrets for me.

Allowing me to manage the repositories and credential templates alongside the
application would improve my user experience.

### Goals

* Allow users to declaratively manage repositories and credential templates
  without intervention by an administrator.
* Allow users to declaratively manage repositories and credential templates
  without needing access to the control-plane namespace.

### Non-Goals

* Allowing users to share repositories and credential templates across
  namespaces. Applications in one namespace will not be able to use the
  repositories and credential templates defined in another non-control-plane
  namespace.
* Allowing users to specify completely arbitrary repositories and credential
  templates. The repository URLs used by an application will still need to match
  what is specified in the `.spec.sourceRepos` field of the application's
  AppProject.
* Allowing users to configure the TLS certificates used to verify the
  authenticity of the repositories and credential templates specified in their
  non-control-plane namespace. Certificates that are self-signed or are signed
  by a custom CA must still be managed through the `argocd-tls-certs-cm`
  ConfigMap in the control-plane namespace.

## Proposal

This proposal only applies to cluster-scoped instances of Argo CD, since the
Argo CD instance would need to access secrets across the cluster.

The proposal is to extend Argo CD to allow applications to use repositories in
the same non-control-plane namespace as the application itself. To make use of
this the _applications in any namespace_ feature would need to be enabled for
the Argo CD instance by specifying the `--application-namespaces` flag. This
flag specifies the namespaces in which applications can be created in and
therefore also the namespaces in which repositories and credential templates
would be allowed to be created.

Applications in a non-control-plane namespace must already belong to a project
which lists that namespace in its `.spec.sourceNamespaces` field. The repository
URL used by an application must already be allowed by the `.spec.sourceRepos`
field of the project that it belongs to. What changes is how Argo CD finds the
repository or credential template for the repository URL. The proposal is that
Argo CD would first look in the same namespace as the application, and if they
are not found it would then look in the control-plane namespace (this preserves
backward compatibility).

Note that one consequence is that now it's possible for there to be multiple
repositories defined for a given repository URL. However, each of these
repositories can only be defined in a separate namespace. Argo CD will need to
use the namespace to disambiguate them.

No changes are needed to the definition of the repository or credential template
yaml.

### Use cases

#### Use case 1:

As a user, I would like to declaratively manage repositories and credential
templates in my non-control-plane namespace without requiring intervention by an
administrator.

#### Use case 2:

As an Argo CD admin, I would like to enable my users to manage manage
repositories and credential templates in a self-service manner without then
needing additional RBAC privileges. However, I should still be able to control
access to repositories in general because the namespaces and repository URLs
specified by user applications must be allowed by the project that the
application belongs to.

### Implementation Details/Notes/Constraints

#### Support for watching secrets across the cluster

Repositories and credential templates are configured using Kubernetes secrets.
Argo CD watches secrets in the Argo CD control-plane namespace using an
informer. The scope of the informer would need to be updated to watch secrets in
any namespace. This can be done by setting the namespace parameter to an empty
string.

Argo CD should only reconcile secret events from namespaces that are specified
by the `--application-namespaces` flag. If the flag is not set, Argo CD should
only watch secrets in the control-plane namespace.

#### Changes to the UI and CLI
The `argocd repo` and `argocd repocreds` commands must accept a namespace flag
that indicates the namespace in which the secret for the repository or
credential template will be created.

The repositories view in the UI should display the namespace of the repository
or credential template. It should also allow new repositories or credential
template to be created in non-control-plane namespaces.

### Detailed examples

Assume there are two users of a single Argo CD cluster instance. Each one has a
namespace in which they wish to create an application. The applications each
have a different git URL. Each user wishes to define the repository for their
application in their own namespace.

There is an application project in the control-plane namespace which lists both
user's namespace in the `.spec.sourceNamespaces` field. This allows each user to
create applications and repositories in their own namespace. The application
project's `.spec.sourceRepos` field contains a pattern which will match each
user's git repository URL. Each user creates an application in their own
namespace which references the application project in it's `.spec.project`
field.

Each user also creates a secret in their own namespace which defines the
repository to be used by their application.

```yaml
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
  name: the-project
  namespace: argocd
spec:
  sourceNamespaces:
  - user1-namespace
  - user2-namespace
  sourceRepos:
  - 'https://github.com/argoproj/*'
---
kind: Secret
apiVersion: v1
metadata:
  name: private-repo
  namespace: user1-namespace
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/argoproj/user1-private-repo
  username: user1-username
  password: user1-password
---
kind: Application
apiVersion: argoproj.io/v1alpha1
metadata:
  name: user1-app
  namespace: user1-namespace
spec:
  project: the-project
  source:
    repoURL: https://github.com/argoproj/user1-private-repo
    path: manifests
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: user1-app-namespace
---
kind: Secret
apiVersion: v1
metadata:
  name: private-repo
  namespace: user2-namespace
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/argoproj/user2-private-repo
  username: user2-username
  password: user2-password
---
kind: Application
apiVersion: argoproj.io/v1alpha1
metadata:
  name: user2-app
  namespace: user2-namespace
spec:
  project: the-project
  source:
    repoURL: https://github.com/argoproj/user2-private-repo
    path: manifests
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: user2-app-namespace
```

### Security Considerations

This proposal should not introduce any new security risks. Applications in one
non-control-plane namespace will not be able to use repositories defined in
another non-control-plane namespace. The proposal will not give applications
access to repositories in the control-plane namespace that they did not have
access to before.

### Risks and Mitigations

An attacker with the correct permissions to an allowed non-control-plane
namespace could try to affect the performance of Argo CD by creating a large
number of repository secrets.  However, this is also true for a rogue party
using the Argo CLI with appropriate Argo CD RBAC permissions to create
repositories. Also, repository creation doesn't cause Argo CD to do a huge
amount of work, unlike for example Applications.

### Upgrade / Downgrade Strategy

When upgrading, no changes are required to an existing Argo CD instance to keep
previous behaviour. Once upgraded, the instance would need to be restarted with
the `--application-namespaces` flag in order to begin to use this feature.

Downgrading would be more difficult. The user would have to migrate any
repositories defined in a non-control-plane namespace into the control-plane
namespace.

## Open Questions

The proposal would make this feature available in any Argo CD instance which has
enabled the _applications in any namespace_ feature. Should we instead make it
so that users have to explicitly opt in to this feature in addition to the
_applications in any namespace_ feature?

## Drawbacks

If users have made use of this feature, it is hard to downgrade.

## Alternatives

Continue with the current approach of managing repositories and credential
templates in a control-plane namespace.
