# Detailed product and WIP asset flows

## Important distinction

The repository does **not** define six independent flows for Products/WIP × Dropship/Non-Dropship/Samples. Product original-rendition launchers do not check a Dropship flag. The only explicit Dropship/Non-Dropship decision found is in WIP VPI collection creation, based on `directShipFlag` metadata. The WIP sample model is defined, but no one of the five repository launcher configs directly starts it.

The labels below describe what the repository code actually does. They do not prove how uploaded files are classified or routed in a deployed AEM instance.

## Products

### Products — Dropship

**No distinct product Dropship flow is configured in this repository.**

- Product original renditions under `/content/dam/belk/en/product/(vpi|samples)/.../renditions/original` start Adobe's `/var/workflow/models/dam/dynamic-media-reupload` model on modified events. The launcher has no `directShipFlag` condition, and the target model is not defined here. [Product Dynamic Media launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_product_asset_mod_DM/.content.xml#L5)
- The separate product metadata-save launcher requires a nonempty `scene7Url` and `appliesToAllProducts == true`; it starts the Belk Salsify-on-save model. It does not branch on Dropship. [Metadata-save launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk-asset-metadata-save/.content.xml#L5)
- The Dropship flag does not select a product workflow or collection in the product post-processing model. That flow moves according to FOB and product path; see [product post-processing](#products--non-dropship).

### Products — Non-Dropship

**No distinct product Non-Dropship flow is configured either.** The same product original-rendition launcher and metadata-save launcher apply regardless of `directShipFlag`. No product-specific Drop/Non-Drop branching condition was found in the repository launchers or product post-processing model.

The WIP collection branch must not be generalized to Products: `CollectionCreationProcess` applies its Dropship-vs-Non-Dropship selection only to the WIP VPI path. [CollectionCreationProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L122)

### Products — Samples

The product `samples` path is included in two separate event-driven paths; neither is a Dropship/Non-Dropship split:

1. **Original rendition modified:** the same product launcher used for product VPI also matches `samples` and starts Adobe Dynamic Media re-upload. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_product_asset_mod_DM/.content.xml#L12)
2. **Scene7 publish completed:** when product VPI/sample metadata reaches `dam:scene7FileStatus == PublishComplete` and the configured property-change exclusions pass, the launcher starts the Belk post-Scene7 model. [Launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk-post-scene7-url-to-salisfy/.content.xml#L5)
   - Extract color-swatch metadata and commit repository changes. [ExtractMetadataForColorSwatchesProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/ExtractMetadataForColorSwatchesProcess.java#L89)
   - Send the metadata map to Salsify. [SalsifyDataUpdateProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyDataUpdateProcess.java#L44)
   - Invalidate the Scene7 CDN cache, then update Workhorse status. [Scene7CDNCacheInvalidation.java](../core/src/main/java/com/belk/aem/assets/core/workflows/Scene7CDNCacheInvalidation.java#L95), [WorkHorseStatusProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/WorkHorseStatusProcess.java#L75)

There is a separate **post-processing** model selected by `InvokeWorkflowModel` only when its input payload contains the configured `/product/final-assets` marker. This is not proof that every product sample upload enters that model; the product-original launcher above targets Adobe Dynamic Media instead. [Routing](../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L85), [product marker](../core/src/main/java/com/belk/aem/assets/core/constants/BelkConstants.java#L23)

That post-processing model runs:

1. **Filename metadata extraction:** receives two regex arguments for four- or six-token underscore-separated filenames. It strips a `jcr:content` suffix, parses filename values, and stores them as workflow metadata. Parsed tokens map to vendor ID, style ID, shot, and NRF color. [Model arguments](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-post-processing-sample-vpi/.content.xml#L27), [extractor](../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L43), [token mapping](../core/src/main/java/com/belk/aem/assets/core/utils/BelkUtils.java#L45)
2. **DAM metadata update:** looks up product data from Salsify using filename-derived fields, maps returned data into asset metadata, and commits. [DAM metadata step](../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L153)
3. **Asset move:** derives FOB from asset metadata. Missing FOB moves the asset to uncategorized, commits, and terminates. Otherwise it calls `BelkUtils.moveAsset`; an empty result terminates, while success stores the destination in workflow metadata and commits. [AssetMoveProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/AssetMoveProcess.java#L55), [move helper](../core/src/main/java/com/belk/aem/assets/core/utils/BelkUtils.java#L154)
4. **PSD/TIFF association:** requires the moved-asset path, queries matching assets using NRF color, shot, style, and vendor metadata, adds relations not already present, commits, and stores the matches in workflow metadata. [AssociatePSDImageProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/AssociatePSDImageProcess.java#L60)
5. **Related-asset metadata copy:** requires a moved path and prior match list, copies metadata from the moved asset to matched related assets, and commits. Despite its name, this implementation does not call Salsify. [UpdateRelatedAssetSalsifyMetadata.java](../core/src/main/java/com/belk/aem/assets/core/workflows/UpdateRelatedAssetSalsifyMetadata.java#L39)
6. **Completion:** Adobe DAM workflow-completed step. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-post-processing-sample-vpi/.content.xml#L84)

No product Dropship/Non-Dropship branch is present in this sequence; the move decision uses FOB and product-path logic.

## WIP

### WIP — Dropship

This is an explicit **WIP VPI collection-routing** branch, not a separate workflow model:

1. A modified original rendition under `/content/dam/belk/en/wip/vpi/.../renditions/original` starts the vendor-provided-images model for non-video MIME types. The separate video launcher has the same path and target but tests for video MIME type. Both exclude workflow-process/PageManagerCopy events. [Non-video launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_vpi_asset_mod/.content.xml#L5), [video launcher](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_vpi_asset_mod_without_DM/.content.xml#L5)
2. **Filename extraction** parses the configured four- or six-token underscore filename format and places derived identifiers into workflow metadata. If the filename does not match, the process moves the asset to uncategorized, commits, and terminates (with a special case for the metadata-asset model). [Extractor](../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L108)
3. **DAM metadata update** calls Salsify using the parsed style/vendor/NRF-color values. Returned data is mapped to DAM metadata, including `directShipFlag`, vendor-provided-image/title fields, shot order, and expiration metadata for WIP VPI/sample assets; repository changes are committed. If product data is absent, it moves the asset to uncategorized, commits, and terminates. [Salsify lookup/mapping](../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L153), [metadata write](../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L203)
4. **Collection creation** reads the persisted `directShipFlag`. When true on the WIP VPI path, it selects the VPI Dropship collection, creates/adds the asset to a dated collection, and commits. [Flag read](../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L87), [route/commit](../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L122)
5. The model ends with Adobe's DAM workflow-completed step. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-vendor-provided-images/.content.xml#L61)

### WIP — Non-Dropship

The entry, filename parse, Salsify lookup, metadata write, and completion steps are the same as WIP Dropship. The distinction occurs only at collection creation:

- `directShipFlag == true` selects the Dropship collection.
- `directShipFlag == false` selects the Non-Dropship collection.
- A missing/null flag is treated as false, so it selects the Non-Dropship collection.

The configured branch and fallback are in [CollectionCreationProcess.java](../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L87). The launchers do not split on `directShipFlag`; it is read only after the metadata update step.

### WIP — Samples

The sample-images model is defined, but **no direct WIP sample launcher was found** among the five repository launcher configs. `InvokeWorkflowModel` can select this model when its payload path contains the configured `/wip/samples` marker; whether an external/manual caller starts that process is not established by this repository. [Routing](../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L85), [sample path marker](../core/src/main/java/com/belk/aem/assets/core/constants/BelkConstants.java#L24)

When started, the configured sequence is:

1. **Filename extraction** → parsed values in workflow metadata; nonmatching filenames go to uncategorized and terminate. [Model step](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L28), [process behavior](../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L126)
2. **DAM metadata update** → Salsify product lookup and DAM metadata commit; missing product data follows the uncategorized/terminate path. [Model step](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L40), [process](../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L188)
3. **Salsify e-commerce status** → calls `updateEcommSatusAtSalsify`. [Model step](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L52), [process](../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyEcommStatusUpdateProcess.java#L81)
4. **Workhorse status** → calls the Workhorse API only on its model/path and metadata branches; selected runtime exceptions are caught. [Process](../core/src/main/java/com/belk/aem/assets/core/workflows/WorkHorseStatusProcess.java#L75)
5. **Asset move** → uses FOB; missing FOB moves to uncategorized, commits, and terminates; a successful move stores the new path and commits. [Process](../core/src/main/java/com/belk/aem/assets/core/workflows/AssetMoveProcess.java#L55)
6. **Collection creation** → creates/adds the asset to a dated sample collection and sets `jcr:content/dam:assetState` to `processed` for moved sample assets; commits. This sample branch does not use `directShipFlag` to split Dropship vs. Non-Dropship. [Process](../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L114)
7. **Completion** → Adobe DAM workflow-completed step. [Model](../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L101)

## Shared routing and failure semantics

`InvokeWorkflowModel` first checks the current model ID:

- From the batch-thumbnail model, it reads a path array and starts the post-Scene7 model for product VPI/sample entries.
- From the post-processing model, it reads the workflow asset-path property and starts Scene7 batch upload when the value is present.
- Otherwise, it routes `/product/final-assets` to post-processing, `/wip/samples` to sample-images, and other paths to vendor-provided-images. [InvokeWorkflowModel.java](../core/src/main/java/com/belk/aem/assets/core/workflows/InvokeWorkflowModel.java#L43)

Some process steps commit and then terminate on validation/missing-data branches; others log selected persistence or external-call errors and allow the workflow to continue. Therefore an uncategorized move, an external API failure, and a workflow still marked active are different outcomes and should be diagnosed from the instance history/current step. See [workflow-in-progress-and-performance.md](workflow-in-progress-and-performance.md).

## Runtime limitations

The repo does not show how assets are initially assigned to product/WIP paths, whether other deployed AEM launchers invoke WIP Samples, or whether installed Adobe workflow models differ from repository references. Confirm the deployed launcher definitions, active model version, workflow payload, and runtime history before treating these flows as the definitive production sequence.

## Flow diagram

See [detailed asset-ingestion diagram](diagrams/detailed-asset-ingestion.mmd).
