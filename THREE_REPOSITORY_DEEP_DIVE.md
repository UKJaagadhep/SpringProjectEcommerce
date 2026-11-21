# Deep Architecture Report: `bi-pas-domain`, `bi-pas-fact-service`, and `bi-pas-dimension-service`

**Report date:** 2026-11-21

**Scope:** Detailed read-only analysis of the three private NIQ GitHub repositories:

- `niq-enterprise/bi-pas-domain`
- `niq-enterprise/bi-pas-fact-service`
- `niq-enterprise/bi-pas-dimension-service`

This report separates verified repository behavior from inferences. It does not claim that a service direct dependency exists when only shared infrastructure or deployment-level integration may be involved.

## 1. Executive summary

The three repositories implement a layered Platform Adapter Services architecture:

```text
External Report/Metadata Client
              |
              +--------------------------+
              |                          |
              v                          v
   bi-pas-fact-service        bi-pas-dimension-service
   Report and job orchestration Metadata HTTP facade
              |                          |
              +-------------+------------+
                            |
                            v
                  bi-pas-domain library
                  Shared DTOs, metadata handlers,
                  payload transformation, selection
                  builders, NodeIdUtil, REST utilities
                            |
                            v
                    BI/DMDS/PAPI adapters
                    Downstream BI and reporting stack
```

### `bi-pas-domain`

`bi-pas-domain` is a Java 21 Maven/Gradle library. It is not an executable Spring application:

- No `@SpringBootApplication`
- No `main` method
- No controller
- No database entity or repository
- No Kafka/JMS/RabbitMQ listener or producer
- No boot JAR

It publishes reusable classes and registers Spring configuration/components. It owns much of the report-payload transformation and metadata/DMDS routing logic consumed by both services.

### `bi-pas-fact-service`

`bi-pas-fact-service` is the richer runtime service. It owns report and job execution:

- `SimplePayload` to complex `BIWebRequest` transformation
- Synchronous and asynchronous BI report execution
- MongoDB-backed job/user state
- Job status and job detail lookup
- BI-package callback processing
- Flat and unified report-data conversion
- Request/selection validation
- Optional Hazelcast throttling
- SSE job-status broadcasting support
- Warmup and external-call timing
- BI package compacting

### `bi-pas-dimension-service`

`bi-pas-dimension-service` is a thin HTTP metadata adapter:

- One implemented controller endpoint
- Action-driven validation
- Delegation to `MetadataQueryHandler`
- DMDS adapter access
- Thread/request-context propagation
- AOP event logging, retry and timing annotations
- No local persistence
- No direct source dependency on fact-service

## 2. Repository snapshots

| Repository | Default branch | Observed Java snapshot | Build model | Primary role |
|---|---|---|---|---|
| `bi-pas-domain` | `main` | Java 21 | Published library JAR | Shared contracts, metadata strategies, payload and selection logic |
| `bi-pas-dimension-service` | `main` | Java 25 | Spring Boot executable | Dimension/metadata HTTP API |
| `bi-pas-fact-service` | `main` / local `feature/dec` checkout | Java 25 | Spring Boot executable | BI fact/report orchestration |

The fact-service local checkout used for this report was `C:\Users\uja5002\bi-pas-fact-service-main`. At analysis time it was on `feature/dec`, commit `294c2f8db372fd1fadb78274bd0116137ad9756d`, and contained existing untracked `deployment/` and `pas-fact-service-service-map.md` paths. This report does not modify those items.

## 3. `bi-pas-domain`

### 3.1 Repository shape

The domain repository is a Gradle library under:

- `src/main/java/com/niq/pas/`
- `src/main/resources/`
- `src/test/java/`
- `src/test/resources/`
- `config/checkstyle/`
- `.github/workflows/`
- `build.gradle`
- `gradle.properties`
- `settings.gradle`

The README contains no substantive API documentation. The repository is primarily understood through package structure and tests.

### 3.2 Build

Main build characteristics:

- Java toolchain 21
- Spring Boot plugin 3.5.11
- Spring dependency-management plugin
- JaCoCo
- Checkstyle
- Maven publishing
- Java library
- Eclipse support
- Standard JAR enabled
- Boot JAR disabled
- No `-plain` classifier

The build publishes a Maven publication using:

```groovy
from components.java
```

to an internal NielsenIQ/NIQ BI publishing Artifactory repository.

Main external dependencies:

- Spring Boot starter and Spring Web
- Jackson core/databind/annotations
- Jackson MessagePack
- Lombok
- Apache Commons Collections, Lang, Text and IO
- Apache HttpClient 5
- Jsoup
- Jakarta/Javax validation
- `com.nielsen.gps.aw:bi-services-adapter`
- `com.nielsen.gps.awaf:gps-domain`
- `cs-commons-logging`
- `cs-commons-security`

### 3.3 Production package inventory

#### `com.niq.pas.config`

- `DmdsStrategyConfig`
- `RestTemplateConfig`

`DmdsStrategyConfig` registers endpoint-specific DMDS strategies. `RestTemplateConfig` creates a pooled Apache HTTP connection manager and shared Spring `RestTemplate` with connect, socket, and request timeouts. It disables cookie management.

#### `com.niq.pas.domain`

- `BIPasResponse`
- `ContextProvider`

These are small response/context abstractions. No persistence annotations or entities are present.

#### `com.niq.pas.dto`

Principal DTOs:

- `BIPackageDto`
- `BIPasResponseDTO`
- `CrfResponse`
- `DataViewResponse`
- `DatasetResponse`
- `DecodeResponse`
- `DimensionResponse`
- `EnrichmentContext`
- `FilterValueResponse`
- `ItemOption`
- `MetadataNode`
- `MetadataQuery`
- `PasServiceLoggingDto`
- `ReportDataRequestDTO`
- `ReportOption`
- `SimplePayload`
- `Type`
- `ValueItemOption`

The DTO classes generally use Lombok builders/data classes and Jackson-compatible serialization.

Principal enum classes:

- `ActionType`
- `BSGroupType`
- `LoyaltySubGroupType`
- `PPLType`
- `Region`
- `SelectionBuilderType`
- `ShareClassification`
- `Status`

#### `com.niq.pas.exception`

Contains:

- `BIPasException`
- `BIPasErrorCode`
- `BIPasApiErrorCode`
- `BIPasErrorhandler`

These form the shared application error model consumed by both executable services.

#### `com.niq.pas.service`

Core classes:

- `MetadataQueryHandler`
- `MetadataEnrichmentService`
- `DownstreamServiceClient`
- `PayloadTransformer`
- `SelectionBuilder`
- `SelectionBuilderResolver`
- `SelectionGroupContext`
- `CompactItemContext`

Specialized selection builders:

- `PODSelectionBuilder`
- `PPLSelectionBuilder`
- `DealSelectionBuilder`
- `LoyaltySelectionBuilder`
- `NonConvertedSelectionBuilder`
- `ExclusiveBuyerSelectionBuilder`
- `ExclusiveRetailerSelectionBuilder`
- `ShopperExclusivitySelectionBuilder`

Logging:

- `service/log/AppLogService`
- `service/log/EventLogService`

#### `com.niq.pas.service.strategy`

- `DmdsRequestStrategy`
- `AbstractDmdsRequestStrategy`
- `GenericDmdsStrategy`

This package implements endpoint strategy and generic external adapter dispatch.

#### `com.niq.pas.util`

- `BIPasDomainConstants`
- `CompletableFutureUtils`
- `CustomHeader`
- `FileUtil`
- `JsonUtil`
- `ListUtil`
- `LogCommonUtil`
- `LoggerUtil`
- `NoHtml`
- `NoHtmlValidator`
- `NodeIdUtil`
- `ObjectMapperConfigurer`
- `PPLSelectionUtil`
- `RestInvocationHelper`
- `XSSUtils`

### 3.4 `MetadataQuery`

`MetadataQuery` is the central metadata transport model. It contains:

- `action`
- request context
- dimension ID and name
- encoded node ID
- pagination
- filter criteria
- join mode
- dataset/dataview identifiers
- client and stack context
- locale
- product-group keys

Nested request context and filter structures support:

- `IN`
- `EQUAL`
- `NOT_IN`
- `LIKE`

### 3.5 DMDS strategy flow

The principal metadata flow is:

1. A consuming service creates `MetadataQuery`.
2. `MetadataQueryHandler` resolves an endpoint to `DmdsRequestStrategy`.
3. `DmdsStrategyConfig` supplies a `GenericDmdsStrategy`.
4. `GenericDmdsStrategy` creates an external DMDS request.
5. Request context and endpoint fields are populated.
6. Encoded node IDs are decoded where applicable.
7. `MetadataQuery` filter options are transformed into `CDFilter`.
8. Reflection attempts to call `setFilter(CDFilter.class)` on external request types.
9. The configured `DMDSAdapter` method is invoked.
10. `CDGenericWebResponse` is returned to the handler.
11. The handler adapts or converts it to PAS-facing types.

Supported strategies include:

| Endpoint | External request | Adapter method |
|---|---|---|
| `/datasets` | `CDCategoriesRequest` | `getCategories` |
| `/dataset/dimensions` | `CDDimensionsRequest` | `getDimensions` |
| `/groups` | `CDGroupsRequest` | `getGroups` |
| `/hierarchies` | `CDHierarchiesRequest` | `getHierarchies` |
| `/characteristics` | `CDCharacteristicsRequest` | `getCharacteristics` |
| `/characteristics:baseItem` | `CDBaseItemCharRequest` | `getBaseItemCharacteristics` |
| `/characteristic/values` | `CDCharacteristicValuesRequest` | `getCharacteristicValues` |
| `/group/levels` | `CDGroupLevelsRequest` | `getGroupLevels` |
| `/group/level/values` | `CDGroupLevelItemsRequest` | `getGroupLevelItems` |
| `/hierarchy/levels` | `CDHierarchyLevelsRequest` | `getHierarchyLevels` |
| `/hierarchy/level/values` | `CDHierarchyLevelItemsRequest` | `getHierarchyLevelItems` |
| `/cmnly-used-items` | `SearchRequest` | `getCommonlyUsedItems` |

The product-group endpoint also uses a characteristics request and maps it to `getProductGroupValues`.

### 3.6 Generic reflection and filter behavior

External DMDS request classes do not share a single filter interface. `GenericDmdsStrategy` therefore uses reflection to locate `setFilter(CDFilter.class)`.

Behavior:

- `NoSuchMethodException` is tolerated.
- Other reflection failures are logged.
- A filter failure does not automatically abort the downstream request.
- The request proceeds without a filter when an external model cannot accept one.

This is fail-open behavior and is relevant to diagnosis: absent or ignored filters may indicate an unsupported external request shape rather than a metadata-handler bug.

### 3.7 Payload transformation

`PayloadTransformer` and `SelectionBuilder` convert simple report requests into complex downstream selection structures.

The flow handles:

- Standard dimensions
- Hierarchy dimensions
- Group dimensions
- Group levels
- Characteristics
- Characteristic values
- Grouped characteristics
- Compact dimensions
- Compact facts
- Fact selections
- Include/exclude semantics
- Missing branches
- Grouped-characteristic early exits

`SelectionBuilderResolver` resolves specialized builders using `SelectionBuilderType`:

- `PPL`
- `POD`
- `DEAL`
- `LOYALTY`
- `NON_CONVERTED`
- `EXCLUSIVE_RETAILER_SHOPPERS`
- `EXCLUSIVE_PRODUCT_BUYERS`
- `EXCLUSIVE_RETAILER_BUYERS`

A null builder type defaults to PPL. Missing a required Spring builder results in `IllegalStateException`.

### 3.8 Node IDs

`NodeIdUtil` encodes and decodes opaque node identifiers.

Supported node types include:

- `GROUP`
- `GROUP_LEVEL`
- `CHARACTERISTIC`
- `HIERARCHY`
- `HIERARCHY_LEVEL`
- `FILTER_VALUE`
- `SELECTION_ITEM`

The encoding format includes:

- format version
- node type
- URL-safe Base64
- MessagePack
- optional plain-text dot-separated representation

Decoded node values can contain:

- dimension ID/name
- parent ID/type
- raw ID/type
- characteristic ID
- display name

Malformed decoding returns `Optional.empty()`. Encoding failures are explicit exceptions.

### 3.9 Security and utilities

Security-oriented utilities include:

- `XSSUtils`
- `NoHtml`
- `NoHtmlValidator`
- `CustomHeader`
- shared security/commons dependencies

`RestTemplateConfig` is the HTTP base and uses pooled Apache HTTP components. `CompletableFutureUtils` and `RestInvocationHelper` cover async downstream handling and REST invocation.

### 3.10 Persistence and messaging

Confirmed absent:

- JPA/Hibernate entities
- Spring Data repositories
- JDBC/R2DBC
- Flyway/Liquibase
- Kafka/RabbitMQ/JMS implementations
- ActiveMQ producer/listener dependencies

The module is stateless and obtains data through external adapters.

### 3.11 Tests

The domain test suite is broad. It covers:

- Metadata endpoint routing
- DMDS request population
- Reflection-based filter assignment
- Node ID encoding/decoding
- Payload transformation
- Selection builders and all major branch variants
- Downstream REST invocation
- Async composition
- RestTemplate configuration
- XSS/HTML validation
- JSON and file utilities
- Logging utilities
- DTO and enum behavior

The test suite uses JUnit Platform and JaCoCo.

## 4. `bi-pas-dimension-service`

### 4.1 Repository shape

Principal production paths:

- `src/main/java/com/niq/pas/BIPasDimensionServiceApplication.java`
- `src/main/java/com/niq/pas/dimservice/controllers/`
- `src/main/java/com/niq/pas/dimservice/service/`
- `src/main/java/com/niq/pas/dimservice/mvc/`
- `src/main/java/com/niq/pas/dimservice/aspect/`
- `src/main/java/com/niq/pas/dimservice/handler/`
- `src/main/java/com/niq/pas/dimservice/util/`
- `src/main/java/com/niq/pas/dimservice/constants/`
- `src/main/java/com/niq/pas/dimservice/log/`

The repository is a single Gradle Spring Boot application.

### 4.2 Build

Key build characteristics:

- Java 25
- Spring Boot 3.5.14
- Dependency locking enabled
- Jackson BOM dependency management
- Maven Central and private NielsenIQ/NIQ Artifactory repositories
- Spring Web and Actuator
- JSON/CBOR
- OAuth2 client
- AOP
- SpringDoc OpenAPI
- Log4j2
- Lombok
- `bi-pas-domain`
- common logging/security
- OkHttp
- Kafka clients
- Jackson MessagePack
- Checkstyle
- JaCoCo

JaCoCo enforces at least 80% line coverage for `com.niq.pas.*`.

The current observed domain version is:

```text
2026-11-05.main.378-SNAPSHOT
```

### 4.3 Runtime bootstrap

`BIPasDimensionServiceApplication`:

- uses `@SpringBootApplication`
- scans `com.niq.pas`
- excludes Spring Security and management security auto-configuration
- loads `bi-adapter.properties` into JVM system properties
- registers `ApplicationInfo`
- registers `AppLogService`
- registers `EventLogService`
- registers `DMDSAdapter.getInstance()`
- registers `ServicesFilter`
- registers a permissive CORS filter

Application constants:

- application code `RBOC_US`
- component code `RBOC_UDS`
- component display name `Dimension Service`
- API base `/bi-pas-dim-service/api`
- data route `/data`

### 4.4 Controller

Only one implemented controller route exists:

```http
POST /bi-pas-dim-service/api/data
```

The request body is `MetadataQuery`. The endpoint consumes and produces JSON/CBOR.

It:

1. delegates to `BIPasDimensionService.queryMetadata(payload)`
2. returns `ResponseEntity.ok(...)`
3. uses `@Eventlog`

The `/dataviews` constant exists but has no implemented controller mapping.

### 4.5 Service and action validation

`BIPasDimensionServiceImpl` exposes:

```java
Object queryMetadata(MetadataQuery payload)
```

Its dependencies are `AppLogService` and `MetadataQueryHandler`.

Execution:

1. reject a null payload
2. validate action-specific fields
3. obtain headers from `ExecutionContext`
4. add the encrypted-response header when absent
5. call `MetadataQueryHandler.queryMetadata(query, httpHeaders)`
6. log BI exceptions
7. rethrow the original or wrapped exception

#### Supported action validation

| Action | Required fields |
|---|---|
| `GET_DATAVIEWS` | dataset ID |
| `LIST_DIMENSIONS` | dataset ID, dataview ID |
| `GET_DIMENSION_ROOTS` | dataset ID, dataview ID, dimension ID |
| `GET_NODE_CHILDREN` | dataset ID, dataview ID, dimension ID, node ID |
| `GET_NODE_VALUES` | same identifiers, plus valid filters if options exist |
| `GET_DATASETS` | valid filters/options, join, filterBy and dataItemType if options exist |
| `GET_PRODUCT_GROUP_VALUES` | dataset ID, dataview ID, dimension ID |

Filter validation requires:

- non-null filter list
- non-null filter objects
- filter ID or `filterBy`
- operator
- values
- at least one nonblank value for dataset filters
- dataset `filterBy`
- dataset `dataItemType`
- dataset join

Unknown/null actions are not treated as success: null action is rejected; unknown enum values receive only the default switch branch and do not perform action-specific validation.

### 4.6 Request context and MVC

`ServicesFilter`:

- wraps requests in a repeatable-body `BufferedRequestWrapper`
- converts headers
- sanitizes the URI
- initializes `ExecutionContext`
- requires a transaction ID
- logs request metadata/body/non-sensitive headers
- invokes the handler chain
- merges external API timings
- stops the request stopwatch
- clears thread-local state

`ExecutionContext` uses thread-local state for:

- HTTP headers
- stopwatch
- debug values
- session
- user
- security token wallet
- processing times
- external API timing map

It validates security tokens via NIQ token infrastructure and provides authentication headers downstream.

It supports:

- transaction IDs
- parent/group transaction IDs
- locale
- target/origin/client/user-home stack IDs
- user identity
- encrypted-response preference

Default locale is English. Default multi-region stack values in `DMDSAdapterUtil` use `US`.

### 4.7 Security and CORS

Spring security auto-configurations are excluded.

Authentication is instead performed manually in `ExecutionContext.init()` using NIQ security-token libraries:

- access token
- client token
- ID token

Missing or invalid tokens become unauthorized responses.

CORS configuration is broad:

- credentials allowed
- origin pattern `*`
- any header
- any method
- `/bi-pas-dim-service/api/**`

No application-local `SecurityFilterChain` is present.

### 4.8 Exception handling

`CommonExceptionHandler` handles:

- `BIPasException`
- `HttpMessageNotReadableException`
- generic `Exception`

Responses contain:

- structured error message
- `BIPasErrorhandler`
- transaction ID
- elapsed request time
- appropriate HTTP status

Malformed filter operators receive `BIPasApiErrorCode.BI_PAS_MISSING_REQUIRED_INPUT`.

### 4.9 AOP and adapters

`@Eventlog` is an around-aspect annotation. It records:

- task start
- input
- success/failure
- key info
- task end
- duration
- transaction/user/context metadata

Logging failures are contained and do not fail the request.

`DmdsRetry` supports `delay` and `maxRetry`, but no local interceptor implementation was found. Actual retry behavior is likely platform/shared infrastructure.

`ExternalApiResponseTime` is a timing marker. It integrates with shared timing collection.

`DMDSAdapterUtil`:

- falls back to singleton BI/DMDS adapters
- creates `UserContext` from HTTP headers
- creates `RequestContext` with:
  - transaction ID
  - dataset/category
  - locale
  - dataview
  - region attributes
- adds transaction origin, client home, user home, target stack and `THRU_DSG=TRUE`

Exposed operations:

- `getGroupLevelItems`
- `getCharacteristicValues`
- `resolveFilterValues`

DMDS calls support cache/timing/retry annotations where configured.

### 4.10 Persistence and messaging

Confirmed absent:

