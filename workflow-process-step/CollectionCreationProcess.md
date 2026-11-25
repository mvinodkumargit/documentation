# CollectionCreationProcess

**Purpose:** select/create a dated DAM collection and add the asset.

**Models:** vendor-provided-images and sample-images. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java).

## Inputs

The step prefers workflow metadata `asset-path` when present; otherwise it resolves the original payload after stripping `/jcr:content`. It reads DAM metadata `fobName`, `groupingType`, `vendorProvidedImage`, and `directShipFlag`. A missing `directShipFlag` is treated as false.

## Selection order

1. If `fobName` equals `BEAUTY` ignoring case, use `/content/dam/collections/beauty`.
2. Otherwise, if the `groupingType` property **exists**, use the VPI or sample grouping collection according to `vendorProvidedImage`.
3. Otherwise, if the resolved asset path contains `/wip/samples`, use `/content/dam/collections/samples`.
4. Otherwise, if the path contains `/wip/vpi` and `directShipFlag` is true, use `/content/dam/collections/vpi/dropship`.
5. Otherwise, use `/content/dam/collections/vpi/non-dropship`.

The process derives a dated collection name, creates the collection if absent, adds the asset, and commits. [Routing](../../core/src/main/java/com/belk/aem/assets/core/workflows/CollectionCreationProcess.java#L87), [collection naming](../../core/src/main/java/com/belk/aem/assets/core/utils/DamAssetUtils.java#L223)

## Other writes and failure behavior

- If a moved path exists and the current model is `belk-sample-images`, the process attempts to set the moved asset's `jcr:content/dam:assetState` to `processed`, including for Beauty/grouping branches.
- No external service is called. Missing resolver, asset, metadata, or collection manager returns early. A missing vendor Boolean can fail in the grouping branch.
- Collection helpers and the outer process catch/log persistence failures. They can still proceed or mark the asset processed, so check actual collection membership and metadata.
