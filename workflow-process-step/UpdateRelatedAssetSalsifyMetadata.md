# UpdateRelatedAssetSalsifyMetadata

**Purpose:** copy selected metadata from the moved asset to the WIP assets found by the previous association step.

**Model:** post-processing-sample-vpi. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/UpdateRelatedAssetSalsifyMetadata.java).

## Inputs and flow

1. Read workflow metadata `asset-path` and `assetList`; skip if the path or list is empty.
2. Read the moved asset's `jcr:content/metadata`.
3. Build a map containing only present properties from the helper's allowlist: `productId`, `taskId`, `styleId`, `vendorId`, `nrfColorCode`, `shotOrder`, `parentProductId`, `omniChannelBrand`, `fobName`, `refinementColor`, `pimNrfColorCode`, `iphCategory`, `specialSizes`, `directShipFlag`, `vendorProvidedImage`, `merchId`, and `deptNo`.
4. For each search hit whose asset still resolves, write those properties to that asset's DAM metadata. The shared helper attempts a commit per asset; the process also commits afterwards. [Allowlist](../../core/src/main/java/com/belk/aem/assets/core/utils/DamAssetUtils.java#L141)

## Integration and failure behavior

Despite its name, this class makes **no Salsify API call**. It copies DAM properties; it does not copy `groupingType`, expiration, or Scene7 URLs through this helper.

Missing hit assets are skipped. Repository and persistence errors can be logged without terminating the step, and per-asset commits mean partial updates are possible. A missing source asset or metadata value map can cause an unchecked failure.
