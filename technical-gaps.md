# Technical gaps and technical debt

These are source-based findings, not claims about production incidents. “Verify” items require checking deployed AEM configuration or actual external credentials before changing behavior.

## Credential and logging hygiene

- **Source defaults for credentials:** Salsify and Workhorse implementations have non-empty fallback values in Java in addition to runmode configuration. The repository does not establish whether those defaults are active or valid. Remove credential defaults, require explicit secret configuration, and rotate any value that is or may have been real. See [SalsifyApiImpl.java](../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L32) and [WorkhorseApiImpl.java](../core/src/main/java/com/belk/aem/assets/core/service/impl/WorkhorseApiImpl.java#L26).
- **Sensitive data can enter logs:** the shared HTTP client logs complete URLs, while Salsify methods log request JSON; the lookup request includes a token in query construction. Redact query parameters and payload fields before production logging. See [RestClient.java](../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L61) and [SalsifyApiImpl.java](../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L266).
- **Permission breadth needs review:** the service principal is granted `jcr:all` on `/content`, broader than the explicit read/write grants used for other paths. Confirm the required operations, then narrow grants and test workflows with the reduced principal. See [service-user RepoInit](../ui.config/src/main/content/jcr_root/apps/belkassets/osgiconfig/config.author/org.apache.sling.jcr.repoinit.RepositoryInitializer~belkassets-service-users.cfg.json#L1).

## Integration reliability and observability

- **HTTP outcomes are ambiguous:** PUT initializes status to `0` and may return that after exceptions; callers often discard returned status codes. GET/POST log non-200 responses without providing a consistent typed result to workflow callers. See [RestClient.java](../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L184), [SalsifyApiImpl.java](../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L339), and [WorkhorseApiImpl.java](../core/src/main/java/com/belk/aem/assets/core/service/impl/WorkhorseApiImpl.java#L83).
- **Query construction is fragile:** GET/POST parameters are concatenated without URL encoding. A value with reserved characters can change request semantics.
- **Retry and idempotency vary by step:** the TCS process has a bounded retry loop, while workflow process steps otherwise catch/log or terminate in different ways. An external write may succeed while a later repository update fails, leaving a partially completed flow.
- **Workflow completion signals can be misleading:** asset purge catches per-asset failures and may still return job success; logs alone may not give an operator a reliable count of failed/pending assets.
- **Potential search workload/log amplification:** the custom predicate performs separate existence queries and logs query inputs and matching metadata at INFO. Measure query cost and reduce routine data-rich logging.

## Authoring/action consistency

- **Synchronous browser request:** the TCS client uses a synchronous POST, which can block the authoring UI.
- **Two-step partial completion:** adding to the target collection and then removing from the current collection are separate operations. If removal fails after a successful add, the UI operation is partially applied.
- **Fixed destination:** the collection path is hard-coded in client code. The repository does not prove that the destination collection is provisioned in each environment.
- **Render-condition assumptions:** the JSP adapts a resource to a JCR `Node` without a visible null guard and uses substring matching for the TCS collection exclusion.

See the action flow in [flows.md](flows.md#author-action-move-to-tcs-collection).

## Build, tests, and repository documentation

- **Java version mismatch in configuration:** the build requires Java 21 while compiling with release 11. Record the intended runtime/bytecode policy and validate it in CI and Cloud Manager.
- **README drift:** the top-level README describes `it.tests`, `ui.tests`, and `ui.frontend` modules; those modules are not in the root Maven reactor, which declares seven different modules.
- **Test boundary:** unit tests exist in `core`; no test module is declared for author-client behavior, workflow launcher conditions, content-package configuration, or Dispatcher rules. These configuration-driven behaviors are important but not currently evidenced by repository tests.
- **Runtime-owned flows are only partially discoverable:** some repository workflow models appear manually/external started, and Adobe-provided models/processes are referenced. AEM runtime export/runbook evidence is needed to verify actual triggers and model versions.
- **TCS destination provisioning is unverified:** the client references a fixed destination path, but no matching content path was found in the inspected repository content package.

## Working-tree qualification

At analysis time, the working tree already contained modified `pom.xml` and RepoInit configuration files, plus untracked files including `core/src/main/java/com/belk/aem/assets/core/search/`. The search predicate findings describe the current workspace file; confirm it is intentionally tracked and packaged before treating it as released functionality.