- JPA/JDBC entities
- repositories
- migrations
- datasource configuration
- direct fact-service code or dependency
- Kafka producer/consumer/listener implementation

The Kafka client dependency and Event Hub placeholders are available, but no application-owned messaging implementation was found.

### 4.11 Docker and deployment

The Docker build uses an internal Java 25 OEL/Corretto base image.

The runtime:

- removes Python 3.12
- applies security updates
- creates `appuser`
- runs as non-root
- copies `app.jar`
- launches `docker-entrypoint.sh`

The entrypoint:

- derives `SPRING_PROFILES_ACTIVE` from `ENVIRONMENT`
- defaults to `dev`
- defaults JVM memory to `-Xms8g -Xmx8g`
- sets Datadog/build environment properties
- sets max HTTP request-header size to 32768

GitHub Actions cover CI, PR builds, quality, Cycode, Pluto tests, AKS deployment and AKS uninstall variants.

### 4.12 Tests

Sixteen test classes cover:

- controller behavior
- service validation
- exception handling
- MVC/filter/context
- headers
- DMDS utility
- AOP annotations
- event logging
- Log4j conversion
- application startup
- Swagger and base configuration

## 5. `bi-pas-fact-service`

### 5.1 Repository shape

Production Java areas:

- application/config classes
- domain documents
- controllers
- Mongo repositories/configuration
- services
- DTOs
- exception handlers
- throttling/Hazelcast classes
- subscriptions
- AOP
- MVC/context/filter utilities
- logging
- DMDS/HTTP utilities

The repository includes Docker, AKS deployment/chart files, JMeter, Gradle wrapper, Checkstyle, and 15 GitHub Actions workflows.

### 5.2 Build

Key build details:

- Java 25
- Spring Boot 3.5.14
- Spring dependency management 1.1.7
- Springdoc 2.8.13
- Spring Web/JSON/Actuator/OAuth2/AOP
- Spring Data MongoDB
- Log4j2
- Hazelcast 5.7.0
- MongoDB driver 5.6.2
- Kafka clients 3.9.2
- OkHttp 5.3.0
- Jackson 2.21.8
- CBOR and MessagePack
- AspectJ 1.9.25
- Apache Commons
- Lombok
- JaCoCo 0.8.13
- Checkstyle 10.24.0

Private Artifactory groups include:

- BI services
- PAPI
- ESS/app services
- entitlement
- BI publishing

Important internal dependencies:

```groovy
com.niq.pas:bi-pas-domain
com.nielseniq.hz:bi-throttling-fw
com.nielsen.gps.aw:bi-services-adapter
com.nielsen.buy.commons:cs-commons-logging
com.nielsen.buy.commons:cs-commons-security
```

The fact-service current domain version is configured dynamically:

```text
release-2026-DEC-I-RELEASE.+
```

at `gradle.properties:26`.

The build uses a dynamic BI adapter version:

```text
snapshot.+ 
```

and a dynamic BI-throttling framework snapshot.

### 5.3 Runtime bootstrap

`BIPasFactServiceApplication`:

- scans `com.niq.pas`
- excludes management and servlet security auto-configuration
- loads `bi-adapter.properties`
- registers `ServicesFilter`
- registers application/event logging
- registers singleton `DMDSAdapter`
- registers singleton `BIAdapter`

The application API base is:

```text
/bi-pas-fact-service/api/v1
```

The BI package processing route is:

```text
/processBiPkg
```

### 5.4 Configuration

Primary resources:

- `application.yml`
- `application-local.yml`
- `application-dev.yml`
- `application-qa.yml`
- `application-qa-eu.yml`
- `bi-adapter.properties`
- `security.properties`
- `log4j2.xml`

MongoDB uses externally supplied values:

```yaml
spring.data.mongodb.uri
spring.data.mongodb.database
```

Important feature and operational groups:

- external entitlement URL
- BI compacting
- async executor pool
- throttling enablement
- slot percentage
- Hazelcast port/TTL
- cluster-lock lease/renewal/retry
- polling interval
- maximum status checks
- delay/backoff
- pod-dataset feature

### 5.5 HTTP controller

`BIPasFactServiceController` exposes 15 logical operations:

| Endpoint | Request | Primary result |
|---|---|---|
| `GET /{jobId}/status` | job ID path | status DTO |
| `POST /jobs/details` | list of job IDs | map of job details |
| `POST /build-report` | `SimplePayload` | complex `BIWebRequest` |
| `POST /execute/sync/bi-package` | `SimplePayload` | raw `BIPackage` |
| `GET /decode/{encodedString}` | encoded node ID | `DecodeResponse` |
| `POST /processBiPkg` | serialized `BIPackage` | `BIStatus` |
| `POST /execute/async` | `SimplePayload` | `BIPackageDto` |
| `POST /multi-status` | `List<JobIdRequest>` | status map |
| `POST /report-data` | `JobIdRequest` | unified object |
| `POST /report-data/simple` | `JobIdRequest` | simplified object |
| `POST /report-data/flat` | `JobIdRequest` | flat object |
| `POST /execute/sync` | `SimplePayload` | unified report response |
| `POST /execute/sync/flat` | `SimplePayload` | flat report response |
| `POST /validate` | `SimplePayload` | raw DMDS validation response |

Most methods are annotated with `@EventLog`.

### 5.6 Synchronous controller behavior

`/execute/sync/bi-package`:

1. calls `generateReport(simplePayload)`
2. requires non-null `BIPackage`
3. requires non-null request context
4. sets compact-model state
5. calls `CompactUtil.compactModel` when enabled
6. returns the BI package

`/execute/sync`:

1. validates unified input
2. starts/report-waits through `generateReportAsync`
3. returns unified report data

`/execute/sync/flat`:

1. validates unified input
2. calls `generateFlatReportDataFromPayload`
3. returns flat report data

### 5.7 `BIPasFactServiceImpl`

This is the central implementation.

Important methods:

- `findJobStatusById`
- `findJobDetailsById`
- `getJobsStatusByJobIds`
- `transformSimplePayloadToComplex`
- `generateReport`
- `getReportBuilderSync`
- `executeReportAsync`
- `generateReportAsync`
- `generateReportData`
- `generateReportDataSimple`
- `generateReportDataFlat`
- `generateFlatReportDataFromPayload`
- `generateReportAsyncUsingThrottling`
- `validateSelections`
- `validateUnifiedInputPayload`

It injects:

- `JobUserRepository`
- `MetadataQueryHandler`
- `HandlingErrorCodeService`
- `AppLogService`
- `BIPackageConsumerService`
- `DatasetMetadataService`
- async executor
- throttling configuration repository
- throttling worker
- event-log aspect
- singleton `BIAdapter`

Status lookup is user-scoped using authenticated user email.

Job details are assembled only after a matching user/job document exists.

### 5.8 Simple payload to BI request

`transformSimplePayloadToComplex`:

1. extracts dataset ID
2. obtains execution HTTP headers
3. validates `SimplePayload`
4. invokes domain `MetadataQueryHandler.transformSimplePayloadToComplex`
5. preserves existing `BIPasException`
6. unwraps/re-maps completion exceptions
7. converts unexpected failures into `BIPasApiErrorCode.BI_PAS_ERROR`

`getReportBuilderSync`:

1. creates a complex `BIWebRequest`
2. invokes `BIAdapter.getUnifiedReportBuilderSync`
3. rejects null response
4. copies/uses the request context

### 5.9 Async execution

The async flow begins with `executeReportAsync`:

1. validate payload
2. create `JobUserDocument`
3. persist job/user state
4. transform payload to complex request
5. set user ID and component ID
6. configure request context
7. submit or queue BI execution
8. return `BIPackageDto`

If throttling is enabled:

- create `BIPasAsyncDistributedContext`
- submit it to the Hazelcast-backed worker
- wait for distributed completion
- retrieve/update job state

Otherwise:

- execute directly through the BI adapter/threading infrastructure

### 5.10 Job status polling

The implementation:

- reads a `BIPackageDto`
- retrieves job status by job ID and user
- uses configurable wait time
- uses configurable maximum poll checks
- uses exponential backoff
- caps delay
- throws on timeout/over-polling
- throws or maps downstream status errors

Default values:

```text
wait: 10 seconds
maximum checks: 81
minimum/default delay: 500 ms
maximum delay: 3000 ms
```

The exact values can be overridden through environment/application properties.

### 5.11 MongoDB documents

#### `JobUserDocument`

Collection:

```text
JobUserDocument
```

Indexes:

- `(uid, jobId)`
- `(jobId, uid)`
- `(jobId, uid, transactionId)`, background index

Fields:

- job ID
- user ID
- BI package blob name
- status
- last access timestamp
- lifetime access count
- transaction ID
- transaction group ID
- access count since refresh
- document ID
- notification locale
- message
- metadata blob name
- dataset stack ID
- BISVC environment
- original `SimplePayload`
- cell count
- error code
- BI cache flag
- TTL expiry and object fields inherited from `BaseTtlExecutionDocument`

#### `BaseTtlExecutionDocument`

Provides:

- `expireAt`
- generic `object`

#### `BIPasThrottlingConfigDocument`

Collection:

```text
ThrottlingConfigDocument
```

Fields:

- config ID
- object
- product offering code
- config type
- dataset IDs

### 5.12 Mongo repositories

`JobUserRepository`:

- `getJobUserByJobIdAndUId`
- `getJobsUsersByJobIdsAndUid`
- `getJobUserByJobId`
- `findByJobIdAndUidAndTransactionId`
- `findJobStatusByJobIdAndUId`

The status lookup uses a projection of only `status` and excludes `_id`.

`BIPasThrottlingConfigRepository` provides Spring Data Mongo CRUD for throttling configuration.

Mongo infrastructure includes:

- custom Mongo client configuration
- retry reads/writes
- pool listener
- pool statistics
- Mongo health/metering components
- TTL/index configuration

### 5.13 BI package callback processing

`BIPackageConsumerServiceImpl.apiProcessBIPackage`:

1. reject null BI package
2. require job ID
3. detect origin-stack redirection
4. require user context
5. retrieve matching job/user documents
6. reject missing or ambiguous state
7. extract BI package paths and attributes
8. update blob and metadata names
9. persist dataset stack, BI environment and cache state
10. set completed/error status
11. extract cell count
12. map upstream error details
13. save and return `BIStatus`

Successful package processing requires:

- BI status success
- package blob path

Fallback behavior:

- Filters blob path can be used as blob name if package path is absent.
- Missing package path can be converted into a BI package metadata blob error.

### 5.14 Report-data conversion

There are three report-data paths:

- unified
- simple
- flat

Unified report data:

1. resolve job/document
2. check status
3. use blob and metadata blob names
4. retrieve downstream package or cache data
5. retrieve dataset refresh date
6. construct unified response metadata
7. attach fact and dimension data

Simple report data:

- reads explicit blob and metadata blob names when supplied
- avoids Mongo lookup when no persisted document is needed

Flat report data:

- resolves job/document
- calls the flat BI package consumer
- converts flat BI package rows and attributes
- simplifies metadata
- produces a flat response envelope

The flat model uses:

- `FactMetaWrapper`
- `FlatResponseDTO`
- `FactMetaWrapper` metadata and fact values
- BI flat response maps

### 5.15 Compaction

The sync BI-package endpoint reads:

```text
feature-toggle.bi-compact
```

When enabled:

1. set `RequestContext.compactModel`
2. call `CompactUtil.compactModel`
3. return the compacted package model

### 5.16 DMDS integration

`DMDSAdapterUtil`:

- acquires a singleton DMDS adapter
- creates a DMDS `UserContext`
- forwards every incoming header as an uppercase DMDS attribute
- creates `RequestContext`
- propagates target/origin/client/user stack headers
- defaults missing stack values to the fact-service region
- adds `THRU_DSG=TRUE`

Annotated operations:

- `getGroupLevelItems`
- `getCharacteristicValues`
- `resolveFilterValues`

They use:

- Spring cache
- retry annotation
- external-response timing
- key functions based on transaction/request/dataset/dataview/dimension

The cache name is:

```text
platformClientServiceCache
```

### 5.17 Hazelcast throttling

Throttling is active only when:

```yaml
fact-service.throttle.enabled=true
```

Configuration includes:

- slot percentage
- Hazelcast port
- working-map TTL
- cluster-lock lease
- cluster-lock renewal
- lock retry

Hazelcast structures:

- processing IMap
- waiting IList
- dataset metadata cache
- dimension-as-data-item cache
- BI web-request cache
- last-minute BI request cache
- working map
- cluster lock map

The distributed flow:

1. create `BIPasAsyncDistributedContext`
2. enqueue or admit by user processor
3. place active work in processing map
4. place waiting work in waiting list
5. acquire distributed TTL lock
6. execute BI request
7. emit listener/update event
8. update job status/document
9. expose statistics

Administrative endpoints can:

- inspect throttling statistics
- inspect Hazelcast cluster info
- clear the processing map
- remove one processing-map key
- clear the waiting list

### 5.18 SSE subscriptions

`JobStatusBroadcaster` maintains a `CopyOnWriteArrayList` of subscribers:

- each subscriber is user-scoped
- SSE timeout is unlimited
- completion removes the subscriber
- broadcasting sends `JobUpdateEvent`
- failed/emitter errors remove the subscriber

No direct controller route for SSE was found in the primary controller.

### 5.19 Warmup

`WarmupInitializer` runs startup warmup:

- validates BI cache service/blob behavior
- instantiates/checks RestTemplate
- forces Jackson converter/module initialization

This addresses startup or first-request performance/classloading concerns.

### 5.20 Request context and security

`ServicesFilter`:

- buffers the HTTP body
- extracts and normalizes headers
- initializes `ExecutionContext`
- validates token presence
- validates transaction ID
- initializes application logging
- logs request body and non-sensitive headers
- resolves exceptions
- merges external timing
- stops stopwatch
- clears context

Sensitive headers are excluded from normal header logging:

- NIQ access token
- NIQ client token
- NIQ ID token
- NIQ impersonation token

The implementation sanitizes:

- job IDs
- URI/query strings
- request DTO objects
- debug values

`HeaderContextUtil` extracts:

- NIQ user
- email
- organization
- country
- transaction/origin/target stack IDs
- async header requirements

### 5.21 Exception handling

The controller advice handles:

- `BIPasException`
- `BIException`
- `JsonProcessingException`
- `IOException`
- `MethodArgumentNotValidException`
- `NullPointerException`
- Hazelcast unavailable/timeout/general exceptions
- `IllegalStateException`
- generic `RuntimeException`

Error responses use `BIPasResponseDTO` and structured `BIPasErrorhandler`.

Hazelcast:

- inactive instance: 503
- operation timeout: 504
- general Hazelcast failure: 503

### 5.22 AOP

`@EventLog` instruments report/build/execution/data endpoints.

`EventLogInterceptorAspect` records:

- method entry and return
- external status
- request context
- BI package cell counts
- timing and debug values

`@ExternalApiResponseTime` records downstream-call timing.

`@DmdsRetry` declares retry behavior for DMDS calls.

### 5.23 Docker and deployment

Docker build:

- build stage uses Java 25 OEL/Corretto
- Artifactory credentials are mounted through BuildKit secrets
- runs Gradle build
- copies `app.jar`
- runtime stage applies security updates
- creates non-root `appuser`
- copies the application
- starts `docker-entrypoint.sh`

Entrypoint:

- derives Spring profile from `ENVIRONMENT`
- defaults to `dev`
- defaults JVM memory to `-Xms8g -Xmx8g`
- sets request-header max size to 32768
- passes Datadog/build version properties

AKS chart and workflow files cover:

- QA and production deployment
- production/non-production CNAME
- install/uninstall
- performance testing
- Pluto tests
- Cycode remediation
- PR builds/dashboards
- code quality

### 5.24 Tests

The fact-service test suite contains 39 Java test classes, covering:

- application startup/configuration
- async executor
- Hazelcast configuration
- Swagger
- AOP
- MVC/wrapper/context/filter/XSS
- Mongo client/pool/TTL
- both controllers
- exception handlers
- throttling event/factory/worker/statistics
- package consumer
- fact service main, continuation and extended flows
- dataset metadata
- error-code handling
- warmup
- BI/DMDS utilities
- logging
- payload validation

Gradle test configuration:

- JUnit Platform
- one fork per test class
- 512 MB minimum heap
- 2 GB maximum heap
- 512 MB metaspace
- Hazelcast phone-home disabled
- network join disabled
- JUnit XML/HTML reports
- JaCoCo reports
- Checkstyle with failures enabled

Coverage HTML and XML reports are already generated under `build/` in the working checkout.

## 6. Cross-repository dependency matrix

| Dependency | Domain | Fact | Dimension |
|---|---|---|---|
| `bi-pas-domain` | itself | direct implementation dependency | direct implementation dependency |
| `bi-services-adapter` | api/direct | direct implementation dependency | indirect/domain/shared |
| `gps-domain` | implementation | indirect/transitive | indirect/platform |
| `DMDSAdapter` | internal adapter client | direct singleton adapter | direct singleton adapter |
| `BIAdapter` | indirect | direct singleton adapter | used only as fallback in utility |
| MongoDB | none | direct persistence | none |
| Hazelcast | none | direct optional throttling | none |
| Kafka client | none | declared dependency | declared dependency |
| `cs-commons-security` | direct | direct | direct |
| `cs-commons-logging` | direct | direct | direct |
| Fact service | none | itself | no direct dependency |
| Dimension service | none | indirect/shared adapter domain | itself |

### Confirmed direct dependency

Both executable services import `bi-pas-domain`.

The fact service imports:

- domain DTOs
- domain documents
- domain services
- shared REST, logging and node utilities
- shared exceptions and response types

The dimension service imports:

- `MetadataQuery`
- `MetadataQueryHandler`
- domain response/error types
- domain logging utilities
- domain/platform adapter types

### Not confirmed as direct dependency

There is no direct Java/Gradle dependency from dimension-service to fact-service.

Likely interactions are external/platform-level:

- both consume shared BI/DMDS infrastructure
- both consume shared domain contracts
- fact reports and dimension metadata may be combined by upstream clients
- deployment/gateway routing may direct calls to each service

No source evidence proves direct runtime RPC calls between the two services.

## 7. Consolidated runtime flows

### 7.1 Metadata-only flow

```text
Dimension client
  -> POST /bi-pas-dim-service/api/data
  -> ServicesFilter
  -> ExecutionContext + token validation
  -> BIPasDimensionServiceImpl.queryMetadata
  -> action/field/filter validation
  -> MetadataQueryHandler
  -> DmdsRequestStrategy
  -> external DMDSAdapter
  -> CDGenericWebResponse
  -> Metadata DTO/data
```

### 7.2 Simple synchronous report flow

```text
Report client
  -> POST /execute/sync/bi-package
  -> ServicesFilter
  -> SimplePayload validation
  -> PayloadTransformer/MetadataQueryHandler
  -> complex BIWebRequest
  -> BIAdapter
  -> BIPackage
  -> optional CompactUtil
  -> HTTP response
```

### 7.3 Unified synchronous report flow

```text
Report client
  -> POST /execute/sync
  -> payload validation
  -> create async BI execution context
  -> poll job status
  -> completed BIPackage
  -> blob/metadata lookup
  -> BI flat/unified response assembly
  -> BIPasResponseDTO
```

### 7.4 Asynchronous report flow

```text
Report client
  -> POST /execute/async
  -> validate payload
  -> create JobUserDocument
  -> persist in MongoDB
  -> transform payload
  -> optional Hazelcast admission/throttling
  -> BIAdapter/downstream execution
  -> BI package callback / processBiPkg
  -> update blob, metadata, status and error
  -> poll or retrieve report data
```

### 7.5 BI-package callback flow

```text
Downstream BI service
  -> POST /processBiPkg
  -> deserialize BIPackage
  -> locate JobUserDocument
  -> verify user context
  -> detect success/error
  -> persist blob names
  -> persist stack/environment/cache/cell/error metadata
  -> return BIStatus
```

## 8. Operational state and storage

The only explicitly persistent fact state in these repositories is:

