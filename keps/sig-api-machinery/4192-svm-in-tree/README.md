# KEP-4192: Move Storage Version Migrator in-tree

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
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
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
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
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary
Kubernetes relies on resource data being actively re-written to perform certain maintenance actives related to at rest storage.  Two prominent examples are the versioned schema of stored resources (i.e. the preferred storage schema changing from `v1beta1` to `v1` for a given resource) and encryption at rest (i.e. rewriting `stale` data based on a change in how the data should be encrypted).

The simplest way to rewrite data is to issue no-op `update` requests via `kubetctl get <resource> | kubectl replace -`.  This is problematic for any resource that can contain a large amount of data such as Kubernetes `secrets` and it is also impractical to perform without automation as the number of resources that need migration is always growing.

Unlike most Kuberentes API interactions, during storage migration, it is safe to ignore conflicts during `update` requests and it is safe to use inconsistent continue tokens during paginated `list` operations (since we only need data to be re-written, we do not care how it is re-written).

This KEP aims to make it easy for users to perform storage migrations without having to worry about the details.

## Motivation

### Goals

- Make it easy for an admin to trigger a storage migration without having to manually run `kubectl get | kubectl replace`
- Enable infrastructure providers to use internal details of their enviorment to trigger storage migration (ex: triggering storage migration of Kubernetes secrets when the KMS v2 key ID has changed)
- Make it easy for existing users of SVM to migrate to the in-tree controller

### Non-Goals

- Any automatic or direct integration with KMS v2
- Any modification regarding `StorageVersion` API for HA API servers
- Adding logic that relies on the hashed storage versions exposed via the discovery API

### UNCLEAR Goals and/or Non-Goals

- Automatic storage version migration for CRDs
- Make it easy for Kuberentes developers to drop old API schemas by guaranteeing that storage migration is run automatically on SV hash changes (should this also be on a timer or API server identity?)
- Automated Storage Version Migration via the hash exposed by the `StorageVersion` API

## Proposal

- Move the existing SVM controller logic in-tree into KCM
- Move the existing SVM REST APIs in-tree (possibly under a new API group to avoid conflicts with the old API being run concurrently)
- **[UNCLEAR]** Create a new controller that watches the storage version API and automatically triggers migrations based on changes (to the SV API)

### User Stories (Optional)

#### Story 1 **[UNCLEAR]**
As an end user of Kubernetes, we get automatic storage version migration whenever the storage version changes due to an api server upgrade/downgrade.

#### Story 2
As an end user using encryption at rest, whenever the key change is detected we can run the storage migration to use the new key for encryption. 

### Notes/Constraints/Caveats (Optional)

### Risks and Mitigations

## Design Details

