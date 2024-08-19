# KEP-2535: Ensure Secret Pulled Images

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Image Pulling Scenarios](#image-pulling-scenarios)
  - [Kubelet Caching](#kubelet-caching)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Deprecation](#deprecation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks [optional]](#drawbacks-optional)
- [Alternatives [optional]](#alternatives-optional)
- [Infrastructure Needed [optional]](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/website]: https://git.k8s.io/website

## Summary

Give the admin the ability to ensure pods that use an image are authorized to access that image. This will culminate in changes to the `IfNotPresent` and
`Never` pull policies, as the `Always` policy will go through an authentication check each time.

When this feature is enabled, and an image in a pod request has not been successfully pulled with the given credentials
(or successfully pulled in the past with no credentials), then the kubelet will consider the credentials unauthenticated.
Thus even if the image is present, it may still be reauthenticated.
- For `IfNotPresent` images, the kubelet will re-pull the image
- For `Never` images, the image creation will fail

This behavior mirrors what would happen if an image was not present on the node (as opposed to present, but yet to be authorized with the credentials present).

This will be enforced for both `IfNotPresent` and `Never` policies.

Kubelet configuration will give the administrator the ability to choose how pre-loaded images should be verified:
1. *Do not* allow access to *any* pre-loaded images without verification.
2. Only allow access to a *specific list* of pre-loaded images without verification.
3. *Allow* access to *all* pre-loaded images without verification (default behavior).

The administrator will also have a choice to allow access to all images without
verification - this is the default behavior without this feature - kubelet configuration
field `noPullImageSecretRecheck`.

**TODO**: describe the kubelet configuration option for allow-listing images
**TODO**: do we want to allow disabling the feature (== reverting back to the old behavior)?

During development, this feature will be guarded by a feature gate.

*** The issue and these changes improving the security posture without requiring the forcing of pull always, will be documented in the kubernetes image pull policy documentation. ***

## Motivation

There have been customer requests for improving upon kubernetes' ability to
secure images pulled with auth on a node. Issue
[#18787](https://github.com/kubernetes/kubernetes/issues/18787) has been around
for a while.

To secure images one currently needs to inject `Always` `ImagePullPolicy` into pod
specs via an admission plugin. As @liggitt [notes](https://github.com/kubernetes/kubernetes/issues/18787#issuecomment-532280931)
the `pull` does not re-pull already-pulled layers of the image, but simply
resolves/verifies the image manifest has not changed in the registry (which
incidentally requires authenticating to private registries, which enforces the
image access). That means in the normal case (where the image has not changed
since the last pull), the request size is O(kb).

However, the `pull` does put the registry in the critical path of starting a container,
since an unavailable registry will fail the pull image manifest check (with or without proper authentication.)

Thus, the motivation is to allow users to ensure the kubelet requires an image pull auth check for each new set of credentials,
regardless of whether the image is already present on the node.

### Goals

Modify the current behavior of images with an `IfNotPresent` and `Never` `ImagePullPolicy` enforced by the kubelet to
ensure the images pulled with a secret by the kubelet are authenticated by the CRI implementation. During the
EnsureImagesExist step the kubelet will require authentication of present images pulled with auth since boot.

Optimize to only force re-authentication for a pod container image when the
`ImagePullSecrets` used to pull the container image has not already been authenticated.
IOW if an image is pulled with authentication for a first pod, subsequent pods that have the same
authentication information should not need to re-authenticate.

Images already present at boot or loaded externally to the kubelet will not require
authentication unless configured otherwise.
Images successfully pulled through the kubelet with no `ImagePullSecrets`/authentication required will
not require authentication.

The new behavior is designed in a way so that it replaces current behavior - it is going
to be an on-by-default feature once graduated.

### Non-Goals

Out of scope for this KEP is an image caching policy that would direct container
runtimes through the CRI wrt. how they should treat the caching of images on a
node. Such as store for public use but only if encrypted. Or Store for private
use un-encrypted...

This feature will not change the behavior of pod with `ImagePullPolicy` `Always`.

Enforcing periodical repull might be important in order to, for example, be able
to recheck image license entitlement, but is a non-goal for this enhancement.

## Proposal

The kubelet will track container images and the list of authentication information
that lead to their successful pulls. This data will be persisted across reboots
of the host and restarts of the kubelet.

The persistent authentication data storage will be done using files kept in the
kubelet directory on the node. The content of these files will be structured and
versioned using standard config file API versioning techniques.

The kubelet will ensure any image requiring credential verification is always pulled if authentication information
from an image pull is not yet present, thus enforcing authentication / re-authentication.

A new kubelet configuration option will be added, as well as a feature
gate to gate its use:
- `noPullImageSecretRecheck`
    - A boolean that toggles the new behavior. If `true`, the kubelet will fallback to the
      old behavior: only pull an image if it's not present.
    - Defaults to `false`.

### User Stories

#### Story 1

User with multiple tenants will be able to support all image pull policies without
concern that one tenant will gain access to an image that they don't have rights to.

#### Story 2

User will no longer have to inject the `PullAlways` imagePullPolicy to
ensure all tenants have rights to the images that are already present on a host.

### Notes/Constraints/Caveats (Optional)

The new behavior might put registry availability in the critical path for cases
where an image was pulled with credentials, image pull policy is `IfNotPresent`
and a new pod with new set of credentials requests an image that is known to
require authentication.

### Risks and Mitigations

- Credentials used to pull an image from a registry may expire.
  - This enhancement considers credentials that already pulled an image always
    valid in order to use the said image. If credential expiration is a worry,
    users should still use the `Always` image pull policy. Further improving
    the new behavior should be a subject to future KEPs.

- Images can be "pre-loaded", or pulled behind the kubelet's back before it starts.
  In this case, the kubelet is not managing the credentials for these images.
  - To mitigate, metadata will be persisted across reboot. The kubelet will compare previously
    cached credentials against the images that exist. On a new image pull, the kubelet will use
    its saved cache and revalidate as necessary.
    In other words: even if the images are already cached, if new images are present that have not
    previously been authenticated against a pods credentials, then the image will be revalidated.
  - **TODO**: consider using an optional list of no-auth images in kubelet configuration


## Design Details

### Image Pulling Scenarios

The following scenarios are being considered based on how images get pulled:
1. preloaded and never pulled by the kubelet
1. pulled by the kubelet without auth
1. pulled by the kubelet with node-level credentials
1. images pulled by the kubelet with pod/serviceaccount-level credentials, then used again by the same pod / service account / credentials

Today, neither of the above scenarios would cause an image (or image manifest, in case the image already exists) repull in case the `ImagePullPolicy`
is set to either `IfNotPresent` or `Never`.

The goal of this enhancement is to allow multiple tenants to live on the same node without
needing to worry about tenant A's image being reused by tenant B, but at the same time to avoid unnecessary
pulls so that registries' availability does not become blocking.

The goal implies that for scenarios 1-3, the proposed solution must not put registries' availability in the critical path.

For scenario 4., the kubelet must remember credentials that were used to successfully pull an image. It must then require
a repull if there's an attempt to fetch the same image with different, currently unknown credentials. In case the
credentials that come with an image pull request were successfully used before, a repull at the registry must not be issued.

Note that using the tag `:latest` is equivalent to using the image pull policy `Always.`

### Kubelet Caching

The kubelet will track, in memory, a pulled image auth cache for the credentials that were successfully used to pull an image.
This cache will be persisted to disk using a well-defined API (**TODO**) to keep track of the authenticated images between
kubelet restarts. **Any image that already exists on the kubelet's node and is not tracked
is considered authless.**
**TODO**: consider an optional list of no-auth images in kubelet config where the remaining images would still need authn.
**TODO**: consider how to keep the cache from growing unbounded. Perhaps on image GC.

The max size of the cache will scale with the number of unique cache entries * the number of unique images that have not been garbage collected.
It is not expected that this will be a significant number of bytes. Will be verified by actual use in Alpha and subsequent metrics in Beta.

See `/var/lib/kubelet/image_manager_state` in [kubernetes/kubernetes#114847](https://github.com/kubernetes/kubernetes/pull/114847)

```
> {
>   "images": {
>     "sha256:eb6cbbefef909d52f4b2b29f8972bbb6d86fc9dba6528e65aad4f119ce469f7a": {
>       "authHash": { ** per review comment use SHA256 here vs hash **
>         "115b8808c3e7f073": {
>           "ensured": true,
>           "dueDate": "2023-05-30T05:26:53.76740982+08:00"
>         }
>       },
>       "name": "daocloud.io/daocloud/dce-registry-tool:3.0.8"
>     }
>   }
> }
```
**TODO**: The above is the original API proposal which will undergo further considerations
that define at which level we want to cache a credential:
- is it the end dockercfg credential - allows sharing the same credential across multiple secrets
- is it the kubernetes Secret (ns/name/uid) containing the pull credentials - prevents unnecessary live pulls in case of credential rotation
- is it all of the above
**TODO**: Describe when a record about image being pulled is written - is it before or after a pull? Is it both? E.g. we can't know if an image would get pulled successfully beforehand, we don't know the image's digest, ...

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes
necessary to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

For alpha, exhaustive Kubelet unit tests will be provided. Functions affected by the feature gate will be run with the feature gate on and with the feature gate off. Unit buckets will be provided for:

- HashAuth - (new, small) returns a hash code for a CRI pull image auth [link](https://github.com/kubernetes/kubernetes/pull/94899/files#diff-ca08601dfd2fdf846f066d0338dc332beddd5602ab3a71b8fac95b419842da63R704-R751) ** per review comment will use SHA256 **
- shouldPullImage - (modified, large sized change) determines if image should be pulled based on presence, and image pull policy, and now with the feature gate on if the image has been pulled/ensured by a secret. A unit test bucket did not exist for this function. The unit bucket will cover a matrix for:

```golang
  pullIfNotPresent := &v1.Container{
    ..
     ImagePullPolicy: v1.PullIfNotPresent,
   }
   pullNever := &v1.Container{
    ..
    ImagePullPolicy: v1.PullNever,
   }
   pullAlways := &v1.Container{
     ..
    ImagePullPolicy: v1.PullAlways,
   }
   tests := []struct {
     description       string
     container         *v1.Container
     imagePresent      bool
     pulledBySecret    bool
     ensuredBySecret   bool
     expectedWithFGOff bool
     expectedWithFGOn  bool
   }
```

[TestShouldPullImage link](https://github.com/kubernetes/kubernetes/pull/94899/files#diff-7297f08c72da9bf6479e80c03b45e24ea92ccb11c0031549e51b51f88a91f813R311-R438)

PersistMeta() ** will be persisting SHA256 entries vs hash **

Additionally, for Alpha we will update this readme with an enumeration of the core packages being touched by the PR to implement this enhancement and provide the current unit coverage for those in the form of:

- <package>: <date> - <current test coverage>

The data will be read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

##### Integration tests

At beta we will revisit if integration buckets are warranted for cri-tools/critest, and after gathering feedback.

<!--
Integration tests are contained in k8s.io/kubernetes/test/integration.
Integration tests allow control of the configuration parameters used to start the binaries under test.
This is different from e2e tests which do not allow configuration of parameters.
Doing this allows testing non-default options and multiple different and potentially conflicting command line options.
-->

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

- <test>: <link to test coverage> (TBD)

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->
At beta we will revisit if e2e buckets are warranted for e2e node, and after gathering feedback.

- <test>: <link to test coverage> (TBD)

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag - KubeletEnsureSecretPulledImages
- Initial e2e tests completed and enabled - No additional e2e identified as yet

#### Deprecation

N/A in alpha
TBD subsequent to alpha

### Upgrade / Downgrade Strategy

### Version Skew Strategy

N/A for alpha
TBD subsequent to alpha

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: KubeletEnsureSecretPulledImages
  - Components depending on the feature gate: Kubelet
- [x] Other
  - Describe the mechanism: Disabling is possible via the kubelet configuration field `noPullImageSecretRecheck`
  - Will enabling / disabling the feature require downtime of the control
    plane?
    - No, only a restart of the kubelet
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?
    - Yes, as the kubelet must be restarted.

###### Does enabling the feature change any default behavior?

Yes. The behavior of `IfNotPresent` and `Never` pull policies will change, incurring more image pulls and pod creation failures, respectively.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes.

###### What happens if we reenable the feature if it was previously rolled back?

Images pulled during the period the feature was disabled will not be present in the cache, and thus could incur redundant pulls/container creation failures.
However, the cache may still be present, and thus it will retain information from when it was previously enabled.

###### Are there any tests for feature enablement/disablement?

Yes, tests run both enabled and disabled.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

Rollout can fail if the registry isn't available when the kubelet is starting and attempting to create pods, in a similar way
to how the kubelet will generally be more sensitive to registry downtime.

Rollback should not fail for this feature specifically. The kubelet will no longer use the cache to determine whether credentials were used, and
the behavior of the pull policies will revert to the previous behavior.

###### What specific metrics should inform a rollback?

If the feature gate is enabled, but the kubelet configuration field is not enabled, the kubelet will gather metrics `image_pull_secret_recheck_miss` and
`image_pull_secret_recheck_hit` which will be both be a histogram counting the number of images that had a cache miss (despite the image potentially being present).

This will allow an admin to see how many images would have reauthorization checks done.

A histogram was chosen to allow an admin to compare registry uptime with cache misses, as the main failure scenerio is registry unavailability
could cause pods not to come up, because the kubelet doesn't have credentials cached.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

They can be. The presence of a feature gate and kubelet configuration will make this path safe. Plus, there are no API objects that cause issue

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

When the feature is enabled, the kubelet will emit a metric `image_pull_secret_recheck_miss` and `image_pull_secret_recheck_hit` that will happen when a cache miss happens.
This will happen regardless of whether the feature is enabled in the kubelet via its configuration flag.

To determine if the feature is actually working, they will have to check manually.

A user could check if images pulled with credentials by a first pod, are also pulled with credentials by a second pod that is
using the pull if not present image pull policy.

It also will show up as network events. Though only the manifests will be revalidated against the container image repository,
large contents will not be pulled. Thus one could monitor traffic to the registry.

###### How can someone using this feature know that it is working for their instance?

Can test for an image pull failure event coming from a second pod that does not have credentials to pull the image
where the image is present and the image pull policy is if not present.

- [x] Events
  - Event Reason: "kubelet  Failed to pull image" ... "unexpected status code [manifests ...]: 401 Unauthorized"


###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

TBD

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

TBD

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

TBD needed for Beta

### Dependencies

TBD

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

TBD

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No existing API objects will be unchanged.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

Yes. When enabled, and when container images have been pulled with image pull secrets (credentials), subsequent image
pulls for pods that do not contain the image pull secret that successfully pulled the image will have to authenticate
by trying to pull the image manifests from the registry. The image layers do not have to be re-pulled, just the
manifests for authentication purposes.

However, this registry round-trip will slow down the pod creation process. This slowdown is the expense of the added security of this feature.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Similar to the increased time above, there will also be a CPU/memory/IO cost when the kubelet instructs the CRI implementation to repull the image
redundantly.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

This feature doesn't interact with the API server or etcd.

###### What are other known failure modes?

A registry being unavailable is going to be a common failure mode for this feature. Unfortunately, this is the cost of this feature. The kubelet
needs to go through the authentication process redundantly, and that will mean the cluster will be more sensitive to registry downtime.

###### What steps should be taken if SLOs are not being met to determine the problem?

Reduce the number of cache misses (as seen through the metrics) by ensuring similar credentials are shared among images.

## Implementation History

TBD

## Drawbacks [optional]

Why should this KEP _not_ be implemented. TBD

## Alternatives [optional]

- Make the behavior change enabled by default by changing the feature gate to true by default instead of false by default.
- Discussions went back and forth on whether this should go directly to GA as a fix or alpha as a feature gate. It seems this should be the default security posture for pullIfNotPresent as it is not clear to admins/users that an image pulled by a first pod with authentication can be used by a second pod without authentication. The performance cost should be minimal as only the manifest needs to be re-authenticated. But after further review and discussion with MrunalP we'll go ahead and have a kubelet feature gate with default off for alpha in v1.23.
- Set the flag at some other scope e.g. pod spec (doing it at the pod spec was rejected by SIG-Node).
- For beta/ga we may revisit/replace the in memory hash map in kubelet design, with an extension to the CRI API for having the container runtime
ensure the image instead of kubelet.
- Discussions went back and forth as to whether to persist the cache across reboots. It was decided to do so.
- `Never` could be always allowed to use an image on the node, regardless of its presence on the node. However, this would functionally disable this feature from a security standpoint.

## Infrastructure Needed [optional]

TBD
