# Migration preparation and release handoff

The source baseline is pinned in `.github/migration-source.json`, currently main
commit `6210d4cbab53ad6b2a44ae48382d80a13d89a99d` from PR #7605. Preparatory CI can
build those foundations from source; no dependency on a formal publication is
required to review the migration, compile clients/examples, or run their tests.

## Prepared in this branch

- Replace JDK8/Spring Boot 2 source with the current main implementation; Java 17
  is the minimum, and Java 21 verification uses main's push-only policy.
- Import client API/HTTP registration modules and all current main client/starter
  tests. The inventory records every imported source/resource path and rejects
  accidental omissions. Main-owned foundation classes cannot be copied back here.
- Remove local registry API ownership, gateway beat/beat-starter ownership and Motan.
- Import nineteen client-facing example roots, including SDK providers with
  transitive client/example dependencies. Their shared example helpers use new
  client-only artifact names. SDK/protocol/registry dependencies continue to come
  from main.
- Separate client and foundation versions. Build unreleased foundations from a
  fixed SHA rather than tracking a moving branch or importing main client modules.
- Replay protocol-JAR fixtures through the actual HTTP sender, preserving request
  field types, the v1 headers, unquoted success responses and retry/lifecycle behavior.
- Disable deployment and enforce a release profile gate until reviewed published
  foundation versions, ownership handoff and runtime compatibility are available.

## Upgrading from the JDK8-adapted source

Use the restored main-owned packages in XML configuration and custom integrations:

| Old copied type | Current main-owned type |
| --- | --- |
| `org.apache.shenyu.client.core.register.config.ShenyuClientConfig` | `org.apache.shenyu.register.common.config.ShenyuClientConfig` |
| `org.apache.shenyu.client.core.register.config.ShenyuRegisterCenterConfig` | `org.apache.shenyu.register.common.config.ShenyuRegisterCenterConfig` |
| `org.apache.shenyu.client.core.register.HttpClientRegisterRepository` | `org.apache.shenyu.register.client.http.HttpClientRegisterRepository` |
| `org.apache.shenyu.client.core.dto.URIRegisterDTO` | `org.apache.shenyu.register.common.dto.URIRegisterDTO` |

The moved examples already use the current package names. The current listener
reads `shenyu.discovery.type` from the Spring environment and defaults to `local`
for URI publication; the old copied `discoveryLocalMode` workaround is unnecessary.
Current source retains `instanceInfo` population and the main shutdown/lifecycle implementation.
The prior copied JDK8 models are not a second protocol definition.

## Verification required before enabling a release

1. Merge/release the reviewed main protocol, shared MCP/register and foundation
   changes. Replace the preparation SNAPSHOT with a verified released
   `shenyu.foundation.version`, and remove source preparation from release CI.
2. Choose the external client's actual release version with the release owner.
   Do not use the temporary `-migration-SNAPSHOT` version as a release coordinate.
3. Confirm that main no longer deploys business-client coordinates once ownership
   transfers. Keep gateway beat modules in main. Remove main source only after
   the external artifacts and runtime compatibility jobs are ready.
4. Run the client candidate on Java 17, and Java 21 under the same main policy,
   against the latest stable and oldest explicitly supported released Admin and
   bootstrap versions. Pin actual image tags/digests in a reviewed client-owned
   matrix. Never add a client dependency to the main reactor for these tests.
5. Record main/client commits, exact dependency/image versions, JDKs, commands,
   route visibility, gateway proxy responses and shutdown/removal observations.
   Only then update the preparation/deployment guards through a reviewed PR.

## Client-owned runtime scenarios

Start Admin and bootstrap from the selected released images and a moved Spring MVC
example built from this candidate. Configure HTTP registration, matching Admin
credentials, a namespace, and `shenyu.discovery.type=local`. Verify metadata/rules
and URI registration, repeat heartbeat renewal, route visibility in Admin,
configuration synchronization and an actual proxied request. Stop the example and
verify the intended offline lifecycle according to the v1 contract.

For discovery, run the moved example against a registry provider with explicit
provider dependencies. Check instance add/update/removal and Admin's listener path,
then ensure bootstrap consumes the changed upstream set. Preserve `instanceInfo`
and its JSON-string representation. Register/URI port is numeric; instance beat
port is a string. Also test MCP nested metadata/namespace/request configuration and
all migrated RPC starters before claiming support for those runtime flows.

Local fixture tests deliberately accept `success` responses without capability or
version response headers, allowing new clients to work with older Admin endpoints.
Versionless legacy client acceptance remains tested in main. Breaking changes need
new protocol versions and dual-support Admin releases before client rollout.

Runtime evidence for released pairs is still outstanding. This preparation branch
does not mark #6366, #6367 or #7602 complete and does not delete main client source.

## Rollback and contribution

Before publication, restore the prior source/release owner from Git if preparation
cannot proceed. After client publication, roll back business services to a tested
client release and retain v1 support in Admin. Main rollback must not introduce a
reverse dependency or resume publishing transferred client coordinates.

Change DTOs/endpoints/defaults in main and add immutable protocol fixtures there.
Change annotation scanning, business-client registration/retry/shutdown and starter
behavior here. Keep the source inventory and all relevant tests when updating the
baseline. Changes affecting a wire contract require coordinated review in both
repositories; release timing does not have to be synchronized.