### APIs to move
#### We will move following [APIs](https://github.com/kubernetes-sigs/kube-storage-version-migrator/blob/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/apis/migration/v1alpha1/types.go) in-tree:
- `v1alpha1` of `storageversionmigrations.migration.k8s.io`  

    ```go
    // StorageVersionMigration represents a migration of stored data to the latest
    // storage version.
    type StorageVersionMigration struct {
        metav1.TypeMeta `json:",inline"`
        // +optional
        metav1.ObjectMeta `json:"metadata,omitempty"`
        // Specification of the migration.
        // +optional
        Spec StorageVersionMigrationSpec `json:"spec,omitempty"`
        // Status of the migration.
        // +optional
        Status StorageVersionMigrationStatus `json:"status,omitempty"`
    }

    // Spec of the storage version migration.
    type StorageVersionMigrationSpec struct {
        // The resource that is being migrated. The migrator sends requests to
        // the endpoint serving the resource.
        // Immutable.
        Resource GroupVersionResource `json:"resource"`
        // The token used in the list options to get the next chunk of objects
        // to migrate. When the .status.conditions indicates the migration is
        // "Running", users can use this token to check the progress of the
        // migration.
        // +optional
        ContinueToken string `json:"continueToken,omitempty"`
        // The storage version hash to avoid races.
        StorageVersionHash string `json:"storageVersionHash,omitempty"`
    }

    // The names of the group, the version, and the resource.
    type GroupVersionResource struct {
        // The name of the group.
        Group string `json:"group,omitempty"`
        // The name of the version.
        Version string `json:"version,omitempty"`
        // The name of the resource.
        Resource string `json:"resource,omitempty"`
    }
    
    // Status of the storage version migration.
    type StorageVersionMigrationStatus struct {
        // The latest available observations of the migration's current state.
        // +optional
        // +patchMergeKey=type
        // +patchStrategy=merge
        Conditions []MigrationCondition `json:"conditions,omitempty"`
    }

    // Describes the state of a migration at a certain point.
    type MigrationCondition struct {
        // Type of the condition.
        Type MigrationConditionType `json:"type"`
        // Status of the condition, one of True, False, Unknown.
        Status corev1.ConditionStatus `json:"status"`
        // The last time this condition was updated.
        // +optional
        LastUpdateTime metav1.Time `json:"lastUpdateTime,omitempty"`
        // The reason for the condition's last transition.
        // +optional
        Reason string `json:"reason,omitempty"`
        // A human readable message indicating details about the transition.
        // +optional
        Message string `json:"message,omitempty"`
    }

    type MigrationConditionType string

    const (
        // Indicates that the migration is running.
        MigrationRunning MigrationConditionType = "Running"
        // Indicates that the migration has completed successfully.
        MigrationSucceeded MigrationConditionType = "Succeeded"
        // Indicates that the migration has failed.
        MigrationFailed MigrationConditionType = "Failed"
    )

    // StorageVersionMigrationList is a collection of storage version migrations.
    type StorageVersionMigrationList struct {
        metav1.TypeMeta `json:",inline"`
        // +optional
        metav1.ListMeta `json:"metadata,omitempty"`
        // Items is the list of StorageVersionMigration
        Items []StorageVersionMigration `json:"items"`
    }
    ```

- APIs in-tree will be _converted to `built-in types`_ from CRD.

#### [Undecided] Changes while we move above APIs in-tree:
To avoid any conflicts with the Storage Version Migrators running out of tree, we will change the _`group`_ from `migration.k8s.io` to `storagemigration.k8s.io`.

The final APIs that will be moved in-tree are:
- `v1alpha1` of `storageversionmigrations.storagemigration.k8s.io`

### [Controller](https://github.com/kubernetes-sigs/kube-storage-version-migrator/tree/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/controller) to move
#### [Migrator Controller](https://github.com/kubernetes-sigs/kube-storage-version-migrator/tree/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/migrator)
Currently, the Storage Version Migrator comprises two controllers: the `Trigger` controller and the `Migrator` controller. The Trigger controller performs resource discovery, identifying supported resources with the preferred server version every `10 minutes`. Subsequently, the Trigger controller creates the `StorageVersionMigration` resource to initiate the migration process. The Migrator controller then picks up this resource and executes the actual migration.

When transitioning the Storage Version Migrator in-tree, we will exclusively move the Migrator controller as a component of KCM. The creation of the Migration resource will be deferred to the user, instead of being triggered automatically.

