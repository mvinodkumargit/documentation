# WorkHorseStatusProcess

**Purpose:** conditionally request a Workhorse image-status update.

**Models:** sample-images and post-scene7-urls-to-salsify. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/WorkHorseStatusProcess.java).

## Inputs and conditions

Resolve the original payload asset, remove its filename extension for `imageId`, and read `vendorProvidedImage` and `shotOrder` from DAM metadata. Both supported branches require `vendorProvidedImage == false` and a shot other than `SW`, ignoring case:

| Workflow model | Status sent | Thumbnail |
|---|---|---|
| `belk-sample-images` | `ShotsSelected` | Empty string |
| `belk-post-scene7-urls-to-salsify` | `FinalImageDelivered` | DAM `scene7Url`; service substitutes empty if null |

Other model IDs or nonqualifying metadata result in no request. The condition checks model ID and metadata, not the asset folder path.

## Output and failure behavior

The Workhorse service sends an HTTP PUT with `imageId`, `status`, and `thumbnailURL`; this step does not persist DAM metadata. [Service](../../core/src/main/java/com/belk/aem/assets/core/service/impl/WorkhorseApiImpl.java#L84)

The process treats HTTP 200 as success and logs other returned statuses. It catches/logs `NullPointerException`, `IllegalArgumentException`, and `IllegalStateException`, including failures from missing metadata, rather than necessarily failing the workflow. The HTTP client also logs I/O errors; verify Workhorse's response and state separately.
