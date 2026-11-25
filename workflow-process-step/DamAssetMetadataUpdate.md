# DamAssetMetadataUpdate

**Purpose:** retrieve product information using filename-derived identifiers and update the asset's DAM metadata.

**Models:** vendor-provided-images, sample-images, post-processing-sample-vpi, and metadata-asset-workflow. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java).

## Inputs and lookup

1. Resolve the payload asset after stripping `/jcr:content`; read workflow metadata `dam:styleID`, `dam:vendorID`, `dam:shot`, and `dam:nrfColor`.
2. Set JCR observation user data to `changedByWorkflowProcess`, then call `SalsifyApi.getProductDataJSON` using style, vendor, and NRF color.
3. The service tries up to three sequential product queries: parent product plus NRF/color-master, style plus NRF, then parent-product ID. It uses the first returned product and takes the first value of each response property. [Salsify lookup](../../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L200)

## Metadata written

The step maps recognized product properties into `jcr:content/metadata` when present. Its mapping includes:

| Salsify property | DAM metadata property |
|---|---|
| `product_id`, `rrd_task_id` | `productId`, `taskId` |
| `Style#`, `Vendor#`, `nrfColorCode` | `styleId`, `vendorId`, `nrfColorCode` |
| `Parent Product`, `OmniChannel Brand` | `parentProductId`, `omniChannelBrand` |
| `fobName`, `refinementColor`, `pim_nrfColorCode` | `fobName`, `refinementColor`, `pimNrfColorCode` |
| `iphCategory`, `Special Sizes` | `iphCategory`, `specialSizes` |
| `directShipFlag`, `groupingType` | `directShipFlag`, `groupingType` |
| `Merch ID`, `Dept#` | `merchId`, `deptNo` |

It also writes `shotOrder` from `dam:shot`, `dc:title` from the filename without extension, and `vendorProvidedImage`. The latter is set **true when the mapped `Vendor Images - A - imagePath` property is present**, otherwise false; the step overwrites the mapped value with that presence test. Strings equal to `true` or `false` are converted to Booleans. [Mapping and flag](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L54)

For payload paths containing `/wip/vpi` or `/wip/samples`, it also writes `prism:expirationDate`, calculated from configured expiration days. The helper attempts a commit and the process calls `resourceResolver.commit()` afterwards. [Metadata writes](../../core/src/main/java/com/belk/aem/assets/core/workflows/DamAssetMetadataUpdate.java#L203), [commit helper](../../core/src/main/java/com/belk/aem/assets/core/utils/DamAssetUtils.java#L39)

## Conditions and failure behavior

- No product or an `IOException`: normally move to uncategorized and terminate. The metadata-asset model terminates without the move.
- Missing resolver or asset manager returns early. Missing payload assets or unexpected null values can fail before the guarded lookup.
- Repository/persistence exceptions are logged; the shared metadata helper also logs some write/commit failures rather than propagating them. A success log alone does not prove all properties persisted.
- The lookup is synchronous and can issue three requests; HTTP timeout behavior depends on the configured client. This step reads from Salsify; it does not upload the image to Salsify.
