# `belk-vendor-provided-images` — end-to-end flow

## Purpose and entry

The configured business result is to enrich a WIP VPI asset from filename/Salsify data and add it to a dated collection. The model is started by either of two author-mode launchers on a modified original rendition under `/content/dam/belk/en/wip/vpi/.../renditions/original`:

- non-video MIME type → this model
- video MIME type → this same model

Both launchers exclude workflow-process and PageManagerCopy events. They split on MIME type, **not** on Dropship status. [Non-video launcher](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_vpi_asset_mod/.content.xml#L5), [video launcher](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk_vpi_asset_mod_without_DM/.content.xml#L5)

The upload/import step before this repository-visible modified event is not implemented by this model. The checked-in product-original launcher is separate and starts Adobe Dynamic Media re-upload, not this model.

## Configured sequence

`FilenameMetadataExtraction` → `DamAssetMetadataUpdate` → `CollectionCreationProcess` → Adobe DAM workflow-completed process. [Model definition](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-vendor-provided-images/.content.xml#L24)

### 1. Parse filename into workflow metadata

The model supplies two filename regex arguments: four- and six-group underscore-separated alphanumeric patterns. [Arguments](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-vendor-provided-images/.content.xml#L28)

The process:

1. Reads the workflow payload and strips a trailing `jcr:content` suffix when present.
2. Checks the file extension and filename against the supplied patterns.
3. Maps parsed filename tokens to vendor ID, style ID, shot, and NRF color.
4. Stores the extracted identifiers as workflow metadata for later steps; it does not persist these parsed values to DAM metadata itself. [Implementation](../../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L43), [field mapping](../../core/src/main/java/com/belk/aem/assets/core/utils/BelkUtils.java#L45)

**Failure branches:**

- Missing extension or invalid filename: move the asset to uncategorized, commit, and terminate the workflow, except for the special metadata-asset model behavior. [Implementation](../../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L89)
- Persistence exceptions are logged. [Implementation](../../core/src/main/java/com/belk/aem/assets/core/workflows/FileNameMetadataExtraction.java#L146)

### 2. Look up Salsify product and write DAM metadata

The next step uses workflow metadata (style ID, vendor ID, NRF color) to query Salsify for product data. [Lookup](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L153)

When product data is returned, the process maps configured product properties, including `directShipFlag`, to DAM metadata; sets vendor-provided-image/title metadata; writes shot order; adds expiration metadata for WIP VPI/sample paths; and commits the asset metadata. [Property mapping](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L54), [metadata writes](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L169)

**Failure branches:**

- No product data: move to uncategorized, commit, and terminate. [Implementation](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L188)
- IOException: workflow termination is explicit; other listed exceptions may be logged without the same termination behavior. [Implementation](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L219)

The Salsify call here is a **product-data lookup used to populate DAM metadata**; it is not an upload of the image to Salsify.

### 3. Create/add to collection

The process reads the persisted `directShipFlag`; a missing/null value is treated as false. [Flag read](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L87)

For `/wip/vpi` assets:

- `directShipFlag == true` → select the VPI Dropship collection.
- `directShipFlag == false` or null → select the VPI Non-Dropship collection.
- The process creates or finds a dated collection, adds the asset, and commits repository changes. [Collection routing](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L122), [collection commit](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L213)

Collection persistence failures are caught/logged; a successful AEM step invocation does not independently prove that membership was committed. [Failure handling](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L251)

**Scope:** this is the repository's explicit Dropship/Non-Dropship branch. It applies to WIP VPI collection routing, not to the product VPI or product samples launcher.

### 4. AEM workflow end vs. business end

The final configured node is Adobe's DAM workflow-completed process. Its internal implementation is outside this repository. [Final model step](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-vendor-provided-images/.content.xml#L61)

The business outcome established by the Belk code is:

- **Success path:** asset metadata is enriched from Salsify data and the asset is added to a dated WIP VPI collection selected by direct-ship status.
- **Invalid filename or missing product data:** asset is moved to uncategorized and the workflow is terminated.
- **External downstream:** Salsify is used as a lookup source; this flow does not establish a Salsify image upload, FTP handoff, or Heroku submission.

The Adobe completion node is orchestration state, not a business acknowledgement from an external system.

## Operational and design caveats

- This model's launcher fires on modified original-rendition events and excludes only the configured workflow-process/PageManagerCopy cases. Confirm actual deployed launcher state and duplicate workflow instances when diagnosing repeated runs.
- External Salsify latency can delay the metadata step; the effective HTTP timeout is determined by the AEM client configuration and is not established by this workflow model.
- The model does not show a durable record of external acknowledgement or a complete retry/recovery contract.