#### Streaming List
Currently, the Migrator controller uses the `chunked List` method to retrieve the list of resources from the API server and subsequently perform storage migrations as needed. However, chunked lists are [resource-intensive]((https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/3157-watch-list/README.md#motivation)) and can lead to a significant overload on the API server, potentially resulting in it being terminated due to out-of-memory (OOM) issues. To address this concern, we have proposed the adoption of an Alpha feature introduced in Kubernetes _v1.27_, known as [`Streaming List`](https://kubernetes.io/docs/reference/using-api/api-concepts/#streaming-lists).

When the Migrator controller is integrated in-tree, it will leverage the `Streaming List` approach to obtain and monitor resources while executing storage version migrations, as opposed to using the resource-intensive `chunked list` method. In cases where, for any reason, the client cannot establish a streaming watch connection with the API server, it will fall back to the standard `chunked list` method, retaining the older LIST/WATCH semantics.
    
- _Open Question_:
    - Does `Streaming List` support inconsistent lists? Currently, with chunked lists, we receive inconsistent lists. We need to determine if we can achieve the same with Streaming List. Depending on the outcome, we may consider removing the [ContinueToken](https://github.com/kubernetes-sigs/kube-storage-version-migrator/blob/60dee538334c2366994c2323c0db5db8ab4d2838/pkg/apis/migration/v1alpha1/types.go#L63) from the API.

### RBAC for SVM
- Storage Version Migrator Controller
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: system:controller:storage-version-migrator-controller
    rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: 
      - get 
      - list
      - watch
      - update
      
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: system:controller:storage-version-migrator-controller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:controller:storage-version-migrator-controller
    subjects:
    - kind: ServiceAccount
      name: storage-version-migrator-controller
      namespace: kube-system
    ```

### Test Plan

- [x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

- `<package>`: `<date>` - `<test coverage>`

##### Integration tests

- [ ] Write a test which will identify the migratable reousrce, create Migration resource for the same and test whether Sotrage Version Mogration is actually carried out.

##### e2e tests

- [ ] Test to validate funtionality of the Migration controller.

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Feature is enabled by default
- All of the above documented tests are complete

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: _`StorageVersionMigrator`_
  - Components depending on the feature gate:
      - Kube Controller Manager (KCM)
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?

###### Does enabling the feature change any default behavior?
No

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes

###### What happens if we reenable the feature if it was previously rolled back?

It should start performing Storage Migration for the given migration request.

###### Are there any tests for feature enablement/disablement?
We will add integration tests to validate the enablement/disablement flow.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?
This may impact workloads, as issuing too many Storage Version Migration requests can increase the latency on the API server.

###### What specific metrics should inform a rollback?
The metric `storage_migrator_core_migrator_migrations` with `failed` status can indicate how many failures there are and can help decide if a rollback is required.

The following metrics should not be available if we perform a rollback:
- storage_migrator_core_migrator_migrated_objects
- storage_migrator_core_migrator_remaining_objects
- storage_migrator_core_migrator_migrations

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?
This will be covered by integration tests.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?
No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?
The following metrics are available when Storage Version Migration is enabled:
- storage_migrator_core_migrator_migrated_objects
- storage_migrator_core_migrator_remaining_objects
- storage_migrator_core_migrator_migrations

###### How can someone using this feature know that it is working for their instance?
- [ ] Events
  - Event Reason: 
- [ ] API .status
  - Condition name: **_MigrationCondition.Type (Running | Succeeded | Failed)_**
  - Other field: **_MigrationCondition.LastUpdateTime_**
- [x] Other (treat as last resort)
  - Details:
      -    The following metrics are available when Storage Version Migration is enabled:
            - storage_migrator_core_migrator_migrated_objects
            - storage_migrator_core_migrator_remaining_objects
            - storage_migrator_core_migrator_migrations

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?
NA

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?
- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [x] Other (treat as last resort)
  - Details:
      - The metric `storage_migrator_core_migrator_migrations` with the status `Failed` can be observed to determine the health of the migrations.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

###### Will enabling / using this feature result in any new API calls?

Yes. Creation of the Migration Request which creates `SotrageVesionMigration` resource. 

###### Will enabling / using this feature result in introducing new API types?

Yes, the Storage Veersion Migration API.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Yes. In Kube Controller Manager (KCM) and API server.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

Maybe (Not exactly sure how does this affect)

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

This feature realy on the API server and etcd availability. If they are not available then it will results in errors.

###### What are other known failure modes?

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

This feature is already implemented in [out-of-tree](https://github.com/kubernetes-sigs/kube-storage-version-migrator). We are moving this in-tree.

## Drawbacks

NA

## Alternatives

NA

## Infrastructure Needed (Optional)

NA