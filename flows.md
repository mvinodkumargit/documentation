# Existing runtime flows

This guide explains what causes repository-defined actions to run and what their configured steps do. It is based on repository configuration and source; external AEM-provided workflow steps and deployed runtime state are called out separately.

For triage of reported product uploads remaining in progress, see [workflow-in-progress-and-performance.md](workflow-in-progress-and-performance.md).
For low-level Product/WIP routing and Dropship/Non-Dropship/Samples distinctions, see [asset-ingestion-flows.md](asset-ingestion-flows.md).
For all ten Belk models, including nested invocation and downstream outcomes, see the [workflow catalog](workflows/README.md).

## Launcher rules

All five repository launchers are enabled author-mode `nt:file` modified-event launchers (`eventType=16`) rooted under `/content/dam`.

| Asset event / condition | Result |
|---|---|
| Original rendition below `/content/dam/belk/en/wip/vpi`; MIME type is not video; workflow-process and PageManagerCopy events excluded | Starts the Belk vendor-provided-images workflow. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_vpi_asset_mod/.content.xml#L5) |
| Same VPI original-rendition path; MIME type is video; same event exclusions | Starts the same vendor-provided-images workflow. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_vpi_asset_mod_without_DM/.content.xml#L5) |
| Original rendition below product `vpi` or `samples` paths | Starts Adobe's `/var/workflow/models/dam/dynamic-media-reupload` model, which is not defined in this repository. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_product_asset_mod_DM/.content.xml#L5) |
| Product metadata path; `scene7Url` is nonempty and `appliesToAllProducts == true` | Starts Belk asset-metadata-save. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk-asset-metadata-save/.content.xml#L5) |
| Product VPI/sample metadata path; `dam:scene7FileStatus == PublishComplete`; configured property-change exclusions apply | Starts Belk post-Scene7-URLs-to-Salsify. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk-post-scene7-url-to-salisfy/.content.xml#L5) |

The model targets, conditions, and paths are therefore configuration-driven. A matching repository event is not, by itself, proof that the external API or the complete workflow succeeds.

## Workflow models

There are ten repository-defined Belk models. The listed step sequence is configuration order; no branching nodes were found in the model definitions.

| Model | Ordered process steps | Trigger evidence |
|---|---|---|
| **Vendor-provided images** | Filename metadata extraction → DAM metadata update → collection creation → Adobe workflow completion | Started by both VPI launchers. Also used as fallback by `InvokeWorkflowModel` for payload paths outside product/sample routes. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-vendor-provided-images/.content.xml#L12) |
| **Asset metadata save** | Salsify update on asset save | Started by the product metadata-save launcher. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-asset-metadata-save/.content.xml#L11) |
| **Post Scene7 URLs to Salsify** | Extract color-swatch metadata → update Salsify data → invalidate Scene7 CDN cache → update Workhorse status | Started by the Scene7-publish-complete launcher; also started by `InvokeWorkflowModel` for qualifying product VPI/sample paths. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-post-scene7-urls-to-salsify/.content.xml#L11) |
| **Post-processing sample/VPI** | Filename extraction → DAM metadata update → asset move → PSD/TIFF association → related-asset Salsify metadata update → Adobe workflow completion | No direct launcher in the five repository launchers. `InvokeWorkflowModel` selects it for product-path payloads. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-post-processing-sample-vpi/.content.xml#L11) |
| **Sample images** | Filename extraction → DAM metadata update → Salsify e-commerce status → Workhorse status → asset move → collection creation → Adobe workflow completion | No direct launcher in the five repository launchers. `InvokeWorkflowModel` selects it for sample-path payloads. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L12) |
| **Scene7 batch upload** | Adobe Dynamic Media upload external process → Adobe workflow completion | No direct launcher. `InvokeWorkflowModel` starts it when the incoming workflow ID matches the post-processing model and an asset path is available. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-scene7-batch-upload/.content.xml#L11) |
| **Batch thumbnails post to Salsify** | Invoke workflow model → Adobe workflow completion | No repository launcher or caller was found; appears manually or externally started. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-batch-thumbnails-post-to-salsify-call/.content.xml#L11) |
| **Assign to TCS** | Add to TCS collection, using configured collection path/name arguments | No repository launcher or caller was found; appears manually or externally started. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-assign-to-tcs/.content.xml#L10) |
| **Scene7 CDN cache invalidation** | Invalidate Scene7 CDN cache | No repository launcher or caller was found; appears manually or externally started. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-scene7-cdn-cache-invalidation/.content.xml#L11) |
| **Metadata asset workflow** | Filename extraction → DAM metadata update | No repository launcher or caller was found; appears manually or externally started. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-metadata-asset-workflow/.content.xml#L10) |

