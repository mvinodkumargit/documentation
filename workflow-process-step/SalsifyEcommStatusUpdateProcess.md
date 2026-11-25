# SalsifyEcommStatusUpdateProcess

**Purpose:** conditionally request an e-commerce photo-status update through the Salsify integration.

**Model:** sample-images. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/SalsifyEcommStatusUpdateProcess.java), [model step](../../ui.content/src/main/content/jcr_root/conf/global/settings/workflow/models/belk/belk-sample-images/.content.xml#L48).

## Inputs and condition

The step resolves the asset from the payload, reads its filename and DAM metadata `vendorProvidedImage`, `shotOrder`, and `productId`, and checks the current model ID. It calls the integration only if all three conditions hold:

1. `vendorProvidedImage` is false.
2. The model ID is `belk-sample-images` (case-insensitive comparison).
3. `shotOrder` is not `SW` (case-insensitive comparison).

The condition checks the model ID, **not** the WIP folder path.

## Request and output

The JSON request contains `asset_id` set to the filename, `Ecomm Photo Status` set to `Shots Selected`, and `product_id` only if the DAM `productId` is non-null. `SalsifyApi.updateEcommSatusAtSalsify` issues a PUT through the configured integration endpoint. [Service](../../core/src/main/java/com/belk/aem/assets/core/service/impl/SalsifyApiImpl.java#L409)

No DAM metadata or workflow metadata is written. If the condition is false, no status request is made.

## Failure behavior

A missing asset resource, null metadata map, null `vendorProvidedImage`, or null `shotOrder` on the eligible branch can cause an unchecked failure. Some missing resolver/manager/asset checks return early. The integration ignores the HTTP status returned by the client, so an INFO success message is not proof that the status was accepted. [HTTP behavior](../../core/src/main/java/com/belk/aem/assets/core/utils/RestClient.java#L196)
