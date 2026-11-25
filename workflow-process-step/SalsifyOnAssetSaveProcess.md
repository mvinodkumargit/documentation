# SalsifyOnAssetSaveProcess

**Purpose:** conditionally request that existing image URLs be applied to related NRF products.

**Model:** asset-metadata-save. Its checked-in launcher targets Product metadata changes with a nonempty `scene7Url` and `appliesToAllProducts == true`. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyOnAssetSaveProcess.java), [launcher](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/launcher/config/belk-asset-metadata-save/.content.xml).

## Inputs and flow

1. Strip `/jcr:content` from the payload and resolve the asset and its metadata.
2. Read Boolean `appliesToAllProducts`. If true, read `shotOrder`, `scene7Url`, `scene7SwatchUrl`, and `scene7ViewerUrl`.
3. Add each non-null URL to a map under its shot-specific Scene7 image property. If at least one URL exists, add `asset_id` and call `updateImageURLforNRFAtSalsify`.
4. After the valid-metadata path, the process explicitly terminates the current workflow, whether or not the flag was true or a URL request was sent.

## Writes, integration, and errors

The step does not persist DAM metadata. The service sends the URL map as an HTTP PUT; its return status is not checked by the process/service. [Salsify service](../../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L396)

Missing resolver, manager, asset, or metadata may result in an early return before explicit termination. A missing asset resource, null `appliesToAllProducts`, or null `shotOrder` when a URL is present can cause an unchecked failure. A logged request or completion does not prove that downstream NRF products were updated.
