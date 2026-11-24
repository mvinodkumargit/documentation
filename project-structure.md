# Project structure and current architecture

## System classification

This is an AEM full-stack content application with Java/OSGi backend behavior and AEM authoring UI extensions. It is not a standalone SPA: the repository has no frontend build module, and the UI is delivered through AEM components/client libraries. Apache Dispatcher configuration is included.

## Maven reactor modules

The root [pom.xml](../pom.xml#L18) declares seven modules:

| Module | Responsibility |
|---|---|
| `core` | Java OSGi bundle: workflow process steps, scheduled job consumers, QueryBuilder predicate evaluator, integration services, and utilities. |
| `ui.apps` | `/apps` content package: AEM components/overlays, client libraries, and author UI customizations. |
| `ui.apps.structure` | Repository-structure package that declares roots used by the content packages. |
| `ui.config` | Runmode-specific OSGi configuration, service-user mapping, and RepoInit ACL/group setup. |
| `ui.content` | DAM/content configuration, metadata schemas, workflow models, and workflow launcher definitions. |
| `all` | Container package embedding the application packages and ACS AEM Commons. |
| `dispatcher` | Apache/Dispatcher farms, filters, cache, virtual hosts, and rewrite rules. |

The package relationships are visible in [all/pom.xml](../all/pom.xml#L34), [ui.apps/pom.xml](../ui.apps/pom.xml#L54), [ui.config/pom.xml](../ui.config/pom.xml#L46), and [ui.content/pom.xml](../ui.content/pom.xml#L53).

## Runtime layers and responsibility boundaries

1. **AEM authoring and repository configuration** — `ui.content` installs DAM metadata/search settings and workflow models/launchers; `ui.config` installs OSGi and repository security configuration.
2. **Author UI** — `ui.apps` adds author actions and overlays. For example, “Move To TCS Collection” is declared in [collectiondetails/.content.xml](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/collections/collectiondetails/.content.xml#L17), its browser behavior is in [addtotcscollection.js](../ui.apps/src/main/content/jcr_root/apps/belkassets/clientlibs/tcscollection/js/addtotcscollection.js#L19), and the action's render condition is in [mainasset.jsp](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/mainasset/mainasset.jsp#L29).
3. **AEM Java services** — `core` implements workflow steps, scheduled maintenance, outbound HTTP integrations, and a custom search predicate.
4. **Repository and external integrations** — workflows read/write DAM/JCR metadata and collections; integration services call Salsify, Workhorse, and Scene7-related endpoints.
5. **Delivery edge** — Dispatcher rewrites the site root to the configured content folder and uses an allowlist-oriented filter configuration. The root rewrite is in [rewrite.rules](../dispatcher/src/conf.d/rewrites/rewrite.rules#L10); default filters are in [default_filters.any](../dispatcher/src/conf.dispatcher.d/filters/default_filters.any#L9).

## Main functional areas

- Vendor-provided image (VPI) ingestion and filename-derived metadata
- Product/sample asset metadata updates and lifecycle movement
- Salsify product, image URL, and e-commerce status synchronization
- Scene7 publish completion, color-swatch metadata, and CDN cache invalidation
- Workhorse asset status updates
- DAM collection creation, assignment, and cleanup
- Expired WIP asset purging and empty-collection purging
- Author-side TCS collection action
- DAM search predicate extension for multi-value metadata matching

## Entry points

- **DAM event launchers:** five repository launcher definitions under `ui.content/.../workflow/launcher/config`; the filters and targets are detailed in [flows.md](flows.md#launcher-rules).
- **Workflow models:** ten repository-defined models under `ui.content/.../workflow/models/belk`; their steps and trigger evidence are in [flows.md](flows.md#workflow-models).
- **Scheduled jobs:** `AssetPurgeScheduler` and `CollectionPurgeScheduler` register Sling `JobConsumer` implementations.
- **Search:** `MultiSearchPredicateEvaluator` registers the `multisearch` QueryBuilder predicate.
- **Author UI:** the TCS action is triggered by selecting assets and invoking the collection-details action.
- **Public delivery:** Dispatcher routes public requests to AEM; it is separate from the author-side DAM workflows.

There are no in-scope custom servlets, Sling models/exporters, request filters, or event listeners in the inspected `core` Java tree.

The repository-defined authoring changes are cataloged in [DAM UI customizations](dam-ui-customizations.md).

## Current architecture

The application is predominantly workflow-driven. Repository events select workflow models; model steps then coordinate repository mutations and external systems. Scheduled jobs perform maintenance independently. Author UI actions use AEM collection operations directly. See the [current architecture diagram](diagrams/current-architecture.mmd) and [flow details](flows.md).