“Manual or externally started” means no trigger/caller was found in the repository workflow/launcher configuration; it does not prove no runtime-created launcher or external caller exists.

## Workflow routing and process logic

`InvokeWorkflowModel` routes workflow payloads by workflow ID and asset path: it can select post-processing for product paths, sample-images for sample paths, and vendor-provided-images as a fallback; it can also start the Scene7 batch-upload flow for the corresponding post-processing workflow/payload. The routing lives in [InvokeWorkflowModel.java](../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L67).

Important process effects include:

- Filename extraction applies configured filename patterns to update asset metadata and may terminate the workflow on invalid input. [FileNameMetadataExtraction.java](../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L39)
- DAM metadata update mutates repository metadata, calls Salsify, and commits repository changes; error/termination behavior is implemented in [DamAssetMetadataUpdate.java](../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L123).
- Collection creation and asset move/association steps mutate repository content as their names and model configuration indicate; examples: [CollectionCreationProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L42), `AssetMoveProcess.java`, and `AssociatePSDImageProcess.java`.
- TCS membership retries collection updates/commits up to ten times in [AddToTCSCollectionProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/AddToTCSCollectionProcess.java#L105).
- Scene7 cache invalidation and Workhorse status updates have local exception handling; see [Scene7CDNCacheInvalidation.java](../core/src/main/java/com/belk/aem/assets/core/workflows/Scene7CDNCacheInvalidation.java#L95) and [WorkHorseStatusProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/WorkHorseStatusProcess.java#L33).

Failure behavior is not uniform across process steps: some explicitly terminate a workflow, some catch/log errors, and the repository does not show one shared retry/idempotency policy.

## Scheduled maintenance jobs

- **Asset purge:** `AssetPurgeScheduler` is a configurable Sling job consumer. It queries expired DAM assets below a configured root (default `/content/dam/belk/en/wip/`), removes versions and assets, commits in batches of 50, then enqueues audit/workflow cleanup jobs. A per-asset runtime error is logged and changes are reverted, but the overall job can still report success after processing. [AssetPurgeScheduler.java](../core/src/main/java/com/belk/aem/assets/core/schedulers/AssetPurgeScheduler.java#L62)
- **Collection purge:** `CollectionPurgeScheduler` checks the collection named for yesterday under configured roots and deletes it only if it exists and is empty. [CollectionPurgeScheduler.java](../core/src/main/java/com/belk/aem/assets/core/schedulers/CollectionPurgeScheduler.java#L48)

## Author action: Move To TCS Collection

1. The action appears in the DAM collection-details selection bar and is guarded by the `dam/gui/content/mainasset` render condition. [Action configuration](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/collections/collectiondetails/.content.xml#L17)
2. The render condition hides it for subassets/pages and for paths containing `tcs-assets-collection`. [Render condition](../ui.apps/src/main/content/jcr_root/apps/dam/gui/content/mainasset/mainasset.jsp#L29)
3. The client library collects selected asset paths and posts `:operation=add` to the fixed TCS collection endpoint. On success it tries to remove the selected paths from the current collection and presents a success dialog; failure paths include permission/cycle messaging. [Client library](../ui.apps/src/main/content/jcr_root/apps/belkassets/clientlibs/tcscollection/js/addtotcscollection.js#L19)

This is a browser-side collection operation rather than a direct call to the Salsify or Workhorse services. The repository does not establish that the fixed destination collection is present in every environment.

## DAM search extension

`MultiSearchPredicateEvaluator` registers the `multisearch` QueryBuilder predicate. It parses a path/property/code expression with optional extra predicates, limits metadata property names to an allowlist, checks for existing values, optionally creates a low-priority task when codes are missing, and contributes an XPath restriction. [MultiSearchPredicateEvaluator.java](../core/src/main/java/com/belk/aem/assets/core/search/eval/MultiSearchPredicateEvaluator.java#L47)

## Integration boundary

- `SalsifyApi` and `WorkhorseApi` are OSGi service interfaces, implemented by `SalsifyApiImpl` and `WorkhorseApiImpl`.
- Both use the shared `RestClient`, which provides GET/POST/PUT operations through AEM's HTTP client factory.
- Workflow steps call these services; the integration endpoints, property IDs, and auth configuration live in `ui.config`.
- Adobe Dynamic Media workflow steps/models are external dependencies and are not implemented in this repository.
