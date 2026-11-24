# `belk-sample-images` — end-to-end flow

## Purpose and entry

The configured business sequence attempts to enrich a sample asset, publish status to Salsify/Workhorse, move it according to FOB, and add it to a dated sample collection.

This model exists in the repository but none of the five checked-in launcher configs directly starts it. `InvokeWorkflowModel` contains a route for payloads containing the `/wip/samples` marker, but the only checked-in model step that configures `InvokeWorkflowModel` is the batch-thumbnails model. Its model-ID-specific branch processes a path array and returns before the general WIP sample routing. Therefore this repository does **not** establish the normal production start for this model; manual, runtime-created, or external starts remain possible. [Model](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L12), [invoker guard/routing](../../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L43), [configured invoker step](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-batch-thumbnails-post-to-salsify-call/.content.xml#L23)

## Configured sequence

Filename extraction → DAM metadata update → Salsify e-commerce status → Workhorse status → asset move → collection creation → Adobe DAM workflow-completed process. [Model definition](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L24)

### 1. Filename extraction

The model supplies the same four- or six-group underscore filename patterns as the VPI model. Parsed identifiers are placed in workflow metadata for later steps; they are not persisted to DAM by this step. Invalid/missing-extension filenames are moved to uncategorized, committed, and the workflow is terminated, subject to the metadata-asset model exception. [Model arguments](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L28), [process behavior](../../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L108)

### 2. Salsify lookup and DAM metadata

The DAM metadata process uses the extracted style/vendor/NRF-color fields to look up product data. It maps the returned properties to DAM metadata, writes shot order, adds expiration for WIP sample/VPI paths, and commits. Missing Salsify product data or the IOException branch can move the asset to uncategorized and terminate. [Lookup](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L153), [missing-data branch](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L188)

### 3. Request Salsify e-commerce status update

The e-commerce process builds a product/asset/status map and calls `SalsifyApi.updateEcommSatusAtSalsify(...)`. [Implementation](../../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyEcommStatusUpdateProcess.java#L76)

This establishes that the request is attempted; it does not establish Salsify accepted or applied the status.

### 4. Request Workhorse status update

The Workhorse process checks workflow/model, path, and metadata conditions. For qualifying sample assets it calls `updateWorkhorseStatus` with an empty URL and the configured sample status code. It logs non-200 responses and catches selected runtime exceptions. [Conditions and call](../../core/src/main/java/com/belk/aem/assets/core/workflows/WorkHorseStatusProcess.java#L75)

Again, an API call is not an external acknowledgement. This step's errors are not all represented as workflow termination.

### 5. Move asset using FOB

`AssetMoveProcess` derives FOB from asset metadata and delegates to `BelkUtils.moveAsset`. [Process](../../core/src/main/java/com/belk/aem/assets/core/workflows/AssetMoveProcess.java#L55)

- Missing FOB: move to uncategorized, commit, and terminate.
- Empty move result: terminate.
- Successful move: store the new asset path in workflow metadata and commit.

The workflow uses the resulting moved-path metadata in following steps. This is a DAM repository move, not evidence of an FTP or external upload.

### 6. Create sample collection and mark moved asset

For the sample path, `CollectionCreationProcess` creates/finds a dated sample collection, adds the asset, and commits. For moved sample assets it also sets `jcr:content/dam:assetState` to `processed`. [Sample branch](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L114), [asset state write](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L146)

No Dropship/Non-Dropship split is applied to this sample collection branch.

### 7. AEM workflow end vs. business end

The last configured node is Adobe's DAM workflow-completed process. [Model final step](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L97)

The repository-visible business outcome is: metadata enrichment attempted; Salsify e-commerce and conditional Workhorse status updates requested; asset moved by FOB; dated sample collection updated; moved asset marked `processed`. A workflow completion state does not prove the external status updates were accepted.

## Downstream and failure summary

| Step | Repository side effect | Downstream effect / failure behavior |
|---|---|---|
| Filename extraction | Workflow metadata; may move to uncategorized | Invalid format terminates. |
| DAM metadata update | DAM metadata and commit; may move to uncategorized | Reads product data from Salsify; missing data can terminate. |
| E-commerce status | None established beyond workflow step | Requests Salsify update; acceptance is not confirmed here. |
| Workhorse status | None established beyond workflow step | Conditional API request; selected errors are caught/logged. |
| Asset move | DAM path change and workflow moved-path metadata | Missing FOB/empty result terminates; successful move commits. |
| Collection creation | Dated collection membership and `dam:assetState=processed` | Repository persistence exceptions may be logged/caught. |

The model's direct start trigger, deployed runtime model version, effective HTTP timeouts, external response acceptance, and any runtime-created launcher are not established by this repository.
