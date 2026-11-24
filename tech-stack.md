# Technology stack and build

## Runtime and languages

- **Platform:** Adobe Experience Manager as a Cloud Service SDK/API, with Apache Sling, OSGi, JCR/DAM, Granite UI, QueryBuilder, and AEM Workflow APIs.
- **Java:** Maven enforcer requires Java 21 in the root build configuration, while the compiler is configured with `release` 11 in [pom.xml](../pom.xml#L80). This may be intentional bytecode targeting, but the runtime/build distinction should be documented and tested in CI.
- **AEM SDK API:** `2026.5.25892.20260506T135241Z-260400` (root property).
- **AEM Core Components:** `2.28.0`.
- **ACS AEM Commons:** `6.17.2` (provided bundle dependency and packaged all-in-one dependency).
- **Other implementation formats:** JavaScript client library code, JSP/HTL, XML/JCR node definitions, JSON OSGi configuration, and Apache Dispatcher configuration.
- **No relational database or ORM** is declared; application data is stored in AEM's JCR/DAM repository.

## Build and packaging

- Maven reactor with seven modules; root project coordinates are `com.belk.aem.assets:belkassets:1.0.0`.
- Core is packaged as an OSGi bundle using Sling and BND Maven plugins.
- AEM content packages use FileVault and the Content Package Maven Plugin.
- `ui.apps` runs HTL validation during `generate-sources`.
- The `all` package embeds `ui.apps`, `core`, `ui.content`, `ui.config`, and ACS AEM Commons.
- The root POM configures Maven 3.3.9 or later and Java 21 for the Maven process; the README documents `mvn clean install`.

## Major libraries and tooling

- AEM SDK API and Core Components
- ACS AEM Commons
- Apache Sling APIs, Sling Models, Sling Jobs, QueryBuilder, and scripting APIs
- AEM HTTP client builder factory for outbound HTTP
- Gson for JSON
- JUnit 5, Mockito, AEM Mocks, Sling CAConfig mocks, and `slf4j-test` for core unit tests
- JaCoCo `0.8.12` configured for the core module
- FileVault, BND, Sling, HTL, AEM Analyzer, and Maven plugins for build/package validation

## Test footprint

The repository includes core unit tests for several workflow processes, schedulers, integration clients, and utilities under `core/src/test`. The root `README.md` describes integration/UI test modules and a frontend module that are not declared in the actual Maven reactor. No equivalent launcher-configuration, Dispatcher-rule, or author-client test module is declared by the root POM.

## External configuration

Runmode-specific configurations are in `ui.config`. Salsify and Workhorse endpoint/auth properties are supplied there, and the service-user mapping and RepoInit scripts establish the runtime principal and repository permissions. Some Java implementations also have non-empty source defaults; see [technical gaps](technical-gaps.md#credential-and-logging-hygiene).
