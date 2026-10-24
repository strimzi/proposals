# Server-Side Apply support in Strimzi CRDs: list and map merge types

This proposal adds the schema metadata that Kubernetes server-side apply (SSA) needs to merge Strimzi custom resources field by field when users apply them.
It is not about Strimzi using SSA itself, which is covered by proposal 052 and proposal 105.
The change is in the CRD schemas.
List and map fields declare how the API server merges them, using `x-kubernetes-list-type`, `x-kubernetes-list-map-keys` and, where it makes sense, `x-kubernetes-map-type`.
Today no field in any Strimzi CRD declares any of these, so every list defaults to atomic and a whole list is owned by a single field manager.
This proposal audits the list and map fields of all ten Strimzi CRDs.
It proposes a merge type for each field that has a defensible merge key or set semantics, and it says plainly which fields have no such key and cannot be fixed by schema metadata alone.
It also covers the generator changes that produce these schemas and the backwards compatibility questions to answer before anything ships.
The tracking issue is [strimzi-kafka-operator#12452](https://github.com/strimzi/strimzi-kafka-operator/issues/12452).

## Current situation

Server-side apply tracks ownership per field in `.metadata.managedFields`, and how a list is merged is decided by the list type declared in the CRD schema.
A list declared as `listType: map` is merged per item, using the fields named in `listMapKeys` as the item identity, so two managers can each own different items.
A list declared as `listType: set` is merged per unique value.
A list with no declared type is atomic.
One manager owns the whole list, and a second manager that touches any item in the list gets a conflict unless it forces the apply.
The reporter of issue #12452 hits this problem.
One pipeline creates the `KafkaMirrorMaker2` resource with the cluster configuration.
A second pipeline fills in `topicsPattern` on `spec.mirrors`.
The API server rejects the second apply because the `mirrors` list is atomic and the first pipeline owns it.
The same problem applies to any split-ownership setup, including GitOps tools that apply the same resource from more than one source.
I checked the generated CRDs in `helm-charts/helm3/strimzi-kafka-operator/crds/` and the Java sources.
No Strimzi CRD and no generator source declares `x-kubernetes-list-type` or `x-kubernetes-map-type` anywhere.
The only occurrences in the repository are inside a vendored cert-manager CRD used as a system test fixture (`systemtest/src/test/resources/cert-manager/cert-manager.yaml`), which shows what the metadata looks like in practice.
Across the ten CRDs there are 782 array-typed fields and 279 map-typed fields in the generated schemas.
Most of them are not fields that Strimzi models by hand.
They sit inside the `template` subtrees, which embed the Kubernetes PodSpec model.
The fields users actually split ownership over are a much smaller set, and this proposal concentrates on those.
The CRD schemas are generated, not handwritten.
The model classes in the `api` module carry annotations from the `crd-annotations` module.
The `io.strimzi.crdgenerator.CrdGenerator` class in the `crd-generator` module turns them into the YAML files under `packaging/install/cluster-operator/` and `helm-charts/helm3/strimzi-kafka-operator/crds/` when the `api` module builds.
The exec-maven-plugin executions in `api/pom.xml` run the generator.
Any schema change therefore has to be made in the annotations and the generator, not by editing the YAML files.

## Motivation

Split ownership of a single custom resource is a normal setup in larger organisations.
Platform teams provide the skeleton of a resource and application teams fill in their parts.
SSA is the Kubernetes mechanism designed for this, and it only works when the CRD schema tells the API server how fields merge.
Without merge types, users have three bad options: force applies that steal ownership of whole lists, merge everything into one pipeline whether it fits their team structure or not, or avoid SSA for Strimzi resources entirely.
The triage of issue #12452 (19.3.2026) decided to keep the issue and to look at adding the proper SSA annotations to all the CRDs, with a proposal covering how to handle it in the code and the impact on users.
Fixing the schemas also removes a recurring source of conflicts between Strimzi resources and the tooling around them, independent of whether Strimzi itself ever applies these resources with SSA.

## Proposal

### Scope

The proposal covers the list and map fields modelled by Strimzi itself in the ten CRDs: `Kafka`, `KafkaConnect`, `KafkaConnector`, `KafkaTopic`, `KafkaUser`, `KafkaNodePool`, `KafkaBridge`, `KafkaMirrorMaker2`, `KafkaRebalance` and `StrimziPodSet`.
Fields inside `template` subtrees are explicitly out of scope for the first phase, for reasons explained below.
Fields are only given a merge type when the audit below can name the merge key, or can argue that order and duplicates carry no meaning.
Everything else stays atomic, and this proposal says so field by field instead of hiding it.

### Field audit

The table below is the first pass of the audit, based on reading the generated schemas of the ten CRDs and the model classes behind the fields named here.
"Current" is atomic for every list, because no list type is declared anywhere today.

| Field | Proposed | Merge key or reasoning |
|---|---|---|
| `Kafka` `spec.kafka.listeners` | `listType: map` | Key `name`. `GenericKafkaListener.getName()` is required, and its description in `api/src/main/java/io/strimzi/api/kafka/model/kafka/listener/GenericKafkaListener.java` states the name identifies the listener and has to be unique within a Kafka cluster. |
| `KafkaConnect` `spec.plugins` | `listType: map` | Key `name`. Items are `ConnectorPlugin` objects with `name` and `artifacts`; the name is the plugin identity. |
| `KafkaNodePool` `spec.storage.volumes` | `listType: map` | Key `id`. JBOD volume items carry an `id`, which is how the operator and users refer to individual volumes. |
| `status.conditions` in all ten CRDs | `listType: map` | Key `type`. This matches the convention of `metav1.Condition` in the built-in Kubernetes APIs. The benefit is consistency more than split ownership, because the operators are the only writers of status. |
| `KafkaNodePool` `spec.roles` | `listType: set` | A list of strings drawn from `broker` and `controller`. Duplicates and ordering carry no meaning. |
| `Kafka` `spec.maintenanceTimeWindows` | `listType: set` | A list of cron expression strings. Duplicates carry no meaning. |
| `KafkaRebalance` `spec.goals` | `listType: set` | A list of Cruise Control goal names. Duplicates carry no meaning. |
| `KafkaUser` `spec.authorization.acls[].operations` | `listType: set` | A list of operation name strings inside one ACL rule. Duplicates carry no meaning. |
| `KafkaMirrorMaker2` `spec.mirrors` | stays atomic | No valid key exists. A mirror is identified by its source cluster, but that is the nested object `mirrors[].source` (which has the `alias` field), and `listMapKeys` can only name scalar fields of the list item itself. |
| `KafkaUser` `spec.authorization.acls` | stays atomic | No valid key exists. A rule is identified by the combination of the nested `resource` object (`type`, `name`, `patternType`), the `operations` list, `host` and the permission `type`, and neither an object nor a list can be a merge key. |
| `Kafka` `spec.kafka.listeners[].networkPolicyPeers` | stays atomic | Peers are identified by label selector objects, and there is no scalar field to key on. |
| `spec.config` maps (`Kafka` `spec.kafka.config`, `KafkaConnect` and `KafkaConnector` and `KafkaTopic` `spec.config`, the cluster `config` maps in `KafkaMirrorMaker2`) | needs investigation | These are `type: object` with `x-kubernetes-preserve-unknown-fields: true` and no schema for the values. Whether per-key ownership works at all for such free-form maps under SSA has to be tested rather than assumed, and it is not clear `x-kubernetes-map-type: granular` is meaningful without a value schema. |
| Everything under `spec.*.template` | out of scope for phase one | These schemas are generated from the Fabric8 Kubernetes model classes, not from Strimzi model classes, so annotations on Strimzi classes cannot reach them. The same fields in the built-in Pod API do carry merge types, but that metadata does not carry over into the copies inside a CRD schema. |

Two consequences follow from this audit.
First, the field that motivated issue #12452, `KafkaMirrorMaker2` `spec.mirrors`, cannot be fixed by schema annotations in the current API shape.
Fixing it would require an API change, for example promoting a name onto the mirror item itself, and that belongs in a separate discussion.
Second, the highest-value fields for split ownership are the ones that do have keys: listeners, Connect plugins and JBOD volumes are exactly the kind of list where one team provides the list skeleton and another team edits individual entries.

### Generator and model changes

The schemas are produced from annotations, so the implementation has three parts.
Add a new annotation in `crd-annotations/src/main/java/io/strimzi/crdgenerator/annotations/`, next to `MinimumItems`, `Pattern` and `CelValidation`.
The annotation carries the list type and the optional list map keys, with the same per-version scoping (`apiVersions`) that the existing annotations support.
Add emission support in `io.strimzi.crdgenerator.CrdGenerator`.
Array schemas are built in `buildArraySchema`, which already emits `minItems` from `@MinimumItems` (used for example on `KafkaClusterSpec.getListeners()`), and that is the natural place to also emit `x-kubernetes-list-type` and `x-kubernetes-list-map-keys`.
If the config map investigation concludes that a map type is needed and valid, the equivalent emission of `x-kubernetes-map-type` belongs in the object schema construction in the same class.
Annotate the model getters in the `api` module for the fields in the agreed first batch, and nothing else.
Add unit tests in `crd-generator`, next to the existing golden-file tests under `crd-generator/src/test/resources/io/strimzi/crdgenerator/`.
Cover a map list with keys, a set list, and a field left atomic.
Regenerate the CRDs under `packaging/install/cluster-operator/` and `helm-charts/helm3/strimzi-kafka-operator/crds/`, and the CRD reference documentation that `DocGenerator` produces in the same `api` build.
No operator code changes.
The cluster, topic and user operators read the custom resources and write status, and this proposal does not change how they reconcile.

### Rollout

The generator support and the first field batch should land together, across all ten CRDs in one release, rather than CRD by CRD.
The CRDs are installed and upgraded as one set, and the schema additions are metadata, so splitting by CRD buys little and leaves users guessing which resources support SSA and which do not.
Batch instead by field confidence.
A field joins the annotated set only when its audit row names a key or argues set semantics, and the table in this proposal is the review artifact for that decision.
Fields whose investigations are still open, meaning the config maps and the template subtrees, stay atomic in the first release and can be added later without any further generator work.

## Affected/not affected projects

Only the `strimzi-kafka-operator` repository is affected.
Within it, the affected parts are the `api`, `crd-annotations` and `crd-generator` modules, the generated CRDs in `packaging/install` and `helm-charts`, and the documentation.
The operator runtimes (`cluster-operator`, `topic-operator`, `user-operator`) are not affected.
Their reconciliation logic does not change.
Proposal 105, which covers Strimzi itself using SSA for the resources it manages, is independent of this proposal: neither blocks or accelerates the other, and the distinction is the one made in issue #12452 itself.
`StrimziPodSet` is in the list of ten CRDs, but in practice only its `status.conditions` would change in the first batch, because `spec.pods` items are opaque pod objects with `x-kubernetes-preserve-unknown-fields: true` and users do not manage PodSets directly.

## Compatibility

Changing a list type on a published CRD is not free, and the following points have to be worked through before the first batch ships.
Once a list is declared `listType: map` or `listType: set`, the API server enforces unique keys and unique values on writes.
Any existing resource that already contains duplicates, which is plausible in hand-written `roles`, `goals` or `maintenanceTimeWindows` lists, would fail subsequent updates until the duplicates are removed.
The audit therefore has to double as a duplicate risk assessment per field, and the behaviour change has to be called out in the release notes.
Objects created before the change carry `managedFields` entries recorded under atomic semantics.
One manager owns each whole list.
After the schema change, a second manager applying a subset of the list gets per-item ownership, but the first SSA operation after the upgrade can still conflict with the previously recorded atomic owner and may need a one-off forced apply.
This needs an explicit upgrade test: install the current CRDs, create a `Kafka` and a `KafkaMirrorMaker2` using two different field managers and also with client-side apply, upgrade the CRDs, and repeat the applies while checking for conflicts and for silently dropped list items.
`kubectl apply` stores the whole object in the `kubectl.kubernetes.io/last-applied-configuration` annotation.
How that interacts with the new per-item ownership after upgrade is part of the same test, because Strimzi cannot break the most common way users manage these resources today.
The operators are the only writers of `status.conditions` in practice, so moving conditions to `listType: map` keyed by `type` is low risk.
It also aligns the CRDs with the convention users already know from built-in resources.
The generator runs with `--target-kube 1.16+` (see `api/pom.xml`), while SSA itself is GA only since Kubernetes 1.22.
Whether `x-kubernetes-list-type` and `x-kubernetes-list-map-keys` can be emitted for the oldest Kubernetes versions Strimzi supports, or whether emitting them raises the effective floor, has to be checked before rollout rather than discovered by users.
The annotations support scoping by API version.
If a CRD serves more than one version again, the merge types can be applied consistently per version instead of only to the storage version.

## Open questions

The following cannot be answered from the code and are called out as investigations, with what each one has to produce.
The config maps need a runtime test.
Does per-key SSA ownership work for the free-form `spec.config` maps with `x-kubernetes-preserve-unknown-fields: true`, and is `x-kubernetes-map-type: granular` valid and useful there?
The investigation is a small test on a local cluster applying one config map from two managers, and its outcome decides whether config joins a later batch or stays as it is.
MirrorMaker 2 mirrors need a project decision.
Is the project willing to consider a future API change that gives a mirror a top-level identity field, or should `spec.mirrors` stay atomic permanently?
Without one of those, the original reporter's scenario keeps needing a workaround, and the documentation should say so honestly.
The template subtrees need a generator design decision.
Should the generator learn path-based merge type overrides for well-known PodSpec fields, such as `containers` keyed by `name` and `volumes` keyed by `name`, or are templates permanently out of scope?
This is a generator design question, and the answer decides the size of any second phase.
Key uniqueness needs a validation decision.
For the map-keyed fields, should the schemas also gain CEL validation rules for key uniqueness, using the existing `@CelValidation` support, or is API server enforcement of the list type enough on its own?
The oldest supported Kubernetes version needs a maintainer answer.
What is the oldest version Strimzi commits to for the next release, and does it accept list type metadata in CRD schemas?

## Implementation plan

1. Turn the field audit in this proposal into a complete inventory.
Classify every list and map field outside the `template` subtrees in all ten CRDs as map, set, atomic or needs-investigation, and name the key for every map candidate.
2. Run the two investigations from the open questions that gate the design: SSA behaviour on the free-form config maps, and the upgrade sequence with old CRDs, two field managers and client-side apply.
3. Add the new annotation to `crd-annotations` and the emission support to `CrdGenerator`, with unit tests in `crd-generator`.
4. Annotate the agreed first batch of fields in the `api` module, regenerate the CRDs and the CRD reference documentation, and review the generated diff field by field against the audit table.
5. Add a section to the user documentation explaining which fields support split ownership through SSA, which stay atomic and why, including the `KafkaMirrorMaker2` `spec.mirrors` case from issue #12452.
6. Ship it in a single release across all CRDs, with the uniqueness behaviour change called out in the release notes.

## Rejected alternatives

Annotate only `KafkaMirrorMaker2` `spec.mirrors`, as the title of issue #12452 suggests.
Rejected, because `mirrors` has no valid merge key in the current API shape, and a one-field change would still leave listeners, plugins, volumes and the other keyed lists atomic.
Restructure the APIs to add an explicit key field to every list item.
Rejected for now, because it is a breaking API design change across all CRDs, and it should be its own proposal if the project ever wants to fix `mirrors` and `acls` properly.
Use the single-source-of-truth GitOps workaround suggested in the issue discussion, where one pipeline owns the whole resource and teams coordinate through pull requests.
It is a reasonable working practice, but it does not fix the schema, and the triage decision was to keep the issue and address the annotations.
Post-process the generated CRD YAML files in `packaging` instead of teaching the generator.
Rejected, because the YAML files are regenerated on every `api` build and the copies in `packaging/install` and `helm-charts` would drift away from the generator output.
Leave the CRDs as they are and document that SSA is not supported for split ownership.
Rejected, because the fields with natural keys are a small, well understood set, and users will keep hitting avoidable conflicts on them.
