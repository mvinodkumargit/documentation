# Belk DAM workflow catalog

This folder documents the ten Belk workflow models under `ui.content/.../workflow/models/belk`, their repository-visible starts, calls to other models, DAM/JCR side effects, external integrations, and business outcomes.

- [Vendor-provided images — detailed flow](belk-vendor-provided-images.md)
- [Sample images — detailed flow](belk-sample-images.md)
- [Workflow invocation map](workflow-invocation-map.mmd)

## What “start from ingestion” means here

The repository does not implement the upstream upload/import mechanism in these ten models. The source-visible flow begins after AEM has an asset and a change event is emitted. The five checked-in launcher configs are author-mode `nt:file` modified-event launchers (`eventType=16`) under `/content/dam`.

| Launcher condition | Target |
|---|---|
| Modified original under `/content/dam/belk/en/wip/vpi/.../renditions/original`, MIME is not video; workflow-process and PageManagerCopy events excluded | `belk-vendor-provided-images` |
| Same WIP VPI original path, MIME is video; same event exclusions | `belk-vendor-provided-images` |
| Modified original under product `vpi` or `samples` paths | Adobe `/var/workflow/models/dam/dynamic-media-reupload` (external to this repo) |
| Product metadata path with nonempty `scene7Url` and `appliesToAllProducts == true` | `belk-asset-metadata-save` |
| Product VPI/sample metadata reaches `dam:scene7FileStatus == PublishComplete`, subject to property-change exclusions | `belk-post-scene7-urls-to-salsify` |

