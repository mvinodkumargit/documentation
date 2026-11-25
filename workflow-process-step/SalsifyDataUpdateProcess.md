# SalsifyDataUpdateProcess

**Purpose:** request an image URL update through the configured Salsify integration.

**Model:** post-scene7-urls-to-salsify, after URL extraction. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyDataUpdateProcess.java).

## Inputs and request

1. Resolve the payload asset, stripping a trailing `jcr:content/metadata` when present.
2. Read DAM metadata `productId`, `shotOrder`, `scene7Url`, `scene7SwatchUrl`, and `scene7ViewerUrl`; read the asset filename.
3. Build a map containing `asset_id`, `product_id` if available, and each non-null URL. URL field names are `Scene7 Images - <shotOrder> - mainImage URL`, `... swatchImage URL`, and `... viewerImage URL`.
4. Call `SalsifyApi.updateDataAtSalsify`, which serializes the map and issues an HTTP PUT through the configured integration endpoint. [Service](../../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L379)

The method can issue a request with an asset ID and no URL properties; it does not have a vendor or shot eligibility check.

## Writes and failure behavior

This step reads DAM metadata but does not update or commit it. The HTTP call requests a downstream change; it does not upload image bytes.

Missing resolver, metadata, or asset manager can cause an early return. The resource is dereferenced before an asset-null check, and a null `shotOrder` can fail when a URL field is present. The Salsify service ignores the HTTP status returned by `RestClient`, which logs failed responses and catches I/O failures. Therefore the process's success log does not confirm downstream acceptance. [HTTP behavior](../../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L196)
