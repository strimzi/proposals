# Maven mirror support for Kafka Connect builds

## Objective

Add first-class support for configuring one or more Maven **mirror(s)** for the Kafka Connect build.
When set, *all* Maven resolution performed during the build - connector artifacts, their transitive dependencies, and the Maven build plugins themselves (notably `maven-dependency-plugin`, used by `dependency:copy-dependencies`) - is routed through the configured internal repository (a corporate Nexus/Artifactory, a proxy, or an air-gapped mirror).
The mirror is an explicit, optional, opt-in build setting.
When it is not set, the generated build is identical to today.

## Current situation

Strimzi can build Kafka Connect images with additional connector plugins declared in `KafkaConnect.spec.build`.
For artifacts of type `maven`, the operator generates a multi-stage `Dockerfile` (see `io.strimzi.operator.cluster.model.KafkaConnectDockerfile`, method `connectorPluginsPreStage`) that runs inside the `maven-builder` image (`quay.io/strimzi/maven-builder:latest` by default).
For each Maven artifact the operator:

1. Downloads the artifact `pom.xml` with `curl`.
2. Writes a small Maven `settings.xml` file to `/tmp`.
3. Runs `mvn dependency:copy-dependencies -s <settings.xml> ...` to fetch the artifact's transitive dependencies.
4. Downloads the artifact `.jar` with `curl`.

When a user sets `MavenArtifact.repository`, that URL is injected into the generated `settings.xml` as an ordinary `<repository>` inside an active profile.
If no repository is set, it defaults to Maven Central (`MavenArtifact.DEFAULT_REPOSITORY`, `https://repo1.maven.org/maven2/`).
The custom repository is added **only** to `<repositories>`:

```xml
<settings xmlns="https://maven.apache.org/SETTINGS/1.0.0">
  <profiles>
    <profile>
      <id>download</id>
      <repositories>
        <repository>
          <id>custom-repo</id>
          <url>${customRepositoryUrl}</url>
        </repository>
      </repositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>download</activeProfile>
  </activeProfiles>
</settings>
```

Two properties of this design matter for this proposal.

- **The `repository` field is, by design, the source of the connector artifact.**
  It is not intended to be - and normally is not - a general Maven repository that hosts Maven *plugins*.
  Conflating "where the connector JAR lives" with "where build tooling comes from" is a category error.
- **There is no way to keep the build off Maven Central.**
  Because `<pluginRepositories>` is never emitted, Maven resolves `maven-dependency-plugin` (and its dependencies) from the implicit `central` plugin repository defined in Maven's built-in super POM.
  Even a user who has pointed every artifact at an internal repository still has a hidden, unavoidable dependency on `repo1.maven.org` for the build machinery.

## Motivation

In enterprise, air-gapped, or heavily mirrored environments, that hidden dependency on Maven Central is a frequent and hard-to-diagnose cause of Connect build failures:

- **HTTP 429 (Too Many Requests):** Maven Central increasingly rate-limits anonymous traffic.
  Shared egress IPs (corporate NAT/proxy) make this common, and the plugin resolution step fails even though every declared artifact is served locally.
- **Network restrictions / air-gap:** Clusters with no route to `repo1.maven.org` cannot resolve the plugins at all, so the build never completes despite a fully populated internal mirror.
- **Connection timeouts:** Locked-down egress and slow proxies cause the plugin download to time out.