The exact paths and event filters are in [flows.md](../flows.md#launcher-rules). In particular, the product-original launcher does not target a Belk model and is not the same model as the WIP VPI launcher.

## Ten models: starts, calls, and business outcomes

“Start unknown/manual/external” means no direct start was found in the repository's five launchers and model references. It does not rule out runtime-created AEM launchers, external callers, or manual starts. A configured “workflow completed” step is listed only as the AEM orchestration end; the business outcome is described separately.

| Model | Repository-visible start | Nested model calls | Business outcome reached by the model |
|---|---|---|---|
| [`belk-vendor-provided-images`](belk-vendor-provided-images.md) | WIP VPI original modified event; two launchers split video/non-video | None in configured model | Parses filename, enriches DAM metadata using a Salsify product lookup, and creates a dated WIP VPI collection chosen by `directShipFlag`. Invalid filename or missing Salsify product data can move the asset to uncategorized and terminate. |
| [`belk-sample-images`](belk-sample-images.md) | No direct launcher in the five; the model exists and `InvokeWorkflowModel` has a `/wip/samples` branch, but checked-in model wiring does not prove that branch is reached | No downstream model in its configured steps | Attempts Salsify e-commerce and Workhorse status updates, moves according to FOB, and creates a dated sample collection; for moved samples it sets asset state to `processed`. |
| `belk-post-processing-sample-vpi` | No direct launcher; `InvokeWorkflowModel` has a `/product/final-assets` route, but the branch is not shown wired by the checked-in model definitions | No configured child model | Salsify lookup enriches metadata; asset moves by FOB; matching PSD/TIFF assets are related; related metadata is copied. The related-metadata step does not itself call Salsify. |
| `belk-metadata-asset-workflow` | No direct launcher or configured invoker found | None | Filename/Salsify metadata processing only; no configured DAM-completion step. |
| `belk-asset-metadata-save` | Product metadata modified with `scene7Url` and `appliesToAllProducts` conditions | None | Attempts an NRF image URL update in Salsify, then terminates; external acceptance is not confirmed by the call site. |
| `belk-post-scene7-urls-to-salsify` | Scene7 publish-complete launcher; also started by batch-thumbnail processing for qualifying paths | No configured child model | Updates swatch/Scene7 metadata, attempts Salsify URL update, requests CDN invalidation, and conditionally updates Workhorse. |
| `belk-batch-thumbnails-post-to-salsify-call` | No direct launcher found; appears manual/external | Starts `belk-post-scene7-urls-to-salsify` for eligible paths in its `paths` payload | Dispatches child workflows for product VPI/sample paths; parent model completion does not prove child workflows completed or downstream calls succeeded. |
| `belk-scene7-batch-upload` | No direct launcher; an `InvokeWorkflowModel` code branch exists, but checked-in post-processing model does not configure that step | Intended code path starts this model from a post-processing workflow ID with an asset-path property; current model wiring does not prove it is invoked | Requests Adobe Dynamic Media upload through an Adobe external process; actual upload outcome/retry behavior is outside the repo. |
| `belk-scene7-cdn-cache-invalidation` | No direct launcher or configured invoker found | None | Requests Scene7 CDN invalidation and checks for an invalidation handle; exceptions are caught/logged. No further business confirmation is recorded. |
| `belk-assign-to-tcs` | No direct launcher or configured invoker found; appears manual/external | None | Adds the payload asset to the configured TCS collection and commits, with up to ten add/commit attempts. |

Model step order and source anchors are in the detailed source files linked above and in [the earlier flow inventory](../flows.md#workflow-models).

## Model invocation and downstream map

### Confirmed configured invocation

- The batch-thumbnails model configures an `InvokeWorkflowModel` process step. For that model ID, the process reads a `paths` property and starts the post-Scene7 model for entries that contain the configured product VPI/sample path. If `paths` is absent, no child is started. [Model step](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-batch-thumbnails-post-to-salsify-call/.content.xml#L23), [invoker code](../../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L43)

### Code branches not shown wired to a model step

`InvokeWorkflowModel` also contains general routing by current model ID and payload path: `/product/final-assets` → post-processing; `/wip/samples` → sample-images; otherwise → vendor-provided-images. It contains a post-processing-model-ID branch that can start Scene7 batch upload when a moved asset path is present. However, the checked-in post-processing model does not include `InvokeWorkflowModel`, and the only checked-in model step using it is the batch-thumbnails model, whose special branch returns before the general routing. Treat these as code capabilities, not proof of normal runtime invocation. [Routing code](../../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L43), [configured invoker model](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-batch-thumbnails-post-to-salsify-call/.content.xml#L23), [post-processing model steps](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-post-processing-sample-vpi/.content.xml#L23)

### External systems and DAM outcomes

- **Salsify:** the workflow steps either look up product data to enrich DAM metadata or request image URL/e-commerce-status updates. A method call is not proof that Salsify accepted or applied the update.
- **Workhorse:** sample and post-Scene7 flows conditionally request asset-status updates; code logs selected failures/non-200 outcomes rather than recording a durable downstream acknowledgement.
- **Scene7/Dynamic Media:** repository code builds Scene7 URLs, sends CDN invalidation requests, and calls Adobe-managed upload workflow steps. Adobe Dynamic Media model internals are not included.
- **DAM/JCR:** workflows write metadata, move assets, create/add collection membership, create asset relations, copy metadata to related assets, and in the sample flow set `dam:assetState=processed`.
- **FOB:** used by asset move routing and WIP VPI collection decisions. The inspected core source does not establish a FOB → FTP → Heroku pipeline; FTP integration was not found. A Heroku endpoint exists among Salsify configuration/source defaults, but that is not evidence of an FTP handoff.
- **Crop:** the color-swatch process reads `dam:crop` to build Scene7 URL parameters; repository evidence does not show DAM crop generation or modification. [Crop URL handling](../../core/src/main/java/com/belk/aem/assets/core/workflows/ExtractMetadataForColorSwatchesProcess.java#L77)

## What counts as an end

There are three distinct end states in these flows:

1. **Business outcome:** repository mutation or an external update is attempted (asset moved, metadata committed, collection membership changed, Salsify/Workhorse/CDN request made).
2. **Workflow process outcome:** a process may continue, return early, terminate the workflow, or catch/log an error. These are not equivalent.
3. **AEM model outcome:** a final Adobe workflow-completed step is reached where configured. That step does not prove external acceptance or completion of any child workflow.

Some models have no explicit Adobe completion step, some terminate early by design, and some catch/log errors. Check each model's history and process outcome before interpreting its terminal state.

## Runtime evidence still required

The repository cannot confirm deployed launcher overrides, manually started models, the AEM upload mechanism before the original-rendition event, production external credentials/response status, Adobe-managed Dynamic Media behavior, or whether the child workflows finished. For an incident, capture the asset path, model ID, workflow instance ID, payload, active step, history, and correlated downstream response/logs.