- MongoDB job/user documents
- MongoDB throttling configuration
- Hazelcast distributed transient state

The domain repository has no persistence.

The dimension repository has no persistence.

The fact service has:

- MongoDB for durable/medium-life job state
- Hazelcast for distributed queue, slots and caches
- Azure/blob infrastructure through BI adapter and callback paths
- external BI/JM/DF/Unified/DMDS/CUA/MathEngine/Couch services through adapter/platform configuration

The exact list of downstream endpoints is configuration/artifact implementation detail, not fully present in repository source.

## 9. Error handling and failure surfaces

### Fact service failure surfaces

- Missing authentication or transaction ID
- Invalid payload
- Invalid dimension/filter selection
- Missing job or unauthorized job lookup
- Null BI package
- Missing request context
- Missing package/metadata blob
- BI execution timeout
- BI package callback missing user context
- no matching JobUserDocument
- upstream BI error code/message
- MongoDB access failure
- Hazelcast inactive/timeout/general failure
- adapter/REST/I/O failure
- XSS/JSON deserialization failure

### Dimension service failure surfaces

- missing transaction ID
- invalid security token
- null payload
- missing action
- missing dataset/dataview/dimension/node
- missing/invalid filters/operators/values/join/data-item type
- malformed JSON
- BI/DMDS adapter exceptions
- missing downstream metadata
- shared handler/strategy failures

### Domain failure surfaces

- malformed node ID
- unsupported/unknown endpoint
- missing selection builder
- unsupported external filter shape
- reflection/filter assignment failure
- HTTP/connection timeout
- malformed downstream response
- payload/selection branch unsupported or inconsistent

## 10. Conventions and design patterns

Confirmed patterns:

- Strategy pattern: endpoint-specific DMDS request strategies
- Generic adapter: `GenericDmdsStrategy`
- Factory/resolver: selection builders
- Template/base strategy: abstract DMDS request strategy
- Adapter/facade: DMDS and BI utilities
- Thin controller: controllers delegate to interfaces/services
- Repository pattern: Spring Data Mongo
- Common response envelope: `BIPasResponseDTO`
- Context propagation: thread-local execution context
- Filter: servlet entry point and request buffering
- AOP: event logging, retries and timing
- Distributed throttling: Hazelcast maps/lists/locks
- Executor: thread pool and caller-runs policy
- Warmup: cache/rest/Jackson startup initialization
- Feature toggles: BI compacting and throttling
- Externalized configuration: environment/profile/property placeholders

## 11. Testing and quality

### Domain

- Extensive unit coverage for metadata and all selection-builder branches
- JaCoCo
- Checkstyle
- JUnit Platform

### Dimension

- 16 test classes
- JUnit Platform
- JaCoCo
- 80% line-coverage verification
- Checkstyle

### Fact

- 39 test classes
- controller/service/AOP/Mongo/Hazelcast/context/utility coverage
- JUnit Platform
- JaCoCo
- Checkstyle
- test result HTML/XML reports

The fact and dimension services use broad test configuration but do not expose integration-test infrastructure for every external downstream dependency.

## 12. CI/CD and release artifacts

Shared characteristics:

- internal Artifactory dependencies
- GitHub Actions PR/build/quality workflows
- Cycode remediation
- Pluto testing
- AKS deployment workflows
- Dockerized non-root runtime
- Spring profile environment selection

Domain publishing:

- Maven JAR publication
- no executable boot JAR

Dimension/fact:

- executable `app.jar`
- Docker build and runtime entrypoint
- 8 GB default container JVM values in the dimension/fact entrypoints
- 32768-byte HTTP request-header maximum

## 13. Missed details and limitations

The most important details that were easy to miss:

1. `bi-pas-domain` is a library and cannot be diagnosed as a standalone HTTP service.
2. `MetadataQueryHandler` owns much more behavior than the dimension controller.
3. External DMDS request filter assignment uses reflection and can fail open.
4. The fact service stores original `SimplePayload` inside `JobUserDocument`.
5. Job lookup is user-scoped, not merely job-ID-scoped.
6. The fact service uses MongoDB for execution state and Hazelcast for transient distributed work.
7. The BI package callback is a second entry point for finalizing asynchronous jobs.
8. The `/execute/sync` route does not simply perform a synchronous BI adapter call; it validates and executes through the async/job/report pipeline.
9. Flat report conversion is a separate response/data assembly model, not just a serializing format.
10. BI compacting is feature-controlled and applied to the BI package result context.
11. The dimension service has only one implemented route; `/dataviews` is a constant only.
12. The dimension service does not provide an application-local Spring `SecurityFilterChain`.
13. Kafka dependencies exist in dimension/fact builds, but no repository-owned listener/producer was found.
14. The fact and dimension services share infrastructure, but source evidence does not prove direct service-to-service HTTP calls.
15. Dynamic domain/adaptor/throttling versions mean behavior can drift even without service source changes.
16. Local fact and dimension source code may differ from private Artifactory/transitive adapter implementation behavior.

## 14. Recommended follow-up documentation points

To maintain these repositories, document:

- Exact supported `SimplePayload` JSON schema
- Exact `MetadataQuery.action` and filter schema
- Relationship of domain artifact versions to fact/dimension snapshots
- BI package callback contract
- Mongo collection TTL policy
- Hazelcast cluster topology and slot sizing
- Job status state machine
- Flat/unified report data schemas
- Region/stack header behavior
- Upstream DMDS/BI/JM/DF/CUA/MathEngine/Unified endpoints
- Entitlement/security token contract
- Supported Kafka/Event Hub behavior, if any
- Retry and timeout ownership boundaries
- Cache invalidation and warmup policy

## 15. Source map

### Domain