The failure is confusing to operators because the artifacts they explicitly configured resolve fine - only the build machinery silently depends on Central.
This was raised in [strimzi-kafka-operator#8087](https://github.com/strimzi/strimzi-kafka-operator/issues/8087).

The idiomatic Maven mechanism for "route everything through our internal repository" is a **mirror**, not an extra plugin repository.
A mirror intercepts requests to the repositories it matches - including the implicit Central plugin repository - and redirects them to a single URL.
This removes Maven Central from the resolution path entirely (rather than merely adding an alternative source and *hoping* the plugin happens to be present), which is what air-gapped and rate-limited environments actually require.
Giving users a simple, declarative way to configure such a mirror is the clean, complete solution to this class of build failures.

## Proposal

Add an optional list of Maven mirrors to the `maven` artifact type in `KafkaConnect.spec.build`.
When set, the operator writes each mirror into the `settings.xml` it already generates for the build and passes it to Maven via the existing `mvn -s <settings.xml>` invocation.
Because a Maven mirror intercepts *every* repository request - including the super POM's implicit `central` used to resolve `maven-dependency-plugin` - this removes Maven Central from the build's resolution path entirely.

Builds exist only in `KafkaConnect`, so this is the only resource affected.

### API

Extend the `io.strimzi.api.kafka.model.connect.build.MavenArtifact` model with a new optional `mirrors` field: a list of mirror objects.
Each entry is an object (not a bare string) so that further per-mirror settings can be added later without a breaking change (see *Future work*).

- `mirrors` (optional): list of `MavenMirror` objects.
- `MavenMirror` - fields:
  - `url` (required): base URL of the mirror.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
spec:
  build:
    output:
      type: docker
      image: my-registry.io/my-org/my-connect:latest
    plugins:
      - name: my-connector
        artifacts:
          - type: maven
            group: com.example
            artifact: my-connector
            version: 1.0.0
            mirrors:
              - url: https://nexus.internal.example.com/repository/maven-public/
```

The `mirrors` list is set on the `maven` artifact type, alongside the existing `repository` field, because mirroring is a Maven-specific concern.
It does not replace `repository`, so the two can be combined (see *Interaction with the existing connector `repository` field*).

`url` is a required field on each `MavenMirror`, so an entry that omits it is rejected by structural CRD validation at admission time.
The URL value itself is not further validated, consistent with the existing `repository` field, which is not validated either.

### Generated `settings.xml`

For the example above, the operator produces the following (the operator assigns each mirror a generated, stable `id`):

```xml
<settings xmlns="https://maven.apache.org/SETTINGS/1.0.0">
  <mirrors>
    <mirror>
      <id>strimzi-mirror-0</id>
      <url>https://nexus.internal.example.com/repository/maven-public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
</settings>
```

Each configured entry produces one `<mirror>` with `<mirrorOf>*</mirrorOf>`.
When `repository` is also set on the artifact, its `<repository>` continues to be emitted in the download profile exactly as today, in addition to the `<mirrors>` block.
When `repository` is left unset — as in the example above — the operator no longer falls back to emitting the default Maven Central `<repository>`; only the `<mirrors>` block is written, since a `mirrorOf: "*"` mirror already serves Central.

### Resolution semantics

- With a mirror in place (`mirrorOf: "*"`), Maven redirects **every** repository request - release and snapshot, artifact and *plugin* repositories, including the super POM's implicit `central` - to the mirror URL.
  `maven-dependency-plugin` and its dependencies are therefore resolved from the internal repository, and Maven Central is never contacted.
  This is what makes the mirror a complete fix for both the rate-limit (429) and true air-gap cases.
- `mirrorOf` is fixed to `*` and is not exposed in the API in this proposal.
  This is the setting that air-gapped and rate-limited environments need, and it keeps raw Maven `mirrorOf` expressions out of the Strimzi API.
  Nothing is lost by fixing it to `*`: the narrower `external:*` variant exists only to leave `file://` and `localhost` repositories unmirrored, and a containerised Connect build resolves solely from remote HTTP(S) repositories, so no such repositories exist in this context.
- The mirrors live in the **user** settings file passed with `mvn -s`.
  They affect only the Maven invocation Strimzi runs during the build; they are not global mirrors and have no effect on any other Maven usage.
- There is intentionally **no fallback** to Maven Central when the mirror does not serve a required dependency.
  With `mirrorOf: "*"` a missing artifact makes the build fail fast with a standard Maven resolution error naming the artifact, rather than silently reaching out to Central.
  This is the whole point in air-gapped and rate-limited environments: a fallback would reintroduce exactly the dependency on Central the mirror exists to remove.
  Keeping the mirror fully populated - typically a Nexus/Artifactory proxy repository that lazily caches from upstream - is therefore an operational prerequisite.
- With `mirrorOf: "*"`, a build has at most one *effective* mirror.
  Maven applies the first mirror whose `mirrorOf` matches a given repository and ignores the rest, so listing several entries today does not create several parallel mirrors.
  The list is provided for forward compatibility; the typical configuration is a single entry.

### Interaction with the existing connector `repository` field

The per-artifact `MavenArtifact.repository` field is unchanged and keeps its meaning: the source of the connector artifact.
Mirrors and repositories can be configured together on the same artifact - for example a snapshot repository holding a freshly built connector, plus a mirror serving all the other dependencies.
Both are emitted into the same `settings.xml`: the repository as a `<repository>` in the download profile, the mirror as a `<mirror>`.

With the mirror's fixed `mirrorOf: "*"`, standard Maven behaviour means the mirror also intercepts requests to that per-artifact repository, so both the connector and its dependencies are fetched through the mirror.

### Mirror authentication

Mirror authentication is out of scope for this proposal.
Authenticated repositories are an existing gap that applies equally to the current `repository` field, and should be addressed uniformly for both rather than only for mirrors - modelling Maven's many authentication mechanisms in the Strimzi API is a larger, separate concern.
When the mirror requires credentials, users can front it with a network-level mechanism or an anonymous read-only mirror; first-class credential support is deferred to *Future work*, where it can be added as an optional field on `MavenMirror`.

### Delivery

- `api`: new optional `mirrors` field on the `MavenArtifact` model and a new `MavenMirror` type with a required `url`; CRD regeneration (`make crd_install`) and API reference documentation.
- `cluster-operator`: `KafkaConnectDockerfile` emits the `<mirrors>` block into the generated `settings.xml`; unit coverage in `KafkaConnectDockerfileTest` for no mirror (output unchanged), a single mirror, and multiple mirror entries.
- User documentation describing when and how to configure a build mirror.

## Affected/not affected projects

**Affected:**

- `strimzi-kafka-operator`:
  - `api` - new optional `mirrors` field on the `MavenArtifact` model and a new `MavenMirror` type; CRD change (additive) and API docs.
  - `cluster-operator` - `KafkaConnectDockerfile` settings generation.

**Not affected:**

- The `maven-builder` container image (no baked-in configuration; behaviour is driven purely by the generated `settings.xml`).
- `KafkaConnector` and `KafkaMirrorMaker2`, which do not run builds (the `Build` model is used only by `KafkaConnect`).
- Topic Operator, User Operator, and all other operands.
- Artifact types other than `maven` (e.g. `jar`, `tgz`, `zip`, `other`), which do not invoke Maven - though a `mirrorOf: "*"` mirror does not interfere with them.
- Existing `KafkaConnect` resources that do not set `mirrors` on a `maven` artifact (see *Compatibility*).

## Compatibility

- **Additive, opt-in API.** `mirrors` on a `maven` artifact is a new optional field.
  When it is absent, the generated `Dockerfile` and `settings.xml` are identical to today, so all existing builds - including those already using `MavenArtifact.repository` - are unaffected.
  This yields full backwards compatibility with no action required by existing users.
- **No behavioural change to the connector `repository` field** unless a mirror is explicitly configured.
  Even then, the change follows standard, documented Maven mirror semantics that the user has opted into.
- **CRD versioning.** The change is a backwards-compatible field addition to the existing API version; no breaking CRD version bump is required.

## Security considerations

Routing build traffic through a mirror changes *where build-time code (Maven plugins) originates*, which has supply-chain implications:

- **Explicit, user-declared trust.** A mirror only takes effect when a user sets `mirrors` on a `maven` artifact and names the mirror URL.
  Nothing changes for users who do not configure it, and the default (no mirror) preserves today's trust anchor (Maven Central over TLS).
- **Trusted internal mirrors only.** Because `mirrorOf: "*"` redirects *all* resolution - including executable build plugins - administrators must point the mirror at a trusted, controlled internal repository (e.g. a vetted Nexus/Artifactory).
  This is precisely the environment in which the problem arises, so posture and use case align.
- **No weakening of existing controls.** The change does not alter the existing per-artifact `insecure` (TLS-skip) handling.
  TLS verification against the mirror follows normal Maven behaviour unless the user separately opts into insecure mode.
- **Authentication deferred, not weakened.** Because mirror authentication is out of scope, this proposal introduces no new credential handling.
  When credential support is added later (for both `repository` and mirror), it should follow the existing Strimzi `Secret`-reference pattern and be exposed only to the build step, taking care not to leak credentials into image layers or logs.

## Future work

Possible enhancements that can be added through future proposals if needed:

- **Configurable `mirrorOf`.** An optional `mirrorOf` field on `MavenMirror`, defaulting to `*`, would let users leave selected repositories direct (e.g. `*,!custom-repo` to keep a snapshot repository unmirrored) or mirror only Central.
- **Mirror (and repository) authentication.** First-class support for authenticated mirrors and repositories, via `Secret` references, handled uniformly for both, added as optional fields on `MavenMirror`.
- **Proxy configuration.** Some environments require an HTTP(S) proxy rather than (or in addition to) a mirror; a proxy configuration could be added following the same opt-in pattern.

## Rejected alternatives

- **A single `mirror` boolean flag (or a single `mirror: <url>`) on the `maven` artifact.**
  This was an earlier shape of this proposal, rejected during review.
  A boolean flag (or single URL) cannot grow to express multiple mirrors or per-mirror settings, so it would very likely have to be deprecated and replaced later - the exact churn the `List<MavenMirror>` shape avoids.

- **A `List<String>` of mirror URLs.**
  Rejected in favour of a list of objects.
  A list of plain strings future-proofs the *number* of mirrors but not their *shape*: adding a configurable `mirrorOf` or authentication later would still require introducing a parallel object-based field and deprecating the string list.
  `List<MavenMirror>` future-proofs both.

- **A dedicated build-scoped section carrying the full Maven mirror model up front (per-mirror `mirrorOf` and authentication).**
  Rejected as the initial scope.
  Propagating raw `mirrorOf` expressions and per-mirror credential references into the Strimzi API from day one adds significant surface and maintenance for capabilities most users do not need, and pulls Strimzi into modelling Maven authentication (which has many mechanisms) prematurely.
  The chosen `mirrors: [ { url } ]` shape keeps the initial API minimal while leaving room to add exactly those fields later if demand appears.

- **Inject the connector repository as a `<pluginRepository>` (behind an operator env var).**
  Rejected because:
  - It **conflates** the connector artifact repository with a plugin repository; that field is not intended to host Maven plugins.
  - It is **additive, not authoritative** - it does not remove Central from the plugin resolution path.
    If the configured repository does not actually mirror `maven-dependency-plugin`, Maven falls back to Central and the build can *still* hit the 429 / air-gap failure.
    A `<mirror>` intercepts the request instead of merely offering an alternative, which is why a mirror is authoritative where a plugin repository is not.
  - Enabling it unconditionally would change behaviour for everyone (proxies returning `401`/`403` rather than `404` break the fallback; subtle plugin-version drift; a wider supply-chain trust surface).

- **Bake a global `<mirror>` into the `maven-builder` image (`$MAVEN_HOME/conf/settings.xml`).**
  Rejected.
  Strimzi invokes Maven with `mvn -s`, which overrides the *user* settings but not the *global* settings, so a global mirror baked into the image would apply to **every** user's build and silently override Maven Central for external customers.
  There is also no single correct mirror URL to bake in.
  This proposal instead writes the mirror into the per-build user settings, so it applies only when and where the user asks for it.

- **Rely solely on retry/backoff for HTTP 429.**
  Rejected.
  Retries do not help the air-gapped case (no route to Central at all) and only mask, rather than remove, the dependency on Central for build plugins.
