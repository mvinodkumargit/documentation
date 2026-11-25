# AssetMoveProcess

**Purpose:** move an asset to a grouping/FOB-based location and pass its destination to subsequent workflow steps.

**Models:** sample-images and post-processing-sample-vpi. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/AssetMoveProcess.java), [move and destination helpers](../../core/src/main/java/com/belk/aem/assets/core/utils/BelkUtils.java#L116).

## Inputs and decisions

1. Resolve the payload asset, stripping `/jcr:content` where present.
2. Read DAM metadata `groupingType` and `fobName`. A nonempty grouping type takes precedence and yields `grouping`; otherwise FOB is lowercased and spaces are replaced by hyphens.
3. Read `vendorProvidedImage` and `directShipFlag` to build the destination. A null `directShipFlag` defaults to false. The helper appends the current year and month:
   - Payload containing `/product/final-assets`: replace `final-assets` in the parent path with `samples/<fob>` for non-VPI, or `vpi/dropship/<fob>` / `vpi/non-dropship/<fob>` for VPI.
   - Other payloads with `vendorProvidedImage == true`: build `/content/dam/<language>/wip/vpi/dropship/<fob>` or `.../non-dropship/<fob>`.
   - Other non-VPI payloads: replace `final-assets` in the current parent path with the grouping/FOB value; if that text is absent, the parent path is not changed by the replacement.
4. If the destination already exists, the collision helper attempts to version the existing asset, copy the incoming asset over it, and delete the incoming resource. Otherwise it calls DAM `moveAsset`. [Collision helper](../../core/src/main/java/com/belk/aem/assets/core/utils/BelkAssetUtils.java#L37)

## Outputs and integrations

On success the process stores the destination in workflow metadata as `asset-path` and commits. This is a DAM repository operation, not a direct Salsify, Workhorse, or Scene7 request.

## Conditions and failure behavior

- If neither nonempty grouping nor FOB is available, the process moves the asset to uncategorized, commits, and terminates.
- An empty result from the move helper terminates the workflow; missing resolver, manager, or asset returns early.
- Persistence exceptions are logged. Null vendor metadata or failures inside the collision helper can also prevent the intended move; inspect the destination and workflow history.