- [`DmdsStrategyConfig`](https://github.com/niq-enterprise/bi-pas-domain/tree/main/src/main/java/com/niq/pas/config)
- [`MetadataQuery`](https://github.com/niq-enterprise/bi-pas-domain/tree/main/src/main/java/com/niq/pas/dto)
- [`MetadataQueryHandler`](https://github.com/niq-enterprise/bi-pas-domain/tree/main/src/main/java/com/niq/pas/service)
- [`PayloadTransformer`](https://github.com/niq-enterprise/bi-pas-domain/tree/main/src/main/java/com/niq/pas/service)
- [`SelectionBuilder`](https://github.com/niq-enterprise/bi-pas-domain/tree/main/src/main/java/com/niq/pas/service)
- [`NodeIdUtil`](https://github.com/niq-enterprise/bi-pas-domain/tree/main/src/main/java/com/niq/pas/util)
- [`build.gradle`](https://github.com/niq-enterprise/bi-pas-domain/blob/main/build.gradle)

### Dimension

- [`BIPasDimensionServiceApplication`](https://github.com/niq-enterprise/bi-pas-dimension-service/blob/main/src/main/java/com/niq/pas/BIPasDimensionServiceApplication.java)
- [`BIPasDimensionServiceController`](https://github.com/niq-enterprise/bi-pas-dimension-service/blob/main/src/main/java/com/niq/pas/dimservice/controllers/BIPasDimensionServiceController.java)
- [`BIPasDimensionServiceImpl`](https://github.com/niq-enterprise/bi-pas-dimension-service/blob/main/src/main/java/com/niq/pas/dimservice/service/BIPasDimensionServiceImpl.java)
- [`DMDSAdapterUtil`](https://github.com/niq-enterprise/bi-pas-dimension-service/blob/main/src/main/java/com/niq/pas/dimservice/util/DMDSAdapterUtil.java)
- [`ServicesFilter`](https://github.com/niq-enterprise/bi-pas-dimension-service/blob/main/src/main/java/com/niq/pas/dimservice/mvc/ServicesFilter.java)
- [`build.gradle`](https://github.com/niq-enterprise/bi-pas-dimension-service/blob/main/build.gradle)

### Fact

- [`BIPasFactServiceApplication`](src/main/java/com/niq/pas/BIPasFactServiceApplication.java)
- [`BIPasFactServiceController`](src/main/java/com/niq/pas/factservice/controller/BIPasFactServiceController.java)
- [`BIPasFactServiceImpl`](src/main/java/com/niq/pas/factservice/service/BIPasFactServiceImpl.java)
- [`BIPackageConsumerServiceImpl`](src/main/java/com/niq/pas/factservice/service/BIPackageConsumerServiceImpl.java)
- [`JobUserDocument`](src/main/java/com/niq/pas/domain/JobUserDocument.java)
- [`JobUserRepository`](src/main/java/com/niq/pas/factservice/dao/repository/JobUserRepository.java)
- [`BIPasFactServiceHazelcastConfig`](src/main/java/com/niq/pas/BIPasFactServiceHazelcastConfig.java)
- [`DMDSAdapterUtil`](src/main/java/com/niq/pas/factservice/util/DMDSAdapterUtil.java)
- [`gradle.properties`](gradle.properties)

## 16. Commit-history analysis

This section adds the Git history to the repository analysis. It was derived from:

- the local fact-service checkout and all of its reachable local/remote refs;
- read-only clones of `bi-pas-domain` and `bi-pas-dimension-service`;
- GitHub commit metadata;
- commit statistics and representative diffs.

All three repositories use the GitHub automation bot for their repository/bootstrap commit, followed by human setup commits.

### 16.1 Contributor identity

The GitHub account is:

- login: `Jagaadhep-UK_niq`
- display name: `Jagaadhep U K`
- observed commit identities:
  - `Jagaadhep U K <jagaadhep.uk@nielseniq.com>`
  - `Jagaadhep U K <Jagaadhep.UK@nielseniq.com>`
  - `Jagaadhep-UK_niq <Jagaadhep-UK_niq@users.noreply.github.com>`

The author counts below include commits across all reachable local/remote refs, including feature branches, merges, recovery branches and deploy/checkpoint commits. They are not a count of unique commits merged into `main`.

| Repository | Reachable commits | Commits by Jagaadhep identities | First contributor commit | Latest contributor commit |
|---|---:|---:|---|---|
| fact | 4,604 | 412 | 2026-03-25 | 2026-11-20 |
| domain | 894 | 144 | 2026-02-10 | 2026-11-20 |
| dimension | 2,318 | 216 | 2026-03-23 | 2026-11-20 |

Estimated numstat volumes are:

| Repository | Insertions | Deletions | Net insertions |
|---|---:|---:|---:|
| fact | 106,503 | 67,286 | 39,217 |
| domain | 20,267 | 7,641 | 12,626 |
| dimension | 31,474 | 25,673 | 5,801 |

These estimates include generated JaCoCo HTML, merge results and build artifacts when they occurred in commit history. They are useful for scale, but not a measure of maintainable source-only contribution.

### 16.2 Initial history

#### Fact

The first fact commit is `d7edcf8eea466a3cff87a77ffa9bfd3afc094f68`, created by `niq-ccoe-gh-ops-automation[bot]` on 2025-09-24:

- created only `README.md`
- added one heading line

The first human setup commit is `ada6afe50e213ccdc2e76760ce735bfd318d7871` from Vijayakumar Rajendran:

- initialized the Gradle/Spring project
- added wrapper and build files
- added `BIPasFactServiceApplication`
- added the initial application test
- added `.gitignore` and `.gitattributes`
- amounted to 459 insertions

The next phase was Vijayakumar establishing the Spring naming/application configuration and Sujit Pradhan establishing project upgrades, Checkstyle, Jacoco, Swagger, repositories and initial fact service/domain classes.

#### Domain

The initial domain commit is `389c6c38aceada4c055fdc72925724b955a0cbcc`, created by the GitHub automation bot on 2025-09-25:

- created `README.md`
- contained no domain implementation

The first human setup commit is `0cad058db6a122692cb582932e877d0120f01232`:

- initialized the Gradle/Spring library/application skeleton
- added wrapper/build/settings files
- added `BIPasDomainApplication`
- added the initial test
- amounted to 457 insertions

After setup, the domain repository was developed in stages:

1. workflow and publishing setup;
2. Spring/Gradle dependency upgrades;
3. logging and test infrastructure;
4. Apache Commons and HTTP client utilities;
5. REST invocation helpers;
6. metadata/enrichment/payload/selection implementation;
7. the domain artifact consumed by fact and dimension.

#### Dimension

The initial dimension commit is `bf375a4c36571a9cfa3222f9bd3c2ab26b8b2652`, created by the GitHub automation bot on 2025-09-24:

- created `README.md`
- contained no service implementation

The first human setup commit is `8b90a2eb9bfdc4a6d455cc6fbda48133660ad409`:

- initialized Gradle/Spring
- added wrapper/build/settings files
- added `BIPasDimensionServiceApplication`
- added initial tests
- amounted to 459 insertions

The service was subsequently expanded through configuration upgrades, Mongo/VMDB/BI wiring, MVC context, dimension/metadata implementation, UI assets, Docker and CI/CD.

### 16.3 Jagaadhep's fact-service history

The first visible fact-service commits attributed to Jagaadhep are from March 2026.

#### March-April 2026: logging, sync acceleration and report rendering

Key commit `9c685d7094ab92a957450cae6c0f93b016583982`:

- `fix(RBOC-196657): add report generation ended log with dataset id`
- changed `BIPasFactServiceImpl`
- added dataset-ID metadata to report-generation completion logging
- updated `BIPasFactServiceImplTest`

Key commits `c569f7450b9eab969105274ed2c41d6f75f3411f` and `077f7f6a30d892d3c51927eb6bb6cbcdb27a5dfa`:

- added payload transformation start/processing logging
- added test assertions
- established observability around transformation time and payload dataset context

Context provider work from `1751fe288ce10a5c22a1a82b649d3eac178fe29c` and follow-up Copilot-suggestion commits established:

- a thread-local/current-execution context implementation
- user, transaction and header-aware request handling
- application logs with transaction context

Key commit `fb03c3aae31f83207e72222cd5de65b904b71e80`:

- `accelerate execute sync with local wait fallback`
- added `SyncStatusWaitRegistry`
- added a same-replica `CompletableFuture` wait mechanism
- retained MongoDB lookup as cross-replica fallback
- added status-projection lookup in `JobUserRepository`
- added tests for wait registration and status waiting

The implementation is important:

```text
same replica/callback:
  register job waiter
  BI callback completes local waiter
  await immediate completion signal

cross replica/restart:
  wait locally for a bounded period
  fall back to Mongo JobUserDocument status lookup
```

The registry uses status-plus-job/UID/transaction wait keys and suppresses/clears waiters after completion.

Key commit `b8583740084542d0b0a1c61ea02b1ef774afcc76`:

- added an HTML report-table endpoint
- added `BIPackageRenderUtil`
- decompacts BI packages through external BI service `/util/bipkg/decompact`
- renders the decompacted package through `/util/bipkg/html`
- forwards request-scoped headers but removes host and content-length
- sets JSON request content and text/JSON accept headers
- added service/controller tests

#### April-May 2026: tooling and DTO introspection

Key commit `953827f70fa9f54e74227bd14ac559ba36d0b1e2` added:

- `POST /utility/dto/fields`
- `DtoIntrospectionRequest`
- `DtoIntrospectionUtilityService`
- `FactDtoResolverAdapter`
- package allowlisting for fact DTO/domain packages
- structured DTO introspection errors

The utility restricts reflection/resolution to fact/domain package allowlists. It is a diagnosis/tooling endpoint, not a general class-loader endpoint.

This period also includes AI-adoption and SDLC-tooling commits:

- `.github/copilot-instructions.md`
- SDLC agent and instruction files
- cross-repository execution details
- scripts for log querying and LLM error categorization

#### May-July 2026: deployment, scaling and test work

The fact history shows sustained KEDA/Coralogix work:

- Coralogix triggers for US/EU dev and QA
- hotfix and production trigger changes
- HPA threshold tuning
- CPU/heap/memory validation
- US/perf, US/UAT and EU/UAT scaling thresholds
- performance JMeter workflows
- KEDA deployment workflow updates

The contributor also worked on:

- entitlement catalog URLs
- OAuth provider/mode security properties
- OpenTelemetry instrumentation annotation
- Datadog/HPA configuration
- deployment chart refactors
- Pluto/performance test assets

The RBOC-210538 share-fact-products Pluto tests were added and corrected during June.

#### July-November 2026: domain integration, payload fixes and performance

Major themes visible in the commit subjects and diffs:

- `factmulti` and multi-replica behavior
- release/dependency version management
- domain snapshot/release updates
- blob/report data fixes
- future status-poll timeout handling
- HTML/report endpoint changes
- multi-region and stack behavior
- flat-report and report-data fixes
- throttle enablement/disablement
- DTO and service validation fixes
- security and Java vulnerability fixes
- Java 25/base-image updates
- build and Checkstyle fixes

The fact repository has 62 commits touching `gradle.properties`, making dependency/domain/artifact version updates one of the largest maintenance themes.

The most changed implementation files include:

- `BIPasFactServiceImpl.java`
- `BIPackageConsumerServiceImpl.java`
- `BIPasFactServiceController.java`
- dataset metadata implementation
- fact service tests
- package consumer tests
- async continuation/extended tests

The 106,503 estimated insertions reflect feature branches and repeated merge/resolution history, not a single linear 106K-line implementation.

#### November 2026: Java vulnerability and DEC domain alignment

Commit `25a13a0d0eb2ed60fea95d155e49ae0516967fef`:

- fixed a Java vulnerability through hotfix deployment values
- changed `deployment/aks/chart/ushotfix.values.yaml`
- 3 insertions and 4 deletions

Commit `294c2f8db372fd1fadb78274bd0116137ad9756d`, the current branch tip:

- updated `bi_pas_domain_version`
- selected `release-2026-DEC-I-RELEASE.+`
- changed only `gradle.properties`

### 16.4 Jagaadhep's domain history

The first domain contributor commit is `2dba4387f2e958ff3b8ee4a231adc94e74926bf3`:

- applied Sonar changes to `DownstreamServiceClient`
- applied Sonar changes to `MetadataEnrichmentService`
- 3 insertions and 7 deletions

The major work areas are described below.

#### Metadata handler and Sonar quality

Commits in February 2026 refactor and harden:

- `MetadataQueryHandler`
- `MetadataEnrichmentService`
- `LogCommonUtil`
- JSON/config utilities
- test classes

Commit `a0669a69d17156cf9f6bed7e38aa68707c1a504a`:

- Sonar-driven refactoring
- metadata node/filter response changes
- 508-line diff in `MetadataQueryHandler`
- implemented encryption logic
- 357 insertions and 155 deletions

The metadata handler work included:

- resolving dimension ID/name
- retrieving hierarchy roots and characteristics
- caching dimension roots
- encoding/decoding nested node IDs
- encrypted versus decoded node response behavior
- filter-value response metadata
- hierarchy-characteristic order
- lower-complexity helper methods
- virtual-thread/future orchestration

#### Transaction logging

The March logging commits:

- obtain and propagate transaction IDs
- add `LoggerUtil`
- add log tags
- update all domain log call sites
- add a virtual/fork-join executor instead of fixed worker threads
- add Copilot/Sonar/QG fixes

#### ObjectMapper and thread safety

May 2026 commits refactor `ObjectMapperConfigurer`:

- avoid duplicate configuration
- tolerate unknown fields
- mark volatile fields
- add inclusion handling
- improve thread safety

These changes support reuse in concurrent metadata/report paths and reduce constructor/configuration races.

#### Refresh date and response metadata

June 2026 introduced `refresh date` work:

- added refresh-date behavior to shared response DTO data
- updated `requestContext`
- added formatting and merge fixes
- improved dataset refresh metadata

#### REST invocation and security hardening

June 2026 also added a dedicated rest-call class and routed `MetadataQueryHandler` through it.

Changes included:

- disallowed host header forwarding
- multiregion usage
- build fixes
- context/request-context refinements
- test coverage

#### Multiregion support

Commit `1dad68268a6195eef6c5ab12a7667af2724fa247` changed:

- `MetadataQueryHandler`
- `MetadataQueryHandlerTest`
- 138 insertions and 15 deletions

It was followed by later encoded selection, RETAILER_SU, compact dimension, selection type and model-ID work.

#### CompletableFuture and timeout support

Commit `dbace71bb3d618b9bedae46eca0f5cd9d4e291e9`:

- added timeouts around downstream service calls
- updated `MetadataEnrichmentService`
- updated `MetadataQueryHandler`
- added a timeout constant
- expanded enrichment and handler test coverage

Commit `2654174274e315dc6e31d3fcf8f0e12d8e43733b`:

- changed enrichment and metadata future handling
- added/expanded `CompletableFutureUtils`
- 127 insertions and 20 deletions

It introduced safer handling for parallel metadata calls and bounded downstream waits.

#### Selection and report data work

Later domain commits show:

- data-items in get-node-children for OSA periods
- encoded-expression fixes
- `ModelId` addition to simple payload
- `RETAILER_SU` shifts
- compact-dimension IDs
- selection type attributes
- item-option retrieval
- `Allothers` selection handling
- filter-value changes

These commits explain the depth of fact/dimension payload compatibility in the current domain library.

#### Build and dependency alignment

Jagaadhep also maintained:

- JDK and Spring Boot versions
- BI adapter/JFrog versions
- `gps-domain` and `bi-services-adapter` versions
- Objenesis exclusions
- test execution in JAR build workflows
- AI-adoption and SDLC support files
- encoded-expression and data-item test fixes

The latest contributor commit is `7278883d61de3745ad26770f64440d6c3c5da355`, updating:

- `gps_domain_version`
- `bi_version`

### 16.5 Jagaadhep's dimension history

The dimension first contributor commit is `bd74d063cdc02fc64055389d23d0df6eaad84c86`:

- updated `bi_pas_domain_version`
- 2026-03-23 snapshot

Key March 2026 commit `f768f15fc63a7b18bda457774b6121eb6cf2d156`:

- added `ContextProviderImpl`
- implemented shared context provider access
- added `ExecutionContext.setHttpHeaders`
- modified application startup/constants
- modified MVC wrappers, exception handling and logging
- added 198 insertions and 33 deletions
- updated application/context/service tests

This commit established a consistent thread-local context contract equivalent to the fact/domain context model:

```text
incoming HTTP headers
  -> ExecutionContext
  -> ContextProviderImpl
  -> MetadataQueryHandler/downstream calls
```

Key commit `d40822f9212ffaee03318702f34bdc9c22aff336` fixed QG issues in:

- buffered request wrapper tests
- execution context tests

#### DMDS/BI wiring proof

Commit `a98788a93a7f2a088b4b492cc3501858f837be95` is one of the most substantial implementation commits:

- added DMDS retry/time annotations
- added action constants
- extended controller/service implementation
- added `DimensionResponse`
- added `DMDSAdapterUtil`
- added 169 lines of DMDS utility implementation
- added 506 lines of DMDS utility tests
- added 183 lines of retry tests
- added 190 lines of timing tests
- fixed Swagger, exception and context tests

It is a major proof of Jagaadhep's work on:

- BI/DMDS adapter wiring
- request context propagation
- retry/timing aspects
- DMDS group/characteristic operations
- multi-region headers
- test and checkstyle quality

#### UI and POC work

Commit history also contains dimension UI/POC artifacts:

- `app.py`
- static JavaScript builders
- Flask/HTML templates
- payload construction for PPL, hierarchy, characteristics and conditions
- navigation and modal UI code

The most touched UI paths include:

- `static/js/ppl/group-modal.js`
- `static/js/payload/ppl-build.js`
- `static/js/app.js`
- `static/js/builder/navigation.js`
- `static/js/builder/conditions.js`
- `static/js/builder/hierarchy.js`
- `static/js/builder/characteristics.js`

These indicate proof-of-concept/report-builder tooling rather than the Java Spring controller itself.

#### Deployment and base-image work

Dimension commits show:

- Docker multi-stage build creation
- non-root runtime setup
- Java 25/OEL/Corretto base-image upgrades
- JVM and profile entrypoint updates
- CI/UI workflow changes
- AKS chart updates
- Java vulnerability fixes
- Cycode/SAST fixes
- KEDA deployment
- CNAME and environment release branches

Commits `Fixed cycode violations`, `Cycode fix`, and `Cycode SAST fix` appear repeatedly, so security remediation is a distinct long-running workstream.

The latest sampled merge `5a4b1b73b583936cff8c283918a5232be9282e9a` updated:

- `Dockerfile`
- `gradle.properties`

It merged the December feature branch into the dimension repository.

### 16.6 Cross-repository chronology

The history shows a coordinated implementation sequence:

```text
2026-02 domain
  - Sonar and MetadataQueryHandler work
  - first domain implementation of Jagaadhep's work

2026-03 fact/dimension
  - ContextProvider and ExecutionContext work
  - payload/report logging
  - metadata and DMDS wiring

2026-04
  - sync-status acceleration
  - HTML BI-package rendering
  - DTO introspection
  - AI-adoption/SDLC tooling

2026-05
  - ObjectMapper thread-safety
  - KEDA/HPA/Coralogix scaling
  - OpenTelemetry/security deployment

2026-06
  - refresh-date support
  - REST invocation refactors
  - host-header hardening
  - Pluto test assets

2026-07
  - multiregion support
  - encoded selections and data-item fixes

2026-08
  - JDK/Spring/BI version alignment
  - agentic workflows

2026-09
  - OSA data-items
  - encoded-expression fixes
  - CI/test fixes

2026-10
  - RETAILER_SU and compact-dimension support
  - selection type/model ID changes
  - CompletableFuture timeout hardening

2026-11
  - Java vulnerability fixes
  - Java 25/base-image updates
  - December domain snapshot/release alignment
```

### 16.7 Repeated merge and automation patterns

The history contains many commits that are not independent implementation units:

- merge-branch commits;
- merge pull requests;
- remote-tracking branch merges;
- deploy commits from Jenkins/bots;
- GitHub Actions deployment-tag updates;
- Copilot checkpoint/WIP commits;
- version-only commits;
- branch/recovery commits;
- agent-host baseline/checkpoint commits.

The fact author count is therefore not a clean count of 412 independently reviewed source changes. Representative implementation commits should be distinguished from:

- deployment metadata,
- generated reports,
- merge resolution,
- dependency-only bumps,
- bot checkpoints,
- repeated feature-branch synchronization.

### 16.8 History conclusions

The clearest sustained Jagaadhep workstreams are:

1. **Metadata and payload correctness**
   - `MetadataQueryHandler`
   - node encoding/decoding
   - selection/filter handling
   - refresh-date and multiregion behavior

2. **Observability**
   - transaction-aware logging
   - dataset/report payload logs
   - external-call timing
   - Coralogix and deployment metrics

3. **Sync and async reliability**
   - same-replica wait acceleration
   - Mongo fallback
   - CompletableFuture timeout behavior
   - status-polling and callback processing

4. **DMDS/BI adapter integration**
   - context providers
   - retry/timing annotations
   - DMDS utility operations
   - report rendering and DTO tooling

5. **Deployment and security**
   - Java 25/JDK upgrades
   - Docker/base image updates
   - Cycode/SAST remediation
   - AKS/HPA/KEDA adjustments
   - Java vulnerability hotfixes

6. **Test and quality**
   - Sonar and QG fixes
   - Checkstyle fixes
   - unit test expansion
   - JaCoCo coverage artifacts
   - JMeter/Pluto/performance assets

7. **Agentic and SDLC tooling**
   - Copilot instructions
   - SDLC agent/skill files
   - cross-repository documentation
   - log query and categorization scripts

## 17. Requested branch deep-dive

The following branches were fetched and compared against the repository's remote `main` and merge bases.

### 17.1 Branch existence summary

| Repository | Branch | Status |
|---|---|---|
| fact | `release/keda-deploy-cname` | Exists |
| fact | `feature/ai-adoption` | Exists |
| fact | `feature/ui` | Not present |
| dimension | `release/keda-deploy-cname` | Exists |
| dimension | `feature/ui` | Exists |
| dimension | `feature/ai-adoption` | Exists |
| domain | `feature/ai-adoption` | Exists |

Fact's remote branch list contained no `feature/ui` branch.

### 17.2 Fact: `release/keda-deploy-cname`

Observed tip:

```text
73f1663a4f2e6af10dbca2f09af6ba292fcd6b0e
Updated deploy tag '2.111.13'
for environments: ushotfix
2026-11-19 14:35:30 UTC
```

The branch is highly divergent from current `main`:

- merge base: `bf13e9f90aaefeeb0f04f60d44f86da7b3eb4877`
- commits in `main` only: 963
- commits in release branch only: 1,778

This is not a small release cut. It is a long-lived, heavily updated branch that diverged early and contains 1,778 branch-only commits.

The diff against `main` shows extensive AKS, CNAME, environment-values, workflow and source/dependency changes. It includes some source deletions, which indicates that the release branch is not a simple additive layer over `main`; it has used older snapshots and branch-specific restructuring.

The branch is primarily deployment/CNAME/KEDA work rather than a clean application-source feature:

- US/EU/US Hotfix/US Perf/US UAT/EU UAT/EU Perf value files
- CNAME-specific value files
- AKS deploy, CNAME and uninstall workflows
- environment tag updates
- CNAME URLs and service aliases
- Docker entrypoint/profile behavior
- MongoDB secret/configuration fixes
- DMS/DMDS/PAS service endpoint configuration
- `check_url.sh` CNAME health checks
- KEDA/HPA/Coralogix configuration
- entitlement, Unified and DMDS gateway URLs
- OpenTelemetry Java annotation/configuration
- Java vulnerability hotfix values
- Artifactory credential and CI runner changes

Representative Jagaadhep commits on this branch include:

- `791d3f40` — update `niq-deploy-ab` dependency to `4.0.8`
- `6f70e52f` — Coralogix trigger for US Dev
- `24200527` — Coralogix triggers for EU QA, US QA and US UAT
- `d3221e55` — US Dev heap validation and CPU tuning
- `c8ea0ee2` — lower scaling threshold from 10 to 4
- `9682e925` — UAT scaling thresholds
- `17e8b08d` / `2f21efff` / `6f7074fa` — Perf, US Perf and EU UAT migration
- `a2516211` — rebase/correct trigger thresholds against release values
- `80baa674` — DMDS REST gateway endpoint
- `f59f5e62`, `f8d3a974`, `b835d0b9` — entitlement URL updates
- `2f3939e6`, `748549e7`, `c902417d` — Unified REST URL values
- `d87320e7`, `1b46a99f`, `e0c0e6e2` — OpenTelemetry removal/addition/annotation changes
- `46b713a1` / `25a13a0d` — Java vulnerability fixes

The KEDA configuration themes are:

- workload CPU/HPA scaling thresholds
- HPA minimum/maximum replica behavior
- Coralogix metric triggers
- US/EU dev, QA, UAT, Perf, Prod and Hotfix values
- throttling/KEDA interaction
- JVM memory validation
- Datadog/OpenTelemetry settings
- environment-specific gateway and catalog endpoints

Its large divergence and source deletion should prevent it from being treated as an exact source-of-truth for the current fact service implementation. It is better understood as a deployment/release branch snapshot with substantial environment configuration.

### 17.3 Fact: `feature/ai-adoption`

Observed tip:

```text
58bf9e9b83345798dfca6df8cad40c4ae4124cd7
Skills addition
2026-08-24 16:13:26 +05:30
```

Comparison against `main`:

- merge base: `70552ee1e96d3bcffbab0a0cc456036c362a7869`
- commits in `main` only: 195
- commits in AI branch only: 13
- diff: 64 files
- diff additions: 9,890

It is primarily an agentic/SDLC tooling branch. The branch does not introduce the fact-service runtime architecture itself; it adds repository knowledge and automation.

Representative unique commits:

- `17148072` — payload transform start app logging
- `fa621120` — AI-adoption agent, instructions and skills
- `4d3f70fa` — structure/refine instructions
- `9ecdce8e` — instructions directory structure
- `4a30771e` — refine step files
- `8188b215` — merge current `main`
- `8e288639` — cross-repository execution details and refined Markdown
- `90e15afa` — update SDLC files
- `873bd153` — merge current `main`
- `4bc6b56e` — merge remote feature copy
- `c55fad1c` — Coralogix additions
- `09a36ba7` — agentic workflow
- `58bf9e9b` — skills additions

The branch diff adds:

- `.github/copilot-instructions.md`
- `.github/agents/sdlc.agent.md`
- specialized agents for triage, implementation, review, documentation and Coralogix
- SDLC step instructions 0 through 10
- reference files for:
  - architecture
  - runtime flows
  - BI-PAS integration
  - cross-repository impact
  - persistence
  - throttling/Hazelcast
  - observability
  - security context
  - testing
  - troubleshooting
  - workflow authority
- skills for:
  - feature implementation
  - API endpoint creation
  - tests
  - trace flows
  - cross-repo impact
  - SDLC routing
  - Coralogix queries
  - implementation safety

The detailed integration report files are based on the fact/domain/dimension architecture and were explicitly designed for cross-repository agent execution.

### 17.4 Dimension: `release/keda-deploy-cname`

Observed tip:

```text
c38d48c4eda0952b68b3181c5e9a6c9d7f54cb
Updated deploy tag '2.121.3'
for environments: usqa
2026-11-20 15:54:13 UTC
```

Comparison against `main`:

- merge base: `ffad03d37e29199e6b00f329d11d2a354bf67d78`
- commits in `main` only: 382
- commits in release branch only: 1,113

The branch is also a deployment/environment branch with extensive AKS and CNAME history.

Key Jagaadhep work includes:

- `160ea3c0` / `05373579` / `a8526327` — DMDS REST base URL for US Dev, EU QA and US QA
- `e58b0ba2` / `952b620a` / `d0ab98d1` — entitlement catalog URL updates
- `3152297b` — dimension-root cache toggle
- merge resolutions with `feature/keda-deploy-new`
- `1c75bbaa` / `69ab22a4` — release-branch synchronization
- OpenTelemetry Java instrumentation and injection annotation changes
- EU Perf/US Perf/US Hotfix/EU Prod/US UAT configuration refactors
- CI runner changes
- deletion of CI/CI-UI workflows and later reintroduction through merges
- Java vulnerability fixes
- final merge to release/KEDA deployment state

The dimension release branch contains source and deployment snapshots, but it is dominated by:

- `deployment/aks/chart/*.values.yaml`
- CNAME value overlays
- AKS deploy/uninstall/CNAME workflows
- Docker and JVM startup
- OpenTelemetry configuration
- environment tags
- DMDS and entitlement URLs
- cache settings
- Java and Cycode remediation

Like the fact release branch, it should not be treated as a linear current-source release. It is a deeply divergent environment/release state.

### 17.5 Dimension: `feature/ui`

Observed tip:

```text
74b0d3a89f90fe34bbe2dc82355dbcd84b6ef438
Added pgkeys in simple payload to support fact service
2026-11-10 16:06:47 +05:30
```

Comparison against `main`:

- merge base: `878a561ba1b53d480bfd3e9cbf5f0755489d9ad5`
- commits in `main` only: 126
- commits in UI branch only: 70

This branch is a POC/report-builder UI combined with deployment and cross-service integration work. It is not just a static frontend.

Major work phases:

#### POC UI foundation

- added transaction ID handling
- added Bought and Shopped At PPL behavior
- created `ci_ui.yml`
- added multi-stage Dockerfile
- renamed POC UI images and services
- moved UI code under `src`
- added deployment files
- added US QA as a default fact/dimension target
- added connection timeout and increased response timeout

#### UI payload builder

The UI branch contains static JavaScript and templates for:

- PPL and group modal behavior
- Bought and Shopped At
- Deal and loyalty selections
- quantity
- OSA and retailer conditions
- buyer/shopper exclusivity
- characteristic and hierarchy builders
- condition and operator rendering
- folder/all-options selection
- cascading selection fixes
- selection preview
- item/data-item rendering
- Product Group widgets
- `pgKeys` context propagation

The committed features include:

- Walmart price-beats UI fixes
- retailer POD condition fixes
- greater-than/equal operator support
- group selection bug fixes
- data-item handling
- characteristic-only type behavior
- dimension labels with dimension IDs
- no-data response rendering
- no-data fact-service handling

This directly explains the fact/dimension report payload interoperability:

```text
UI payload -> pgKeys/modelId/dimension data
           -> dimension metadata API
           -> fact SimplePayload
           -> downstream BI report
```

#### Deployment and runtime

The branch includes:

- UI Docker build
- CI/UI workflow
- US Dev value files
- fact/dimension US QA target endpoints
- image/tag changes
- Azure runner changes
- fetch-inputs/job runner changes
- KEDA/CNAME release synchronization

Representative commits:

- `2f9e4bee` — transaction IDs
- `78076f97` — Bought and Shopped At PPL
- `5828c9b9` / `793aa3cb` — UI CI/Docker foundation
- `991cbbad` — US QA endpoints
- `67886eb9` / `991f3ffc` — loyalty
- `50884460` / `12eaf373` — Walmart fixes
- `6a0c58e1` — buyer/shopper exclusivity
- `94ef3ea0` — data items and conditions
- `b5c1759a` — folder/all/cascading fixes
- `6f790b9f` — Product Group widgets
- `b0ef7ed1` — `pgKeys` in dimension calls
- `74b0d3a8` — `pgKeys` in fact `SimplePayload`

The UI branch is currently 70 branch-only commits ahead of the merge base and diverges substantially from `main`, so it should be treated as a POC branch rather than a guaranteed production source snapshot.

### 17.6 Dimension: `feature/ai-adoption`

Observed tip:

```text
d84843c026690a8c983d053c3aab8a9ac6376b0f
Added details in .github md files
2026-10-09 16:23:08 +05:30
```

Comparison against `main`:

- merge base: `fc8408acbddd1e93574b4d37667c6b65db3b260c`
- commits in `main` only: 24
- commits in AI branch only: 10

Unique non-merge commits:

- `b2469455` — AI SDLC files
- `528e92e8` — updated SDLC files
- `354897bc` — merge current `main`
- `1a83c5e7` — merge feature branch
- `f04cf827` — Coralogix addition
- `2fcf0616` — agentic workflow
- `4f0c24da` — skills addition
- `df420f72` — Cycode fix
- `8c3ebaa9` — merge current `main`
- `d84843c0` — `.github` Markdown details

This branch adds:

- AI SDLC agents
- Copilot instructions
- workflow/step definitions
- cross-repository context
- Coralogix query/workflow support
- security-remediation fixes

It is less runtime-specific than the UI branch and less deployment-specific than the KEDA branch.

### 17.7 Domain: `feature/ai-adoption`

Observed tip:

```text
663dbac7c093415b75abdaa707f47d786dfdd309
Skills addition
2026-08-24 16:11:28 +05:30
```

Comparison against `main`:

- merge base: `a76c37bcfefaceacba5b97b8d031903364ae277c`
- commits in `main` only: 203
- commits in AI branch only: 7

Unique commits:

- `3e9f6644` — Markdown AI-SDLC files with fact-service cross-repo support
- `9ec162f0` — update SDLC files
- `d5705472` — merge current `main`
- `85ce5761` — merge current `main`
- `1f528959` — Coralogix addition
- `62eafcea` — agentic workflow
- `663dbac7` — skills addition

The domain AI branch is a documentation/tooling branch:

- it references fact-service integration
- it adds reusable agent and skill instructions
- it adds Coralogix operational support
- it synchronizes with `main` twice
- it does not add separate domain runtime implementation

### 17.8 Branch relationship conclusions

#### KEDA deployment branches

Both fact and dimension have long-lived `release/keda-deploy-cname` branches with:

- environment values
- CNAME routing
- KEDA/HPA thresholds
- Coralogix triggers
- JVM/telemetry settings
- entitlement and gateway endpoints
- Java vulnerability fixes

They are deployment branches, not merely small release tags.

#### UI branch

The dimension `feature/ui` branch is the largest substantive POC UI branch found:

- it builds report payloads
- it selects PPL/deal/loyalty/exclusivity/options
- it builds dimension/fact request contexts
- it adds POC Docker/CI/AKS deployment
- it explicitly handles fact-service payload compatibility
- it includes Product Group and `pgKeys` support

No fact-service `feature/ui` branch was found.

#### AI-adoption branches

All three AI-adoption branches exist in the three repositories:

- domain: tooling/documentation branch
- fact: tooling plus payload-logging and 64-file agent/SDLC diff
- dimension: tooling plus Coralogix/security additions

The strongest branch implementation is in fact, where `.github` agent and instruction additions are extensive.

#### Important branch caveat

The branch comparisons use merge-base arithmetic. Because the requested branches are often old or deployment-oriented, commit counts and current diffs do not mean that every branch-only commit is still present, or that every commit from `main` is absent in the branch. The merge/base diffs must be interpreted as historical lineage plus current divergence, not as a linear change list.

## 18. Jira work history: Unified Services and Unified Services Suite

**Snapshot:** 2026-11-21. This section supplements the repository analysis with Jira evidence; it does not assert that every historical experiment is present in the current production code.

### 18.1 Scope and evidence

The local Jira MCP server resolved the requested email to Jira user `uja5002`. The search included issues assigned to that user during the Jira date window 2026-01-01 through 2026-11-22, plus issues reported by that user and updated since 2026-01-01. It therefore includes older issues still assigned during the period and work reported for teammates, not just issues created in 2026.

The search returned **124 issues**: 81 RBOC, 41 OPTIQ, one OMNI2, and one TESTPJ. At retrieval, 106 were currently assigned to the user and 66 were reported by the user; these groups overlap. Statuses were 99 Done, eight Accepted, six Withdrawn, five Not Started, three In Progress, and one each Created, Dev Complete and Ready for UAT. These are Jira workflow states, not proof of production deployment.

All 124 issue records were retrieved with their descriptions, fields, comments and expanded changelogs. The returned comment and changelog counts matched the totals on each record: 342 comments in all. There were 390 attachment records across 53 issues, including screenshots, JSON payloads, logs, a spreadsheet, a document and a JMeter archive. For eight selected issues, 209 attachments were downloaded and 116 JSON files parsed successfully; selected payloads and screenshots were inspected, especially for entitlement, POD, shifting and the current condition investigation. Attempts to download the other 43 ordinary-size attachment sets failed when the Jira host stopped resolving; the two sets containing very large files were not completely downloaded. **The remaining attachments, most downloaded screenshots and linked Confluence pages, recordings, PRs and workflows were not independently reviewed.** Some large attachments alone total hundreds of megabytes. Attachment filenames or comments are not treated as proof that their contents were inspected.

This is a defensible assignment/reporter-based portfolio, not an exhaustive search for every issue on which the user commented or collaborated. Jira permissions, search semantics, historical assignment coverage, future updates and issues owned entirely by others can limit completeness. Separate a person's explicit comments or linked changes from work attributed only by assignment, and treat contradictory ticket titles, descriptions, screenshots and closure states cautiously.

### 18.2 Work trajectory and service context

The user's work spans the shared domain library, fact and dimension services, the BI services adapter, the POC UI in the dimension repository, deployment configurations and cross-team operational tools. The recurring flow is: select metadata through dimension/DMDS, construct a simple request in the POC UI or client, transform it in PAS domain, validate selections/entitlements, execute a BI report through fact service, and inspect the resulting BI package, metadata and flattened response.

1. **January-February:** onboarded to dimension, fact and domain, exercised metadata queries and validated selection variants. [RBOC-188559](https://adlm.nielseniq.com/jira/browse/RBOC-188559), [RBOC-189605](https://adlm.nielseniq.com/jira/browse/RBOC-189605), [RBOC-189867](https://adlm.nielseniq.com/jira/browse/RBOC-189867).
2. **February-April:** followed execution and logging across the three repositories, investigated transaction-context propagation, supported deployment readiness, and built operational dashboards and alerts. [RBOC-180616](https://adlm.nielseniq.com/jira/browse/RBOC-180616), [RBOC-194233](https://adlm.nielseniq.com/jira/browse/RBOC-194233), [RBOC-197495](https://adlm.nielseniq.com/jira/browse/RBOC-197495).
3. **April-June:** developed AI-assisted SDLC workflows, investigated a fact-service QA failure, validated entitlement and multiregion scenarios, and added performance-test automation. [RBOC-199481](https://adlm.nielseniq.com/jira/browse/RBOC-199481), [RBOC-202351](https://adlm.nielseniq.com/jira/browse/RBOC-202351), [RBOC-201678](https://adlm.nielseniq.com/jira/browse/RBOC-201678), [RBOC-206642](https://adlm.nielseniq.com/jira/browse/RBOC-206642), [RBOC-210324](https://adlm.nielseniq.com/jira/browse/RBOC-210324).
4. **July-September:** expanded POC UI support for people/POD and conditions, compared Discover and Unified outputs, investigated template compatibility, developed AI tooling, and improved monitoring and test gates. [RBOC-217696](https://adlm.nielseniq.com/jira/browse/RBOC-217696), [OPTIQ-1065](https://adlm.nielseniq.com/jira/browse/OPTIQ-1065), [OPTIQ-553](https://adlm.nielseniq.com/jira/browse/OPTIQ-553), [OPTIQ-1733](https://adlm.nielseniq.com/jira/browse/OPTIQ-1733).
5. **October-November:** worked on shifting/leakage templates, product-group propagation, security and large-report triage; current work includes Optiq validation and a POD condition failure. [OPTIQ-710](https://adlm.nielseniq.com/jira/browse/OPTIQ-710), [OPTIQ-2533](https://adlm.nielseniq.com/jira/browse/OPTIQ-2533), [OPTIQ-2475](https://adlm.nielseniq.com/jira/browse/OPTIQ-2475), [OPTIQ-2241](https://adlm.nielseniq.com/jira/browse/OPTIQ-2241), [OPTIQ-2962](https://adlm.nielseniq.com/jira/browse/OPTIQ-2962).

### 18.3 Cross-repository investigations and implementation

**Request context and logging.** [RBOC-180616](https://adlm.nielseniq.com/jira/browse/RBOC-180616) documents adding app logs and tracing transaction IDs across domain, fact and dimension. In [RBOC-194792](https://adlm.nielseniq.com/jira/browse/RBOC-194792), asynchronous execution initially reused the first request's transaction ID; the user investigated an interface/provider and per-thread context for headers and IDs. [RBOC-194795](https://adlm.nielseniq.com/jira/browse/RBOC-194795) records corresponding tests. The parent [RBOC-194233](https://adlm.nielseniq.com/jira/browse/RBOC-194233) says the implementation approach later changed and was merged; the early prototype must not be presented as the final architecture. Later [OPTIQ-1699](https://adlm.nielseniq.com/jira/browse/OPTIQ-1699) reports removing old Datadog log-pattern/agent pieces, using the new telemetry schema and checking for duplicate and downstream logs in QA.

**QA timeout and serialization compatibility.** In [RBOC-202351](https://adlm.nielseniq.com/jira/browse/RBOC-202351), a sync timeout was traced through `processBiPkg` to an unrecognized `cmpPrdcPresentInRequest` field and a mismatch between BI adapter release versions. The user reports changing domain ObjectMapper configuration to ignore unknown properties, testing fact and dimension locally, publishing appropriate domain jars for three branches and validating the deployed QA endpoints. A QA teammate reported successful manual and pipeline checks. This is the ticket's reported root cause and fix, not a general rule that all timeouts have that cause.

**Entitlements and payload shape.** [RBOC-201678](https://adlm.nielseniq.com/jira/browse/RBOC-201678) records comparisons of generated DMDS requests and responses for characteristic, hierarchy and group selections, `ALL`, `ALL EXCEPT` and `BETWEEN`. The user identified missing selection type/group IDs and reported failures for `ALL` and `BETWEEN`; [RBOC-202841](https://adlm.nielseniq.com/jira/browse/RBOC-202841) and [RBOC-202843](https://adlm.nielseniq.com/jira/browse/RBOC-202843) were reported by the user and subsequently marked fixed by another assignee. In [RBOC-206619](https://adlm.nielseniq.com/jira/browse/RBOC-206619), the user describes testing hierarchy selections built from both `filterValues` and `values` in the adapter. For `HIDE COLUMN` and `EXCLUDE COLUMN`, [RBOC-206622](https://adlm.nielseniq.com/jira/browse/RBOC-206622) and [RBOC-206621](https://adlm.nielseniq.com/jira/browse/RBOC-206621) instead identify incorrect example payloads missing selection type/IDs; do not conflate these with the hierarchy code defect. The downloaded DMDS response attachments illustrate why an overall status alone is insufficient: `DMDSResponse6.json` has overall `SUCCESS` but two individual `FAILURE` responses, and `DMDSResponse7.json` has one.

**Multiregion routing.** [RBOC-205446](https://adlm.nielseniq.com/jira/browse/RBOC-205446) distinguishes two issues: a region-specific DMDS URL could resolve in the home region rather than the target region, and a BI-package callback could look for a job in the wrong region. [RBOC-206642](https://adlm.nielseniq.com/jira/browse/RBOC-206642) describes gateway URLs, propagation of stack headers to entitlement catalog calls, a dimension-cache toggle and US/EU validation. [RBOC-214712](https://adlm.nielseniq.com/jira/browse/RBOC-214712) and [RBOC-216555](https://adlm.nielseniq.com/jira/browse/RBOC-216555) record the shared REST invocation bean and consumer updates, including work on the fact `sync/biPackage` path. Two separate follow-up tickets, [RBOC-216562](https://adlm.nielseniq.com/jira/browse/RBOC-216562) and [RBOC-216586](https://adlm.nielseniq.com/jira/browse/RBOC-216586), remain Not Started in this snapshot; that does not erase the validation reported in other tickets.

**Large report / metadata collision.** [OPTIQ-2241](https://adlm.nielseniq.com/jira/browse/OPTIQ-2241) reports that downloading the BI package and dimension-selection metadata succeeded, but flat conversion failed when a metadata lookup returned a reference-only entry. The user's analysis attributes this to colliding hash-generated IDs stored in the same map, and escalates the issue to BI. [OPTIQ-2305](https://adlm.nielseniq.com/jira/browse/OPTIQ-2305) references the BI adapter fix and a validation example. The two very large blob attachments were inventoried but not independently downloaded or checked, so the collision mechanism here is the Jira-reported analysis, not a separate verification of the blobs.

### 18.4 POD, conditions, templates and product groups

**POD and POC UI.** [RBOC-203110](https://adlm.nielseniq.com/jira/browse/RBOC-203110) documents comparing Discover and fact-generated payloads for facts, products, retailer, demographics and people-related groups. The user observed incorrect/mismatched group attributes and helped reason about the simple-payload design. [RBOC-217696](https://adlm.nielseniq.com/jira/browse/RBOC-217696) traces PPL selections through `expressionGroup` references to buyer/shopper groups and reports testing Bought and Shopped At. [OPTIQ-1065](https://adlm.nielseniq.com/jira/browse/OPTIQ-1065) and [OPTIQ-213](https://adlm.nielseniq.com/jira/browse/OPTIQ-213) contain paired simple/complex examples and Discover/POC comparisons for bought, shopped, conversion, deal, loyalty, frequency and quantity cases. Each issue's downloaded JSON set contains 17 matched simple/complex filename pairs; all parse as JSON, but parsing does not independently establish numerical equivalence. The September update on OPTIQ-1065 explicitly excluded non-conversion and contribution from completed end-to-end verification at that time; [OPTIQ-2042](https://adlm.nielseniq.com/jira/browse/OPTIQ-2042) later reports additional non-converted-shopper UI work.

**Shifting and leakage.** [OPTIQ-553](https://adlm.nielseniq.com/jira/browse/OPTIQ-553) compares Discover and POC BI requests and identifies model and selection differences, fixed items dependent on report-module prompts, and an aggregate-engine/DDR mismatch requiring BI collaboration. [OPTIQ-706](https://adlm.nielseniq.com/jira/browse/OPTIQ-706) records subsequent domain changes and an open model-ID question at that point. The later [OPTIQ-710](https://adlm.nielseniq.com/jira/browse/OPTIQ-710) update reports dimensions `BRAND_SU`, `RETAILER_SU`, `SHPGP1`, `BUYGP1` and `COMP_PRD`, model IDs `BSF-P`, `BSF-R` and `CSE` with `MKSE` default, and model-specific `filterWithValues`; its attached compact payload independently contains `BSF-P`. The ticket links separate domain PRs for simple/compact and encoded payloads; the ticket does not by itself prove every deployment or all leakage scenarios are complete.

**T&D product group.** [OPTIQ-2533](https://adlm.nielseniq.com/jira/browse/OPTIQ-2533) says the UI calls `GET_PRODUCT_GROUP_VALUES`, stores selections and passes `pgKeys` in later dimension and fact request context, warning before other selections if no product group is chosen. [OPTIQ-2534](https://adlm.nielseniq.com/jira/browse/OPTIQ-2534) reports comparison against Discover for multiple API calls and compact, structured and encoded fact payloads. [OPTIQ-2275](https://adlm.nielseniq.com/jira/browse/OPTIQ-2275) reports currency-node calls returning no values without the keys and matching Discover with them; it attributes the shared metadata-query implementation to another engineer.

**Selection validation and UI details.** [RBOC-219053](https://adlm.nielseniq.com/jira/browse/RBOC-219053) records research into Discover context-carry validation and a proposed fact flow: validate the request, transform the simple payload, call DMDS consolidated items and expose selection errors. The user reports changes in the BI adapter and a fact feature branch; the record does not establish their current deployment state. [RBOC-218219](https://adlm.nielseniq.com/jira/browse/RBOC-218219) identifies the `SHR_DIM` metadata attribute as a way to determine the dimension requested by a share fact. In [OPTIQ-1102](https://adlm.nielseniq.com/jira/browse/OPTIQ-1102), the user reports that treating plain-text labels such as `% Price Gap` as jQuery selectors broke POC tree selection, and describes a targeted label-extraction fix.

**Current POD condition investigation.** [OPTIQ-2962](https://adlm.nielseniq.com/jira/browse/OPTIQ-2962) is In Progress. Its attached simple payload combines a PRDC top-three condition on `MKS_E_PURCHASE_DOLLARS_O` with an `AND`/`LESS_THAN` rule and a PPL Shopped-at expression. The attached fact-service BI payload represents a top-level `biCondition`; the Discover BI payload instead includes a `conditionAreaFilters` condition set, with a top/bottom condition and a `BiConditionFilter`. The attached screenshots show a Discover report and a fact response with HTTP 500 and `DataFactoryServiceImpl - retrieveData`. These are concrete differences and symptoms, **not an established root cause**: the two request representations are not necessarily directly equivalent, and no fix or successful retest is recorded in the ticket.

### 18.5 Reliability, testing and delivery

* **Observability and autoscaling:** [RBOC-197495](https://adlm.nielseniq.com/jira/browse/RBOC-197495) records fact/dimension dashboards with environment, service and version drilldowns. [RBOC-192097](https://adlm.nielseniq.com/jira/browse/RBOC-192097) records CPU, heap, pod memory, restart, thread, throughput and latency alerts, plus Datadog-to-Coralogix HPA work; [RBOC-200967](https://adlm.nielseniq.com/jira/browse/RBOC-200967) reports subsequent environment validation. [RBOC-210327](https://adlm.nielseniq.com/jira/browse/RBOC-210327) compares log counts and fixes production alert targets. [RBOC-214439](https://adlm.nielseniq.com/jira/browse/RBOC-214439) initially used Checkly for non-production and Datadog for production; [OPTIQ-548](https://adlm.nielseniq.com/jira/browse/OPTIQ-548) later reports Checkly monitoring for production and UAT.
* **Performance:** [RBOC-206822](https://adlm.nielseniq.com/jira/browse/RBOC-206822) and [RBOC-210324](https://adlm.nielseniq.com/jira/browse/RBOC-210324) describe a JMeter CI workflow and InfluxDB/Grafana integration. [RBOC-218218](https://adlm.nielseniq.com/jira/browse/RBOC-218218) reports generated EU performance payloads at 0.5M-2.5M cells and smaller US/EU cases. [OPTIQ-1142](https://adlm.nielseniq.com/jira/browse/OPTIQ-1142) reports timeouts in its JDK 25 test matrix at the stated 150-request/30-second ramp, not a universal JDK 25 benchmark. [OPTIQ-681](https://adlm.nielseniq.com/jira/browse/OPTIQ-681) records a cache warmup change and explicitly cautions that further issues may remain.
* **Tests and CI:** [RBOC-203273](https://adlm.nielseniq.com/jira/browse/RBOC-203273) adds Pluto coverage for missing report items; [RBOC-206809](https://adlm.nielseniq.com/jira/browse/RBOC-206809) covers missing dimension auth/transaction headers; [RBOC-210538](https://adlm.nielseniq.com/jira/browse/RBOC-210538) and [RBOC-214567](https://adlm.nielseniq.com/jira/browse/RBOC-214567) cover product and market share facts. [OPTIQ-1733](https://adlm.nielseniq.com/jira/browse/OPTIQ-1733) reports adding Gradle tests to domain jar and consumer image builds. [RBOC-217698](https://adlm.nielseniq.com/jira/browse/RBOC-217698) has a title about a missing transaction ID but its stated example describes a missing authentication token; its actual test contract should be checked in the linked PR before reuse.
* **Security:** [OPTIQ-1809](https://adlm.nielseniq.com/jira/browse/OPTIQ-1809), [OPTIQ-1954](https://adlm.nielseniq.com/jira/browse/OPTIQ-1954), [OPTIQ-1955](https://adlm.nielseniq.com/jira/browse/OPTIQ-1955) and [OPTIQ-1956](https://adlm.nielseniq.com/jira/browse/OPTIQ-1956) describe dimension/POC UI SAST, dependency and stale-workflow secret findings. [OPTIQ-2475](https://adlm.nielseniq.com/jira/browse/OPTIQ-2475) reports additional UI sanitization, restricted health-endpoint CORS and limited authenticated DAST coverage. A planned retirement date for an old connection URL is not evidence that retirement occurred. [OPTIQ-2330](https://adlm.nielseniq.com/jira/browse/OPTIQ-2330) was reported by the user but worked by another assignee; its comments identify dependencies and upstream library remediation still needed at the time.

### 18.6 AI-assisted SDLC and evidence boundaries

[RBOC-196929](https://adlm.nielseniq.com/jira/browse/RBOC-196929), [RBOC-197499](https://adlm.nielseniq.com/jira/browse/RBOC-197499), [RBOC-199481](https://adlm.nielseniq.com/jira/browse/RBOC-199481) and [RBOC-200968](https://adlm.nielseniq.com/jira/browse/RBOC-200968) trace the user's work on MCP-backed agent instructions, quality gates, cross-repository jar handoffs, deployment checks and team enablement. [RBOC-196657](https://adlm.nielseniq.com/jira/browse/RBOC-196657) records a small logging task with reported CI, deployment and sanity evidence. [RBOC-199783](https://adlm.nielseniq.com/jira/browse/RBOC-199783) reports a domain-only NodeId experiment on a feature branch, with consumer integration still a handoff. [RBOC-200375](https://adlm.nielseniq.com/jira/browse/RBOC-200375) reports a cross-repository DTO-introspection experiment, but its eventual Jira status is Withdrawn. Neither should be assumed to exist on main.

[RBOC-197208](https://adlm.nielseniq.com/jira/browse/RBOC-197208) requested a same-replica notification fast path with Mongo fallback and required matched baseline/post-change performance measurements. Its comments report a deployment sanity pass and one sample of one Mongo read, but the retrieved ticket does **not** contain the requested before/after comparison. A Done status and a single observation do not establish the performance improvement. Symbio work later covered SDLC workflows, RAG, LaunchFlow and telemetry triage in [RBOC-219498](https://adlm.nielseniq.com/jira/browse/RBOC-219498), [OPTIQ-321](https://adlm.nielseniq.com/jira/browse/OPTIQ-321), [OPTIQ-713](https://adlm.nielseniq.com/jira/browse/OPTIQ-713) and [OPTIQ-720](https://adlm.nielseniq.com/jira/browse/OPTIQ-720); [OPTIQ-719](https://adlm.nielseniq.com/jira/browse/OPTIQ-719) contains a teammate's RAG ingestion update, not evidence that the user performed that particular ingestion.

### 18.7 Current and unresolved work at the snapshot

* [OPTIQ-2726](https://adlm.nielseniq.com/jira/browse/OPTIQ-2726) and [OPTIQ-2959](https://adlm.nielseniq.com/jira/browse/OPTIQ-2959): Optiq v1 flow and use-case discovery, In Progress; the ticket points to the `E2E_Optiq_Web` issue set. [OPTIQ-2960](https://adlm.nielseniq.com/jira/browse/OPTIQ-2960), the related validation task, is Not Started.
* [OPTIQ-2962](https://adlm.nielseniq.com/jira/browse/OPTIQ-2962): POD custom condition and rule failure, In Progress, with the payload comparison above.
* [OPTIQ-2673](https://adlm.nielseniq.com/jira/browse/OPTIQ-2673): further product security score work, Not Started; [OPTIQ-2043](https://adlm.nielseniq.com/jira/browse/OPTIQ-2043) is Ready for UAT, with pending follow-up in its November comment.
* [RBOC-199373](https://adlm.nielseniq.com/jira/browse/RBOC-199373), [RBOC-216562](https://adlm.nielseniq.com/jira/browse/RBOC-216562) and [RBOC-216586](https://adlm.nielseniq.com/jira/browse/RBOC-216586) are Not Started, despite related work being documented elsewhere.

### 18.8 Complete retrieved issue index

Dates below are issue creation dates, not necessarily when work occurred. **Owner** is the current assignee at retrieval: `user` means `uja5002`; `other` and `unassigned` are retained because historical assignment or reporting matched the search. Consult the linked issue for full descriptions, comments, attachments and subsequent status changes.

| Issue | Created | Summary | Status | Owner |
|---|---|---|---|---|
| [OMNI2-1870](https://adlm.nielseniq.com/jira/browse/OMNI2-1870) | 2023-05-22 | Buyer Conversion/Trip Conversion Facts | Withdrawn | user |
| [RBOC-180616](https://adlm.nielseniq.com/jira/browse/RBOC-180616) | 2025-11-28 | Validate app and event logs from dimension and fact APIs | Done | user |
| [RBOC-188559](https://adlm.nielseniq.com/jira/browse/RBOC-188559) | 2026-01-27 | KT and local setup | Done | user |
| [RBOC-189605](https://adlm.nielseniq.com/jira/browse/RBOC-189605) | 2026-02-06 | Validate updated DF code for period | Done | user |
| [RBOC-189867](https://adlm.nielseniq.com/jira/browse/RBOC-189867) | 2026-02-09 | KT | Done | user |
| [RBOC-190178](https://adlm.nielseniq.com/jira/browse/RBOC-190178) | 2026-02-11 | Fix Sonar issues | Done | user |
| [RBOC-191155](https://adlm.nielseniq.com/jira/browse/RBOC-191155) | 2026-02-20 | HPA query, memory and CPU analysis | Done | other |
| [RBOC-191436](https://adlm.nielseniq.com/jira/browse/RBOC-191436) | 2026-02-22 | Merge event log PR | Done | user |
| [RBOC-192097](https://adlm.nielseniq.com/jira/browse/RBOC-192097) | 2026-02-25 | Configure API monitoring and alerts | Done | user |
| [RBOC-192228](https://adlm.nielseniq.com/jira/browse/RBOC-192228) | 2026-02-26 | POC code deployment | Done | user |
| [RBOC-194233](https://adlm.nielseniq.com/jira/browse/RBOC-194233) | 2026-03-06 | Transaction IDs in logs | Accepted | user |
| [RBOC-194301](https://adlm.nielseniq.com/jira/browse/RBOC-194301) | 2026-03-06 | PERF and hotfix environment readiness | Accepted | user |
| [RBOC-194454](https://adlm.nielseniq.com/jira/browse/RBOC-194454) | 2026-03-08 | Deploy POC UI in dev | Done | other |
| [RBOC-194792](https://adlm.nielseniq.com/jira/browse/RBOC-194792) | 2026-03-09 | Implement transaction IDs in logs | Done | user |
| [RBOC-194795](https://adlm.nielseniq.com/jira/browse/RBOC-194795) | 2026-03-09 | Test transaction ID classes | Done | user |
| [RBOC-195382](https://adlm.nielseniq.com/jira/browse/RBOC-195382) | 2026-03-11 | USS validation task | Done | user |
| [RBOC-195625](https://adlm.nielseniq.com/jira/browse/RBOC-195625) | 2026-03-12 | USS migration support | Done | user |
| [RBOC-196657](https://adlm.nielseniq.com/jira/browse/RBOC-196657) | 2026-03-23 | Add application log for payload transformation | Accepted | user |
| [RBOC-196929](https://adlm.nielseniq.com/jira/browse/RBOC-196929) | 2026-03-25 | AI adoption in SDLC | Done | user |
| [RBOC-196931](https://adlm.nielseniq.com/jira/browse/RBOC-196931) | 2026-03-25 | Dummy ticket using custom SDLC agent | Done | user |
| [RBOC-197208](https://adlm.nielseniq.com/jira/browse/RBOC-197208) | 2026-03-31 | Multi-replica-safe sync status acceleration | Done | user |
| [RBOC-197495](https://adlm.nielseniq.com/jira/browse/RBOC-197495) | 2026-04-02 | Coralogix dashboard creation | Done | user |
| [RBOC-197499](https://adlm.nielseniq.com/jira/browse/RBOC-197499) | 2026-04-02 | AI adoption in SDLC | Done | user |
| [RBOC-197501](https://adlm.nielseniq.com/jira/browse/RBOC-197501) | 2026-04-02 | JM AKS-02 migration support | Done | other |
| [RBOC-197511](https://adlm.nielseniq.com/jira/browse/RBOC-197511) | 2026-04-02 | BI package HTML table endpoint experiment | Withdrawn | user |
| [RBOC-199373](https://adlm.nielseniq.com/jira/browse/RBOC-199373) | 2026-04-16 | Coralogix dashboard | Not Started | user |
| [RBOC-199481](https://adlm.nielseniq.com/jira/browse/RBOC-199481) | 2026-04-17 | AI SDLC | Done | user |
| [RBOC-199748](https://adlm.nielseniq.com/jira/browse/RBOC-199748) | 2026-04-20 | Test Copilot SDLC agent on a ticket | Done | user |
| [RBOC-199783](https://adlm.nielseniq.com/jira/browse/RBOC-199783) | 2026-04-21 | Domain NodeId validation and structured logging | Dev Complete | user |
| [RBOC-200375](https://adlm.nielseniq.com/jira/browse/RBOC-200375) | 2026-04-28 | DTO field introspection and fact utility endpoint | Withdrawn | user |
| [RBOC-200967](https://adlm.nielseniq.com/jira/browse/RBOC-200967) | 2026-05-04 | Coralogix HPA migration and alerts | Done | user |
| [RBOC-200968](https://adlm.nielseniq.com/jira/browse/RBOC-200968) | 2026-05-04 | AI adoption in SDLC | Done | user |
| [TESTPJ-33](https://adlm.nielseniq.com/jira/browse/TESTPJ-33) | 2026-05-06 | Dummy | Created | unassigned |
| [RBOC-201339](https://adlm.nielseniq.com/jira/browse/RBOC-201339) | 2026-05-06 | Enriched fallback transaction IDs | Withdrawn | user |
| [RBOC-201379](https://adlm.nielseniq.com/jira/browse/RBOC-201379) | 2026-05-06 | Dimension-only SDLC task | Done | unassigned |
| [RBOC-201380](https://adlm.nielseniq.com/jira/browse/RBOC-201380) | 2026-05-06 | Dimension/domain multi-repo SDLC task | Done | unassigned |
| [RBOC-201678](https://adlm.nielseniq.com/jira/browse/RBOC-201678) | 2026-05-08 | Entitlement validation and BI adapter move | Done | user |
| [OPTIQ-213](https://adlm.nielseniq.com/jira/browse/OPTIQ-213) | 2026-05-11 | POC UI validation and enhancement | Done | user |
| [RBOC-201990](https://adlm.nielseniq.com/jira/browse/RBOC-201990) | 2026-05-12 | Missing stack-header fallback experiment | Withdrawn | user |
| [RBOC-202351](https://adlm.nielseniq.com/jira/browse/RBOC-202351) | 2026-05-13 | Fact service QA timeout | Accepted | user |
| [RBOC-202841](https://adlm.nielseniq.com/jira/browse/RBOC-202841) | 2026-05-15 | Entitlement `ALL` failure | Done | other |
| [RBOC-202843](https://adlm.nielseniq.com/jira/browse/RBOC-202843) | 2026-05-15 | Entitlement `BETWEEN` failure | Done | other |
| [RBOC-203110](https://adlm.nielseniq.com/jira/browse/RBOC-203110) | 2026-05-18 | POD dataset analysis | Done | user |
| [RBOC-203273](https://adlm.nielseniq.com/jira/browse/RBOC-203273) | 2026-05-18 | Pluto missing-report-items tests | Done | user |
| [RBOC-204281](https://adlm.nielseniq.com/jira/browse/RBOC-204281) | 2026-05-21 | Entitlement change validation | Done | user |
| [RBOC-205194](https://adlm.nielseniq.com/jira/browse/RBOC-205194) | 2026-05-25 | Pluto test creation using AI SDLC | Done | user |
| [RBOC-205446](https://adlm.nielseniq.com/jira/browse/RBOC-205446) | 2026-05-26 | Validate multiregion use cases | Done | user |
| [RBOC-206619](https://adlm.nielseniq.com/jira/browse/RBOC-206619) | 2026-06-01 | Entitlement hierarchy selection gaps | Done | user |
| [RBOC-206621](https://adlm.nielseniq.com/jira/browse/RBOC-206621) | 2026-06-01 | Entitlement exclude-column gap | Done | other |
| [RBOC-206622](https://adlm.nielseniq.com/jira/browse/RBOC-206622) | 2026-06-01 | Entitlement hide-column gap | Done | user |
| [RBOC-206623](https://adlm.nielseniq.com/jira/browse/RBOC-206623) | 2026-06-01 | Entitlement exclude-totals gap | Done | other |
| [RBOC-206640](https://adlm.nielseniq.com/jira/browse/RBOC-206640) | 2026-06-01 | POD simple-payload design support | Done | user |
| [RBOC-206642](https://adlm.nielseniq.com/jira/browse/RBOC-206642) | 2026-06-01 | Implement and validate multiregion support | Done | user |
| [RBOC-206809](https://adlm.nielseniq.com/jira/browse/RBOC-206809) | 2026-06-01 | Dimension missing-header Pluto tests | Done | user |
| [RBOC-206816](https://adlm.nielseniq.com/jira/browse/RBOC-206816) | 2026-06-01 | Execute AI SDLC task | Done | user |
| [RBOC-206822](https://adlm.nielseniq.com/jira/browse/RBOC-206822) | 2026-06-01 | Performance testing CI/CD | Done | user |
| [RBOC-210324](https://adlm.nielseniq.com/jira/browse/RBOC-210324) | 2026-06-15 | Performance testing CI/CD | Done | user |
| [RBOC-210327](https://adlm.nielseniq.com/jira/browse/RBOC-210327) | 2026-06-15 | Coralogix enhancement | Done | user |
| [RBOC-210538](https://adlm.nielseniq.com/jira/browse/RBOC-210538) | 2026-06-16 | Pluto product share-fact test | Done | user |
| [RBOC-210625](https://adlm.nielseniq.com/jira/browse/RBOC-210625) | 2026-06-16 | Test AI SDLC workflow | Done | user |
| [RBOC-210627](https://adlm.nielseniq.com/jira/browse/RBOC-210627) | 2026-06-16 | Refresh-date PRs | Done | user |
| [RBOC-210641](https://adlm.nielseniq.com/jira/browse/RBOC-210641) | 2026-06-16 | Validate production APIs through POC UI | Done | user |
| [RBOC-214439](https://adlm.nielseniq.com/jira/browse/RBOC-214439) | 2026-06-29 | Synthetic monitoring with Checkly | Done | user |
| [RBOC-214567](https://adlm.nielseniq.com/jira/browse/RBOC-214567) | 2026-06-29 | Pluto market share-fact test | Done | user |
| [RBOC-214712](https://adlm.nielseniq.com/jira/browse/RBOC-214712) | 2026-06-30 | REST invocation bean | Done | user |
| [RBOC-216555](https://adlm.nielseniq.com/jira/browse/RBOC-216555) | 2026-07-02 | Fact/dimension multiregion invocation changes | Done | user |
| [RBOC-216562](https://adlm.nielseniq.com/jira/browse/RBOC-216562) | 2026-07-02 | Test fact multiregion usage | Not Started | user |
| [RBOC-216586](https://adlm.nielseniq.com/jira/browse/RBOC-216586) | 2026-07-03 | Test dimension multiregion usage | Not Started | user |
| [RBOC-216645](https://adlm.nielseniq.com/jira/browse/RBOC-216645) | 2026-07-03 | POC UI transaction ID parameter | Done | user |
| [RBOC-217225](https://adlm.nielseniq.com/jira/browse/RBOC-217225) | 2026-07-08 | People dimension in POC UI | Done | user |
| [RBOC-217696](https://adlm.nielseniq.com/jira/browse/RBOC-217696) | 2026-07-13 | PPL dimension in POC UI | Done | user |
| [RBOC-217698](https://adlm.nielseniq.com/jira/browse/RBOC-217698) | 2026-07-13 | BI package missing-transaction-ID Pluto test | Done | user |
| [RBOC-218218](https://adlm.nielseniq.com/jira/browse/RBOC-218218) | 2026-07-17 | EU PERF payload generation | Done | user |
| [RBOC-218219](https://adlm.nielseniq.com/jira/browse/RBOC-218219) | 2026-07-17 | Identify share facts | Done | user |
| [RBOC-218220](https://adlm.nielseniq.com/jira/browse/RBOC-218220) | 2026-07-17 | Discover invalid-selection investigation | Done | user |
| [RBOC-218675](https://adlm.nielseniq.com/jira/browse/RBOC-218675) | 2026-07-23 | Okta-to-Ping migration support | Done | user |
| [RBOC-219053](https://adlm.nielseniq.com/jira/browse/RBOC-219053) | 2026-07-27 | Invalid-selection validation flow | Done | user |
| [RBOC-219054](https://adlm.nielseniq.com/jira/browse/RBOC-219054) | 2026-07-27 | BI package missing-token Pluto test | Done | user |
| [OPTIQ-321](https://adlm.nielseniq.com/jira/browse/OPTIQ-321) | 2026-07-28 | Symbio SDLC rollout | Done | user |
| [OPTIQ-713](https://adlm.nielseniq.com/jira/browse/OPTIQ-713) | 2026-07-28 | Symbio LaunchFlow and sandboxes | Done | user |
| [OPTIQ-719](https://adlm.nielseniq.com/jira/browse/OPTIQ-719) | 2026-07-28 | Symbio RAG rollout | Done | other |
| [OPTIQ-720](https://adlm.nielseniq.com/jira/browse/OPTIQ-720) | 2026-07-28 | Symbio Triage onboarding | Done | user |
| [RBOC-219319](https://adlm.nielseniq.com/jira/browse/RBOC-219319) | 2026-07-28 | Omni template/model mapping | Done | user |
| [RBOC-219498](https://adlm.nielseniq.com/jira/browse/RBOC-219498) | 2026-07-29 | Symbio epic analysis | Done | user |
| [RBOC-219616](https://adlm.nielseniq.com/jira/browse/RBOC-219616) | 2026-07-30 | Compare RMS templates with Unified API | Done | user |
| [RBOC-219870](https://adlm.nielseniq.com/jira/browse/RBOC-219870) | 2026-08-03 | RMS template validation support | Done | user |
| [RBOC-220265](https://adlm.nielseniq.com/jira/browse/RBOC-220265) | 2026-08-06 | Performance regression analysis | Done | other |
| [RBOC-220276](https://adlm.nielseniq.com/jira/browse/RBOC-220276) | 2026-08-06 | POD group-level items as characteristics | Done | other |
| [OPTIQ-548](https://adlm.nielseniq.com/jira/browse/OPTIQ-548) | 2026-08-09 | Cost optimization and infrastructure stability | Done | user |
| [OPTIQ-553](https://adlm.nielseniq.com/jira/browse/OPTIQ-553) | 2026-08-10 | Shifting analysis template investigation | Done | user |
| [OPTIQ-681](https://adlm.nielseniq.com/jira/browse/OPTIQ-681) | 2026-08-14 | QA timeout investigation | Done | user |
| [OPTIQ-703](https://adlm.nielseniq.com/jira/browse/OPTIQ-703) | 2026-08-17 | BI handshake for leakage template | Done | user |
| [OPTIQ-706](https://adlm.nielseniq.com/jira/browse/OPTIQ-706) | 2026-08-17 | Expand fact API for shifting analysis | Accepted | user |
| [OPTIQ-710](https://adlm.nielseniq.com/jira/browse/OPTIQ-710) | 2026-08-17 | Shifting analysis end-to-end validation | Done | user |
| [OPTIQ-1065](https://adlm.nielseniq.com/jira/browse/OPTIQ-1065) | 2026-08-24 | POC UI PPL and condition use cases | Accepted | user |
| [OPTIQ-1102](https://adlm.nielseniq.com/jira/browse/OPTIQ-1102) | 2026-08-25 | Walmart Price Beats selection defect | Done | user |
| [OPTIQ-1142](https://adlm.nielseniq.com/jira/browse/OPTIQ-1142) | 2026-08-28 | Load tests and thread dumps | Done | user |
| [OPTIQ-1388](https://adlm.nielseniq.com/jira/browse/OPTIQ-1388) | 2026-09-08 | Auto-created stack-overflow candidate | Withdrawn | unassigned |
| [OPTIQ-1699](https://adlm.nielseniq.com/jira/browse/OPTIQ-1699) | 2026-09-24 | Logging standardization | Done | user |
| [OPTIQ-1732](https://adlm.nielseniq.com/jira/browse/OPTIQ-1732) | 2026-09-25 | Encoded POD payload validation | Done | other |
| [OPTIQ-1733](https://adlm.nielseniq.com/jira/browse/OPTIQ-1733) | 2026-09-25 | Run Gradle tests in CI and jar build | Done | user |
| [OPTIQ-1805](https://adlm.nielseniq.com/jira/browse/OPTIQ-1805) | 2026-09-30 | Fact product security score | Accepted | other |
| [OPTIQ-1808](https://adlm.nielseniq.com/jira/browse/OPTIQ-1808) | 2026-09-30 | Dimension Cycode findings | Done | user |
| [OPTIQ-1809](https://adlm.nielseniq.com/jira/browse/OPTIQ-1809) | 2026-09-30 | Dimension product security score | Accepted | user |
| [OPTIQ-1954](https://adlm.nielseniq.com/jira/browse/OPTIQ-1954) | 2026-10-08 | Cycode SAST findings | Done | user |
| [OPTIQ-1955](https://adlm.nielseniq.com/jira/browse/OPTIQ-1955) | 2026-10-08 | Cycode SCA findings | Done | user |
| [OPTIQ-1956](https://adlm.nielseniq.com/jira/browse/OPTIQ-1956) | 2026-10-08 | Cycode secret findings | Done | user |
| [OPTIQ-2042](https://adlm.nielseniq.com/jira/browse/OPTIQ-2042) | 2026-10-14 | POC UI enhancement | Done | user |
| [OPTIQ-2043](https://adlm.nielseniq.com/jira/browse/OPTIQ-2043) | 2026-10-14 | Fact/dimension security score | Ready for UAT | user |
| [OPTIQ-2045](https://adlm.nielseniq.com/jira/browse/OPTIQ-2045) | 2026-10-14 | Buyers and shoppers template validation | Done | other |
| [OPTIQ-2205](https://adlm.nielseniq.com/jira/browse/OPTIQ-2205) | 2026-10-22 | Fact/domain dependency version improvements | Done | user |
| [OPTIQ-2241](https://adlm.nielseniq.com/jira/browse/OPTIQ-2241) | 2026-10-23 | Activate large-report support | Done | user |
| [OPTIQ-2243](https://adlm.nielseniq.com/jira/browse/OPTIQ-2243) | 2026-10-23 | Product-group selection in POC UI | Done | user |
| [OPTIQ-2275](https://adlm.nielseniq.com/jira/browse/OPTIQ-2275) | 2026-10-26 | Product group keys in currency dimension | Done | user |
| [OPTIQ-2305](https://adlm.nielseniq.com/jira/browse/OPTIQ-2305) | 2026-10-27 | Validate large-selection metadata blob | Done | user |
| [OPTIQ-2330](https://adlm.nielseniq.com/jira/browse/OPTIQ-2330) | 2026-10-27 | Wiz findings | Done | other |
| [OPTIQ-2475](https://adlm.nielseniq.com/jira/browse/OPTIQ-2475) | 2026-11-02 | Improve fact/dimension security score | Done | user |
| [OPTIQ-2533](https://adlm.nielseniq.com/jira/browse/OPTIQ-2533) | 2026-11-05 | POC UI product-group support | Done | user |
| [OPTIQ-2534](https://adlm.nielseniq.com/jira/browse/OPTIQ-2534) | 2026-11-05 | Product-group end-to-end validation | Done | user |
| [OPTIQ-2673](https://adlm.nielseniq.com/jira/browse/OPTIQ-2673) | 2026-11-11 | Fact/dimension product security score | Not Started | user |
| [OPTIQ-2726](https://adlm.nielseniq.com/jira/browse/OPTIQ-2726) | 2026-11-12 | Optiq v1 use-case analysis | In Progress | user |
| [OPTIQ-2959](https://adlm.nielseniq.com/jira/browse/OPTIQ-2959) | 2026-11-19 | Gather Optiq flow and use cases | In Progress | user |
| [OPTIQ-2960](https://adlm.nielseniq.com/jira/browse/OPTIQ-2960) | 2026-11-19 | Validate Optiq use cases | Not Started | user |
| [OPTIQ-2962](https://adlm.nielseniq.com/jira/browse/OPTIQ-2962) | 2026-11-19 | Validate POD condition use cases | In Progress | user |

## 19. Second-pass ticket-by-ticket analysis

**Refreshed:** 2026-11-21. Section 18 remains the original 124-ticket assignment/reporter snapshot; this section expands it. A second Jira query using `issueFunction in commented("by uja5002 after 2026-01-01")` found eight additional tickets with user-authored comments, giving **132 distinct issues**. A worklog-author search since January returned no issues. The comment search is not a guarantee of all collaboration: issue visibility, search indexing and activity outside comments/assignment/reporting still apply.

The refreshed records include full descriptions, 355 returned comments and expanded change histories; parent and linked issue context was also checked for 48 additional visible related records. The eight additions are explicitly marked below. Statuses and current ownership for the original 124 are in section 18.8; for the eight additions they are stated in their entries. A Jira comment is evidence of what its author reported at that time, not an independent verification of a later release. Empty records and experiments are identified rather than filled in by assumption.

### 19.1 Earlier issues and onboarding

- [OMNI2-1870](https://adlm.nielseniq.com/jira/browse/OMNI2-1870) — An older epic for buyer/trip conversion facts combining trip and purchase data. Its 2023 comment says the work was integrated under another epic and recommends withdrawal; it was assigned to the user during the search window, but contains no user-authored work update.
- [RBOC-180616](https://adlm.nielseniq.com/jira/browse/RBOC-180616) — App/event logging validation across fact, dimension and domain. February-March comments trace adding request context and transaction IDs, testing against Datadog, investigating a provider/SPI to avoid tight coupling, adding flow logs and raising PRs; later design changes are recorded under RBOC-194233.
- [RBOC-188559](https://adlm.nielseniq.com/jira/browse/RBOC-188559) — Onboarding and local setup: Artifactory access, building/running dimension and fact services, trying metadata queries, and reviewing Discover/Unified Services KT material. The cited recordings and spreadsheet are references, not independently reviewed here.
- [RBOC-189605](https://adlm.nielseniq.com/jira/browse/RBOC-189605) — Period-related DF validation. The user's comment lists single/multiple and hierarchy selections, `ALL`, `ALL EXCEPT`, levels, sums, null/remaining and three additional LLD use cases; it does not provide a separate result for each case.
- [RBOC-189867](https://adlm.nielseniq.com/jira/browse/RBOC-189867) — KT on dimension payloads, results, logs and documentation, followed by domain integration with fact and dimension; no implementation claim beyond the user's updates.
- [RBOC-190178](https://adlm.nielseniq.com/jira/browse/RBOC-190178) — Sonar cleanup reported for metadata handling/enrichment, downstream calls, simple payload, exception/utilities and their tests. Its parent covers Sonar work across all three repositories.
- [RBOC-191155](https://adlm.nielseniq.com/jira/browse/RBOC-191155) — A historically assigned/shared HPA and load-testing ticket, currently owned by another engineer. Their comments discuss dimension replica configuration, JMeter measurements and fact tests at two million cells; do not credit those measurements to the user without additional evidence.
- [RBOC-191436](https://adlm.nielseniq.com/jira/browse/RBOC-191436) — Child of the app/event logging ticket. The only user comment says the event-log PR was still awaiting merge on March 6, even though the ticket is now Done; the record alone does not establish when it merged.
- [RBOC-192097](https://adlm.nielseniq.com/jira/browse/RBOC-192097) — Coralogix alerts and HPA migration for fact/dimension. The user lists CPU, heap, pod memory, throughput, latency, throttling, restarts, OOM and thread alerts, tests scaling, and notes synthetic monitoring/AppDex gaps and environment validation blockers.
- [RBOC-192228](https://adlm.nielseniq.com/jira/browse/RBOC-192228) — Investigated POC hosting and CI/deployment options and learned the existing GitHub Actions/Docker flow. The final comment says another engineer took over the actual UI deployment.
- [RBOC-193298](https://adlm.nielseniq.com/jira/browse/RBOC-193298) — **Newly found through comments; Done, unassigned.** The user reports a dimension Coralogix dashboard covering pod count/configuration, CPU, JVM/heap/pod memory, thread count, endpoint requests and error/OOM/Hazelcast logs, with environment widgets and region/namespace/image-tag drilldowns.
- [RBOC-193299](https://adlm.nielseniq.com/jira/browse/RBOC-193299) — **Newly found through comments; Done, unassigned.** The corresponding fact dashboard comment lists the same operational panels and drilldown variables. The two tickets link the same dashboard; they are not evidence of two independent dashboard implementations.
- [RBOC-193300](https://adlm.nielseniq.com/jira/browse/RBOC-193300) — **Newly found through comments; Done, unassigned.** Dimension alert setup included resource, request, latency and error conditions. The April comment explicitly says synthetic endpoint monitoring and an AppDex alert were not yet available through the same approach.
- [RBOC-193301](https://adlm.nielseniq.com/jira/browse/RBOC-193301) — **Newly found through comments; Done, unassigned.** Fact-service alert coverage mirrors the dimension ticket, with the same then-unresolved synthetic monitoring and AppDex limitations.
- [RBOC-193302](https://adlm.nielseniq.com/jira/browse/RBOC-193302) — **Newly found through comments; Withdrawn, unassigned.** The user compared fact/dimension dashboards with Datadog and said alert validation was in progress; a later team comment withdrew the ticket because dashboards and alerts were already available.
- [RBOC-193304](https://adlm.nielseniq.com/jira/browse/RBOC-193304) — **Newly found through comments; Withdrawn, unassigned.** The user reports checking non-production configuration, mapping Datadog to Coralogix trigger queries, deploying dev/QA configuration and verifying USQA scaling; a later comment says equivalent triggers were already available.
- [RBOC-194233](https://adlm.nielseniq.com/jira/browse/RBOC-194233) — Transaction IDs on application log lines. The user reports end-to-end validation, cross-repository PR coordination, static logger access and test/checkstyle work. The accepting comment says the eventual approach changed, so the prototype details are historical.
- [RBOC-194301](https://adlm.nielseniq.com/jira/browse/RBOC-194301) — PERF/hotfix readiness: compared deployment values with related services and surfaced a mismatched BI sync URL for another team to address. The April update still awaited that team's configuration change and deployment.
- [RBOC-194454](https://adlm.nielseniq.com/jira/browse/RBOC-194454) — Shared UI deployment ticket currently assigned to another engineer. Their updates describe workflow/image setup and deployment failures involving CSR mapping and a missing KEDA trigger; no user-authored deployment outcome is recorded.
- [RBOC-194792](https://adlm.nielseniq.com/jira/browse/RBOC-194792) — Child implementation for transaction IDs. The user describes a domain interface/holder and fact implementation, a `CompletableFuture` context-leak symptom, thread-local/header propagation, rebasing and tests. The final design must be verified against merged code, not inferred from the prototype alone.
- [RBOC-194795](https://adlm.nielseniq.com/jira/browse/RBOC-194795) — Tests for the new context-provider classes and affected service tests in domain, fact and dimension, with failing tests and Checkstyle addressed according to the user.
- [RBOC-195382](https://adlm.nielseniq.com/jira/browse/RBOC-195382) — Mixed validation/AI workflow updates: cross-repository instructions and a domain jar-build handoff were explored; a separate comment says deployment validation had not started. A later team update says non-production and production YAML validation looked good for AKS-02 migration.
- [RBOC-195625](https://adlm.nielseniq.com/jira/browse/RBOC-195625) — KEDA/CNI migration support for BI services. The user's closure comment says no client queries arose; it should not be read as evidence that the user performed all migration deployments.
- [RBOC-196657](https://adlm.nielseniq.com/jira/browse/RBOC-196657) — Small SDLC exercise to log the start of simple-to-complex transformation with dataset ID. The user reports unit assertion, branch PR, build, US dev deployment and sanity success; the comment explicitly notes Cycode scans were skipped by instruction for that run.
- [RBOC-196929](https://adlm.nielseniq.com/jira/browse/RBOC-196929) — Established an MCP-backed SDLC agent and step instructions, compared other teams' approaches, ran a logging task through deployment/sanity, and refined fact/domain handoff instructions. The user also reports trying a more substantive sync-flow ticket.
- [RBOC-196931](https://adlm.nielseniq.com/jira/browse/RBOC-196931) — A dummy ticket for validating the SDLC instruction workflow through PR/deployment. Description only; no comments establish an independent outcome.
- [RBOC-197208](https://adlm.nielseniq.com/jira/browse/RBOC-197208) — Requested local notification plus Mongo fallback for replica-safe sync completion, preserving error/timeout behavior and requiring comparable latency/read-count measurements. Comments report USQA sanity and one sample with one Mongo read; the mandatory before/after table is not present.

### 19.2 April-May: observability, AI workflow and entitlement work

- [RBOC-197495](https://adlm.nielseniq.com/jira/browse/RBOC-197495) — Built and refined dashboards across eight listed environments, validated against Datadog and added service/region/namespace/version drilldowns and operational metrics. Comments describe iterative query and widget refinement.
- [RBOC-197499](https://adlm.nielseniq.com/jira/browse/RBOC-197499) — Demonstrated and shared AI-SDLC setup, helped colleagues, prepared Cycode CLI guidance and integrated deployment-specific Coralogix queries and Pluto triggering into the workflow instructions.
- [RBOC-197501](https://adlm.nielseniq.com/jira/browse/RBOC-197501) — AKS-02 migration support, currently assigned to another engineer. Its sole comment reports no issues; it does not describe user-authored remediation.
- [RBOC-197511](https://adlm.nielseniq.com/jira/browse/RBOC-197511) — Withdrawn proposal for a fact endpoint to orchestrate BI-package decompaction and HTML rendering. It has detailed acceptance criteria but no comments establishing delivery.
- [RBOC-199373](https://adlm.nielseniq.com/jira/browse/RBOC-199373) — Not Started Coralogix dashboard ticket with no description/comments. Separate completed dashboard work exists, but should not be substituted as evidence of completion of this ticket.
- [RBOC-199481](https://adlm.nielseniq.com/jira/browse/RBOC-199481) — Extended SDLC instructions to domain and fact, identified cross-repository behavior such as log-level mismatch, and tested domain-only and domain-plus-fact exercises. The user reports iterative refinement of gates, MCP steps and downstream jar handoff.
- [RBOC-199748](https://adlm.nielseniq.com/jira/browse/RBOC-199748) — Tracking ticket referring to SDLC exercises RBOC-199783 and RBOC-200375; the comment reports their execution rather than a separate feature.
- [RBOC-199783](https://adlm.nielseniq.com/jira/browse/RBOC-199783) — Feature-branch experiment for NodeId validation, stable reason codes and safe logging without raw IDs. The comment reports domain tests/checks and jar publication, but explicitly calls for consumer integration/runtime-path verification; current Jira state is Dev Complete.
- [RBOC-200375](https://adlm.nielseniq.com/jira/browse/RBOC-200375) — Cross-repository DTO metadata reflection experiment with resolver/allowlist guards and a fact utility endpoint. Comments report branch PRs, jar handoff, deployment and telemetry checks, with observed non-blocking warnings; the ticket is now Withdrawn, so do not equate that experiment with current main.
- [RBOC-200967](https://adlm.nielseniq.com/jira/browse/RBOC-200967) — Reports deploying/testing Coralogix-based CPU and JVM scaling across several non-production environments and preparing the deployment PR. The screenshots use deliberately low test thresholds in some environments; those values should not be treated as production policy.
- [RBOC-200968](https://adlm.nielseniq.com/jira/browse/RBOC-200968) — Extended agent instructions to dimension and multi-repository work, including task routing and handoff, and exercised transaction-ID and missing-stack experiments. The referenced RBOC-201339 and RBOC-201990 tickets were later withdrawn.
- [TESTPJ-33](https://adlm.nielseniq.com/jira/browse/TESTPJ-33) — Unassigned dummy test-project issue, Created, with no description/comments. It is included because the user reported it; no Unified Services deliverable is evidenced.
- [RBOC-201339](https://adlm.nielseniq.com/jira/browse/RBOC-201339) — Withdrawn dimension experiment to generate descriptive fallback transaction/group IDs while preserving supplied headers. The comments report a dev deployment and sanity pass with partial Coralogix data; they do not prove this behavior is on main.
- [RBOC-201379](https://adlm.nielseniq.com/jira/browse/RBOC-201379) — Unassigned tracking issue for the dimension-only RBOC-201339 exercise; comment and screenshots describe generated fallback IDs rather than a separate implementation.
- [RBOC-201380](https://adlm.nielseniq.com/jira/browse/RBOC-201380) — Unassigned tracking issue pointing to the RBOC-201990 multi-repository missing-stack experiment; no independent implementation detail.
- [RBOC-201678](https://adlm.nielseniq.com/jira/browse/RBOC-201678) — Validated entitlement cases using fact/DMDS and coordinated adapter ownership. The user recorded requests/responses for seven selection scenarios, identified a missing selection type/group ID, and reported `ALL`/`BETWEEN` failures. Some attached top-level `SUCCESS` responses contain failed individual subrequests.
- [OPTIQ-213](https://adlm.nielseniq.com/jira/browse/OPTIQ-213) — A POC UI validation ticket with a September matrix of 17 structured POD scenarios, paired simple/complex request examples and Discover/POC screenshots. The ticket is broad and long-lived; the matrix documents examples, not automated coverage of every possible case.
- [RBOC-201990](https://adlm.nielseniq.com/jira/browse/RBOC-201990) — Withdrawn multi-repository proposal to infer missing stack headers from link/deployment context with a final default. It sets compatibility and logging guardrails but has no comments establishing a delivered production change.
- [RBOC-202351](https://adlm.nielseniq.com/jira/browse/RBOC-202351) — Fact QA sync/BI-package timeout investigation traced to an unknown serialized field and adapter-version mismatch. The user reports a domain mapper change, consumer jar updates and QA deployment; QA separately reports successful manual/pipeline verification.
- [RBOC-202841](https://adlm.nielseniq.com/jira/browse/RBOC-202841) — User-reported entitlement gap: `ALL` generated no values needed by the DMDS validation flow. Another assignee reports the fix validated; the original description and screenshot do not specify the final implementation.
- [RBOC-202843](https://adlm.nielseniq.com/jira/browse/RBOC-202843) — User-reported entitlement gap: `BETWEEN` was passed in a shape DMDS could not consolidate. Another assignee reports it working after changes.
- [RBOC-203110](https://adlm.nielseniq.com/jira/browse/RBOC-203110) — Compared Discover selections/BI requests to POC-generated payloads for POD facts, product, retailer, demographics and people groups. Comments record naming/attribute mismatches and analysis of conversion, contribution and loyalty before assisting simple-payload design.
- [RBOC-203273](https://adlm.nielseniq.com/jira/browse/RBOC-203273) — Added fact Pluto error-handling assets for sync and flat requests missing `reportItems`, expecting a 400/error code 2004. The user reports PR review/build success and merge to the Pluto test branch; this was test collateral, not a production behavior change.
- [RBOC-204281](https://adlm.nielseniq.com/jira/browse/RBOC-204281) — Validated entitlement changes in dev/local across LLD, Activate, Walmart, compact and encoded scenarios. The user also identified a duplicate-key counter issue and checked `ALL`, `ALL EXCEPT` and `BETWEEN`.
- [RBOC-205194](https://adlm.nielseniq.com/jira/browse/RBOC-205194) — Tracking item for Pluto test creation through AI SDLC. No description or comments provide a separate result; related detail is in RBOC-203273.
- [RBOC-205446](https://adlm.nielseniq.com/jira/browse/RBOC-205446) — Found that regional DMDS routing and BI-package callbacks could target the wrong region for a US-origin/EU-target case. The comment proposes gateway URLs and callback routing changes; subsequent work is recorded in RBOC-206642.

### 19.3 June-July: selection fixes, routing and testing

- [RBOC-206619](https://adlm.nielseniq.com/jira/browse/RBOC-206619) — Investigated compact hierarchy entitlement failures against structured/encoded equivalents. The user reports adapting the BI adapter to read hierarchy selections from both `filterValues` and `values`, testing LLD/Activate/Walmart cases, and a later merged adapter PR.
- [RBOC-206621](https://adlm.nielseniq.com/jira/browse/RBOC-206621) — Reported `EXCLUDE COLUMN` failures. The user's follow-up says the LLD example lacked selection type/IDs and was corrected with a POC-generated payload; another assignee subsequently confirmed the corrected request.
- [RBOC-206622](https://adlm.nielseniq.com/jira/browse/RBOC-206622) — Investigated `HIDE COLUMN` in structured and compact examples. The user reports the same missing selection metadata in the LLD payload, correction and successful DMDS/sync checks.
- [RBOC-206623](https://adlm.nielseniq.com/jira/browse/RBOC-206623) — Reported an `EXCLUDE TOTALS` discrepancy between LLD and Walmart examples. Another assignee concluded the payload was the issue and posted a working request; the record does not show a distinct production-code fix by the user.
- [RBOC-206640](https://adlm.nielseniq.com/jira/browse/RBOC-206640) — Supported POD people-group simple-payload design by comparing candidate fields with BI payloads and discussing them with the designer; the ticket does not claim ownership of the complete design.
- [RBOC-206642](https://adlm.nielseniq.com/jira/browse/RBOC-206642) — Implemented and validated multiregion behavior: DMDS/entitlement gateway URLs, target-stack headers, dimension cache control and US/EU endpoint scenarios. The user later reports fact validation from USQA to EU with a feature branch/tag; both success and earlier `GET_DATAVIEWS` failures are recorded.
- [RBOC-206809](https://adlm.nielseniq.com/jira/browse/RBOC-206809) — Dimension Pluto tests for absent authentication and absent transaction ID, with expected 401 and 400 error contracts respectively. The user reports a PR with two negative specs and request fixtures.
- [RBOC-206816](https://adlm.nielseniq.com/jira/browse/RBOC-206816) — AI-SDLC tracking ticket pointing to the same dimension missing-header Pluto PR as RBOC-206809, not an additional implementation.
- [RBOC-206822](https://adlm.nielseniq.com/jira/browse/RBOC-206822) — Initial JMeter CI workflow: gathered PerfLite guidance, created a feature branch/JMX assets, worked on token generation and added a run summary; the final comment still described further integration work.
- [RBOC-210324](https://adlm.nielseniq.com/jira/browse/RBOC-210324) — Follow-on performance workflow: configured JMeter listener, InfluxDB and Grafana, reconciled outdated configuration guidance and reports merged fact workflow PRs including an input-reporting step.
- [RBOC-210327](https://adlm.nielseniq.com/jira/browse/RBOC-210327) — Production alerts/dashboard corrections and Coralogix/Datadog parity. The user reports 333 versus 380 logs for a sampled transaction, including duplicates/additional event and diagnostic logs, and a corrected per-pod thread-count query; those counts are for that one investigation.
- [RBOC-210538](https://adlm.nielseniq.com/jira/browse/RBOC-210538) — Added a Pluto product share-fact scenario and fixture with an expected response example. The user reports green PR/security checks and an MCP connectivity investigation; the cited final comment says ready for merge, not necessarily independently verified merged.
- [RBOC-210625](https://adlm.nielseniq.com/jira/browse/RBOC-210625) — Tracking ticket repeating the product share-fact Pluto PR and checks from RBOC-210538; avoid counting it as a second distinct test implementation.
- [RBOC-210627](https://adlm.nielseniq.com/jira/browse/RBOC-210627) — Prepared existing refresh-date changes across domain and fact, validated against Discover/Activate payloads, added tests and reports merged PRs in both repositories.
- [RBOC-210641](https://adlm.nielseniq.com/jira/browse/RBOC-210641) — Used the POC UI against production fact/dimension APIs. The user's evidence covers RMS, multicountry, OmniSales, on-shelf availability and Walmart across listed regions; POD was explicitly excluded from the task scope.
- [RBOC-214439](https://adlm.nielseniq.com/jira/browse/RBOC-214439) — Investigated and configured synthetic monitoring. The July update reports non-production checks in Checkly and production checks in Datadog, with production migration planned; later monitoring updates belong to OPTIQ-548.
- [RBOC-214567](https://adlm.nielseniq.com/jira/browse/RBOC-214567) — Added the market share-fact Pluto spec/fixture using the existing share-test branch, with a reported successful PR build and no application source change.
- [RBOC-214712](https://adlm.nielseniq.com/jira/browse/RBOC-214712) — Introduced a shared REST invocation service/configuration in domain, incorporated review feedback and reports the PR merged/validated in QA.
- [RBOC-216555](https://adlm.nielseniq.com/jira/browse/RBOC-216555) — Updated multiregion domain and consumer changes to use the invocation bean. Comments identify dimension PRs and a fact PR, plus troubleshooting of `sync/biPackage` compaction and US/EU response comparison.
- [RBOC-216562](https://adlm.nielseniq.com/jira/browse/RBOC-216562) — Dedicated fact multiregion test placeholder, Not Started and without description/comments. Related tested paths appear in RBOC-206642 and RBOC-216555, but this ticket has no completion evidence.
- [RBOC-216586](https://adlm.nielseniq.com/jira/browse/RBOC-216586) — Dedicated dimension multiregion test placeholder, also Not Started with no description/comments.
- [RBOC-216645](https://adlm.nielseniq.com/jira/browse/RBOC-216645) — POC UI transaction-ID input for triage. The user describes suffix counters that distinguish calls and report executions, logs for each outbound call and an updated POC reference package.
- [RBOC-217225](https://adlm.nielseniq.com/jira/browse/RBOC-217225) — Investigated POD selection calls in POC UI, dimension Swagger and Discover, then reports adding PPL selections to the UI with screenshot evidence.
- [RBOC-217696](https://adlm.nielseniq.com/jira/browse/RBOC-217696) — Follow-on PPL implementation: generated a compact Bought expression referencing a buyer-group item, tested build-report/sync, refactored PPL logic into separate files, and worked on Shopped At and Conversion.
- [RBOC-217698](https://adlm.nielseniq.com/jira/browse/RBOC-217698) — Added a fact BI-package Pluto endpoint alias and a missing-transaction-ID spec, with a reported merged PR. The ticket's sample response describes missing authentication instead, so the exact intended assertion must be checked in the test itself.
- [RBOC-218218](https://adlm.nielseniq.com/jira/browse/RBOC-218218) — Generated EU performance payloads at 0.5M, 1M, 1.5M, 2M and 2.5M cells plus smaller US/EU cases, shared them with the performance team and reports their tests were good.
- [RBOC-218219](https://adlm.nielseniq.com/jira/browse/RBOC-218219) — Determined that `SHR_DIM` in dimension `GET_NODE_VALUES` attributes identifies the dimension to use for a share-fact selection; the comment compares this with Discover behavior.
- [RBOC-218220](https://adlm.nielseniq.com/jira/browse/RBOC-218220) — Initial investigation into how Discover identifies invalid selections. Its only user update says they examined selections and consulted another engineer; more detail follows in RBOC-219053.
- [RBOC-218675](https://adlm.nielseniq.com/jira/browse/RBOC-218675) — Okta-to-Ping support: reports security/configuration updates and successful fact endpoints without downstream calls, while DMDS/BI-dependent endpoints still failed because those services had not yet migrated.
- [RBOC-219053](https://adlm.nielseniq.com/jira/browse/RBOC-219053) — Traced Discover context-carry validation through consolidated items and planned a fact-side selection-validation flow. The user reports BI adapter changes, a fact feature branch and a sample invalid-selection response; final deployment is not established by this ticket.
- [RBOC-219054](https://adlm.nielseniq.com/jira/browse/RBOC-219054) — Added a Pluto test for missing access token on fact `sync/biPackage`, reusing the existing fixture and expecting 401/code 2000. The user reports QA validation and merge.
- [OPTIQ-321](https://adlm.nielseniq.com/jira/browse/OPTIQ-321) — Symbio SDLC rollout: integrated Coralogix MCP/query reference material and reports restructuring the workflow into sequential subagents and skills on an AI-adoption branch.
- [OPTIQ-713](https://adlm.nielseniq.com/jira/browse/OPTIQ-713) — Investigated LaunchFlow and performed a sample POC UI deployment. The user considered it useful for short-lived MVPs but noted missing integration with existing deployment/secrets processes and one-hour sandbox lifetime at the time.
- [OPTIQ-719](https://adlm.nielseniq.com/jira/browse/OPTIQ-719) — Symbio RAG rollout, currently owned by another engineer. Their comment reports ingesting roughly 30 Unified Services Confluence documents and querying an architecture summary; the ticket does not attribute that ingestion to the user.
- [OPTIQ-720](https://adlm.nielseniq.com/jira/browse/OPTIQ-720) — Triage onboarding: the user reports creating an integration group and fact/dimension service maps with callers, downstream dependencies and symptoms. The triage team was to complete its own onboarding steps.
- [RBOC-219319](https://adlm.nielseniq.com/jira/browse/RBOC-219319) — Coordinated with the Omni team on template-to-model mapping and documented how model IDs can be found in BI payloads; the mapping was initially incomplete pending team documentation.
- [RBOC-219498](https://adlm.nielseniq.com/jira/browse/RBOC-219498) — Surveyed Symbio SDLC, code-quality instrumentation, LaunchFlow, RAG and Triage. The user reports creating a test RAG project with repository/Jira/Confluence sources and starting Triage setup.
- [RBOC-219616](https://adlm.nielseniq.com/jira/browse/RBOC-219616) — Shared validation of RMS templates against Discover and Unified/POC UI. Teammate comments provide multiple template payloads/screenshots; the user's contributions include analyzing Discover async requests and documenting comparisons, including business-driver examples.

### 19.4 August-September: templates, POD and security preparation

- [RBOC-219870](https://adlm.nielseniq.com/jira/browse/RBOC-219870) — RMS template validation support ticket with no description/comments. Its parent and RBOC-219616 provide related context, but no separate contribution or outcome is documented here.
- [RBOC-220265](https://adlm.nielseniq.com/jira/browse/RBOC-220265) — Shared performance-regression analysis, currently assigned to another engineer. The described EU PERF run used sync and flat traffic for one hour; their comments report generally acceptable metrics and one failed request, not a user-authored analysis.
- [RBOC-220276](https://adlm.nielseniq.com/jira/browse/RBOC-220276) — User-reported investigation into passing POD group-level items as characteristics. The current assignee reports consulting another team; no resolution details are documented.
- [OPTIQ-548](https://adlm.nielseniq.com/jira/browse/OPTIQ-548) — Cost and infrastructure stability: the user reviewed fact/dimension workload cost views across production regions and QA, reports production alerts and later Checkly synthetic monitors. The screenshots are evidence of the historical assessment, not a quantified cost saving.
- [OPTIQ-553](https://adlm.nielseniq.com/jira/browse/OPTIQ-553) — Compared shifting-template Discover and POC BI payloads. Found a model-ID mismatch, extra dimensions and fixed items depending on report-module prompts; later BI changes exposed a DDR/dot mismatch and SQL failure, requiring BI/Aggregation Engine collaboration.
- [OPTIQ-681](https://adlm.nielseniq.com/jira/browse/OPTIQ-681) — Investigated QA timeouts and reports a blocking `BICacheService` issue, a warmup initializer and a JDK/library combination that helped. The user expressly cautions that further issues may remain.
- [OPTIQ-703](https://adlm.nielseniq.com/jira/browse/OPTIQ-703) — BI handshake for leakage analysis: requested a working payload, comparison and any needed fact changes. The only user update says BI planned related changes in an upcoming sprint; a Done status alone is not evidence that every leakage scenario was validated.
- [OPTIQ-706](https://adlm.nielseniq.com/jira/browse/OPTIQ-706) — Compared shared and generated payloads for shifting/leakage, reports domain changes and a `filterWithValues` fix. The October update still needed dynamic template-to-model-ID resolution and end-to-end validation; a later comment accepted the work based on demo.
- [OPTIQ-710](https://adlm.nielseniq.com/jira/browse/OPTIQ-710) — Reports support for additional shifting/leakage dimensions, selectable `modelId` with `MKSE` default and model-specific `filterWithValues`. Separate domain PRs are linked for simple/compact and encoded paths; attached request/complex examples include `BSF-P` and retailer variants.
- [OPTIQ-1065](https://adlm.nielseniq.com/jira/browse/OPTIQ-1065) — POC UI enhancements and a documented matrix for bought, shopped, conversion, deal, loyalty, frequency and quantity cases, with paired simple/complex examples. The September update excludes non-conversion/contribution from completed verification pending fact-service changes despite UI work; the ticket was later Accepted.
- [OPTIQ-1102](https://adlm.nielseniq.com/jira/browse/OPTIQ-1102) — Diagnosed Walmart Price Beats items not being added: plain labels beginning `%` were interpreted as jQuery selectors and threw during tree selection. The user reports a targeted text-extraction fix and validates selection/report generation.
- [OPTIQ-1142](https://adlm.nielseniq.com/jira/browse/OPTIQ-1142) — Ran a specified JDK/security-library load-test matrix with increasing request counts and collected a thread dump. The user reports JDK 25 timeouts at the stated 150-request/30-second ramp and an occupied Hazelcast queue, without claiming a proven universal cause.
- [OPTIQ-1203](https://adlm.nielseniq.com/jira/browse/OPTIQ-1203) — **Newly found through comments; Accepted, assigned to another engineer.** Scope was entitlement checks for condition cases in POD and multicountry. The user reports instrumenting the BI adapter to inspect DMDS requests/responses and following up with the DF team on SQL failures; the accepting comment refers to a demo.
- [OPTIQ-1388](https://adlm.nielseniq.com/jira/browse/OPTIQ-1388) — User-reported but unassigned, automatically created critical `StackOverflowError` candidate, later Withdrawn. The source explicitly says no stack trace was captured, so it cannot establish a root cause.
- [OPTIQ-1453](https://adlm.nielseniq.com/jira/browse/OPTIQ-1453) — **Newly found through comments; Done, assigned to another engineer.** The user found encoded PPL items lacked filters, so dimension resolution returned `UNKNOWN` and the PPL expression was not built; they report returning `PPL` when an `expressionGroup` exists on branch `feature/encodedExp`. The attached simple example contains a contribution expression without a PPL filter; the paired complex example contains a PPL selection, but those examples alone do not prove a production rollout.
- [OPTIQ-1699](https://adlm.nielseniq.com/jira/browse/OPTIQ-1699) — Logging standardization across fact/dimension main and deployment branches. The user reports moving to the newer telemetry resource schema, removing old Datadog agent/pattern pieces and checking QA for duplicate and downstream logs; higher-environment rollout was planned later.
- [OPTIQ-1732](https://adlm.nielseniq.com/jira/browse/OPTIQ-1732) — User-reported encoded POD validation request, assigned to a teammate. It asks for PPL expression verification except non-conversion and contains exported POC payloads; the teammate's comment does not list a complete pass/fail matrix.
- [OPTIQ-1733](https://adlm.nielseniq.com/jira/browse/OPTIQ-1733) — Reports adding Gradle tests to the domain jar build and fact/dimension CI/image builds, with three PRs and sample workflow runs.
- [OPTIQ-1805](https://adlm.nielseniq.com/jira/browse/OPTIQ-1805) — Fact security-score ticket currently assigned to another engineer, who reports SAST work and escalation of an old workflow token finding. Do not attribute that work solely to the user.
- [OPTIQ-1808](https://adlm.nielseniq.com/jira/browse/OPTIQ-1808) — User reports obtaining Cycode access, reviewing unused branches and fixing dimension main/release findings in a feature branch; no detailed scan inventory is in this ticket.
- [OPTIQ-1809](https://adlm.nielseniq.com/jira/browse/OPTIQ-1809) — Dimension security-score parent. The user reports resolving secrets/SAST/SCA findings, references subtasks, records a historical score and notes DAST was pending the scanner migration/access at that time.

### 19.5 October-November: current and follow-on work

- [OPTIQ-1954](https://adlm.nielseniq.com/jira/browse/OPTIQ-1954) — Dimension/POC UI SAST work included replacing insecure random generation in UI branches, consolidating stale-branch changes and a dimension main PR. The user reported zero SAST violations in the cited historical scan.
- [OPTIQ-1955](https://adlm.nielseniq.com/jira/browse/OPTIQ-1955) — POC UI SCA remediation and stale-branch cleanup. The user reports updating dependency versions and a zero SCA finding count at that time; it is not a permanent guarantee.
- [OPTIQ-1956](https://adlm.nielseniq.com/jira/browse/OPTIQ-1956) — Investigated old Artifactory-token findings attributed to historical workflows and coordinated with the Cycode team. The final comment reports those violations resolved through a service request; no credential values are reproduced here.
- [OPTIQ-2042](https://adlm.nielseniq.com/jira/browse/OPTIQ-2042) — POC UI support for additional POD/non-converted-shopper scenarios, fixing folder selection and error propagation behavior, and updating workflow runners; reported validated by the user.
- [OPTIQ-2043](https://adlm.nielseniq.com/jira/browse/OPTIQ-2043) — Fact/dimension product security score, Ready for UAT in this snapshot. The user reports fewer findings and escalations but calls out new Cycode findings and an old connection URL requiring follow-up.
- [OPTIQ-2045](https://adlm.nielseniq.com/jira/browse/OPTIQ-2045) — Shared buyers/shoppers template validation, currently assigned to another engineer, whose comment enumerates ten available templates. The ticket does not contain user-authored results for each template.
- [OPTIQ-2205](https://adlm.nielseniq.com/jira/browse/OPTIQ-2205) — Replaced dynamic `latest.release` dependencies with explicit Commons versions and aligned Tomcat/Jackson versions in domain/fact PRs. The ticket records the proposed historical versions; use the build files, not these values, for current dependency state.
- [OPTIQ-2241](https://adlm.nielseniq.com/jira/browse/OPTIQ-2241) — Activate large-report investigation: downloads of BI-package/metadata blobs succeeded, while flat conversion hit null metadata. The user identified a hash-key collision between reference and metadata entries, notified BI and linked subsequent fix/validation tickets; large blob contents were not independently reviewed here.
- [OPTIQ-2243](https://adlm.nielseniq.com/jira/browse/OPTIQ-2243) — Parent T&D requirement to select product groups in POC UI before generating the expected product/market values. The user points to OPTIQ-2533 and OPTIQ-2534 for actual implementation and validation.
- [OPTIQ-2275](https://adlm.nielseniq.com/jira/browse/OPTIQ-2275) — Validated `pgKeys` for currency `GET_NODE_CHILDREN`/`GET_NODE_VALUES` against Discover; reports empty results without keys and matching results with them. The user attributes the underlying domain metadata-query change to another engineer.
- [OPTIQ-2305](https://adlm.nielseniq.com/jira/browse/OPTIQ-2305) — Validated the large-selection metadata collision scenario after a BI adapter change. The user links the prior analysis and the other engineer's fix PR and reports a sample validation; detailed independent blob verification was not performed.
- [OPTIQ-2330](https://adlm.nielseniq.com/jira/browse/OPTIQ-2330) — User-reported Wiz remediation ticket assigned to a teammate. Their updates describe dependency upgrades, fact/dimension dev testing and remaining vulnerabilities in upstream NIQ or embedded libraries; the reported blockers and CI issues should not be attributed as the user's own changes.
- [OPTIQ-2475](https://adlm.nielseniq.com/jira/browse/OPTIQ-2475) — User reports UI HTML sanitization, Cycode follow-up, and restricting dimension health-endpoint CORS after DAST findings. Authenticated scanning remained limited by scanner setup, and revocation of an old connection URL was planned; the planned date alone is not confirmation of revocation.
- [OPTIQ-2533](https://adlm.nielseniq.com/jira/browse/OPTIQ-2533) — Added product-group listing/selection, stored selected keys in UI state, propagated them to later dimension/fact requests and warned when required selections were missing. The comment reports validation across calls and datasets.
- [OPTIQ-2534](https://adlm.nielseniq.com/jira/browse/OPTIQ-2534) — End-to-end product-group validation: compared requests with/without `pgKeys` in Swagger, POC UI and Discover, checked persistence/editing and unrelated solution types, and reports compact, encoded and structured fact cases.
- [OPTIQ-2673](https://adlm.nielseniq.com/jira/browse/OPTIQ-2673) — Further fact/dimension security-score ticket, Not Started with no description/comments. Related earlier security work does not prove this task was completed.
- [OPTIQ-2726](https://adlm.nielseniq.com/jira/browse/OPTIQ-2726) — In-progress parent for Optiq v1 use-case discovery, validation and gap identification. A teammate provided the `E2E_Optiq_Web` issue-set query; the ticket itself does not yet list a completed validation matrix.
- [OPTIQ-2959](https://adlm.nielseniq.com/jira/browse/OPTIQ-2959) — In-progress child for gathering Optiq flow/use-case information. No description/comments give further progress.
- [OPTIQ-2960](https://adlm.nielseniq.com/jira/browse/OPTIQ-2960) — Not Started child to validate Optiq v1, important v2 and USS use cases, with a link to the E2E-labelled v1 set; no test outcomes recorded.
- [OPTIQ-2962](https://adlm.nielseniq.com/jira/browse/OPTIQ-2962) — In-progress POD condition task. The user records a custom condition with rule causing a fact DF error and attaches Discover, fact-generated and simple payloads; screenshot and payload differences are summarized in section 18.4, but the root cause remains unconfirmed.
