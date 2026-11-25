# Scene7CDNCacheInvalidation

**Purpose:** request CDN invalidation for the asset's stored Scene7 URLs.

**Models:** post-scene7-urls-to-salsify and scene7-cdn-cache-invalidation. [Source](../../core/src/main/java/com/belk/aem/assets/core/workflows/Scene7CDNCacheInvalidation.java).

## Inputs and flow

1. Accept a JCR-path payload, strip a trailing `jcr:content/metadata`, and resolve the DAM asset.
2. Resolve the Scene7 configuration for the asset path, falling back to the default configuration.
3. Read `scene7SwatchUrl` and `scene7Url` from DAM metadata. Build a Scene7 CDN invalidation request using both values and the configured company handle; the code does not filter null URL entries before adding them.
4. Send an XML HTTP POST to the Scene7 API using the resolved configuration. The request config sets connection and socket timeouts of five seconds.
5. Check whether the response text contains `<invalidationHandle>`; absence is treated as an error by this class.

## Writes and failure behavior

The step does not modify DAM metadata. It requests cache invalidation; finding a handle is not proof of completed CDN propagation.

The outer `execute` catches and logs exceptions, including missing assets/configuration and failed handle checks, without rethrowing them. It logs the response; protect logs appropriately. The HTTP client is closed, but this implementation does not close the service resource resolver it acquires. [Request and handling](../../core/src/main/java/com/belk/aem/assets/core/workflows/Scene7CDNCacheInvalidation.java#L95)
