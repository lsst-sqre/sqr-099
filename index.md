# Breakdown of adaptations to the CADC TAP Service for the RSP

```{abstract}
This technote details the modifications made to the CADC TAP service for the Rubin Science Platform, covering both QServ and PostgreSQL implementations. 

We highlight which components of the upstream codebase we adapt, add implementations to, or replace, as well as breakdown whether these would be required assuming we move towards a more event-based architecture.
```


## Overview

This document provides an overview of the modifications made to the CADC Table Access Protocol (TAP) service for implementation within the Rubin Science Platform (RSP). 
The RSP requires adaptations to work within our infrastructure and technical requirements.

We currently maintain two separate codebases that extend the upstream CADC TAP repository:

- **QServ-backed TAP services** ([lsst-sqre/lsst-tap-service](https://github.com/lsst-sqre/lsst-tap-service)): This implementation focuses on integrating with QServ, our distributed database system.
- **PostgreSQL-backed TAP services** ([lsst-sqre/tap-postgres](https://github.com/lsst-sqre/tap-postgres)): This implementation provides TAP services backed by standard PostgreSQL databases. While it shares many components with the QServ implementation, it has its own specific requirements, particularly around use of pgsphere and support for TAP_UPLOAD.

Both implementations have been adapted to work within our Google Cloud Platform infrastructure, and specifically rely on use of Google Cloud Storage (GCS) for temporary results storage.

## Key Modifications (lsst-tap-service)

### 1. TableWriter Class Modification

**Modified Behavior:**

Our implementation extends the TableWriter to support writing temporary results to Google Cloud Storage (GCS).

**Rationale:**

Our infrastructure runs in Google Cloud Platform, so this modification allows our service to store temporary query results in GCS buckets.
Our adaptation here also generates meta resources for datalinks, which our determined through the use of a datalink-manifest.json file,
which is fetched from https://github.com/lsst/sdm_schemas/ at runtime.

**Implementation Details:**
- `public class ResultStoreImpl implements ResultStore`
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/ResultStoreImpl.java

- `public ResultSetWriter implements TableWriter<ResultSet>`
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/ResultSetWriter.java

- `public class RubinTableWriter implements TableWriter`
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/RubinTableWriter.java

### 2. Adding Log4j2

**Original Behavior:**

Main repo uses log4j

**Modified Behavior:**

Enabled log4j2
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/resources/log4j2.xml

**Rationale:**

Change was needed because we started using Sentry for reporting metrics, errors and traces, which requires Log4j2.

### 3. Formatter for modifying ObsCore access_urls

**Modified Behavior:**

Add a formatter to overwrite the access_url field for results from ObsCore queries.
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/RubinFormatFactory.java
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/RubinURLFormat.java

**Rationale:**

Main reason for this addition is to be able to format/modify the results of queries to ObsCore that contain access_urls which link to our datalinker service, to point to the base_url that matches what our TAP service is running on, available in a base_url system property.

### 4. Add implementation (AdqlQueryImpl) of AdqlQuery

**Modified Behavior:**

Added an implementation of AdqlQuery to add support for QServ user defined functions.
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/AdqlQueryImpl.java

**Rationale:**

We need to add the QServRegionConverted to the navigatorList

**Implementation Details:**

Added the following to AdqlQueryImpl.java:
```java
super.navigatorList.add(new QServRegionConverter(new ExpressionNavigator(), new ReferenceNavigator(), new FromItemNavigator()));
```

### 5. ALMATableServlet

**Modified Behavior:**

ALMATableServlet class seems similar to upstream, we may not need this implementation: 
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/ALMATableServlet.java

### 6. Customized /capabilities endpoint

**Modified Behavior:**

- Add CapGetAction implementation:
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/CapGetAction.java
- Modify capabilities.ml to specify list of securityMethods supported, list of ADQL Geometry functions supported & whether TAP_UPLOAD is supported:
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/webapp/capabilities.xml
- Add CapGetInitAction implementation: (Looking at this again, I'm not sure if this one is needed):
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/CapInitAction.java

**Rationale & Implementation Details:**

Add path_prefix to Capabilities endpoints (CapGetAction)
```java
private static final String pathPrefix = System.getProperty("path_prefix");
..
String npath = path.replace(basePath, pathPrefix);
```

The reason for this is that we wanted to be able to modify what the path prefix is for each endpoint in the capabilities for a given TAP service. (/tap, /ssotap, /consdbtap). In our configuration this is defined in our Helm charts and passed along a system property.
There may be a better way to do this in Java/Tomcat.

### 7. Implementation of MaxRecValidator

**Modified Behavior:**

We need to revisit why this was added and whether it is still needed, as it matches the upstream ALMA MaxRecValidator.
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/MaxRecValidatorImpl.java

### 8. QServ ADQL functions

**Modified Behavior:**

Added the following implementations:
- QServCircle.java (Function):
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/QServCircle.java
- QServPolygon.java (Function):
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/QServPolygon.java
- QServPoint.java (Function):
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/QServPoint.java
- QServRegion.java:
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/QServRegion.java
- QServRegionColumn.java:
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/QServRegionColumn.java
- QServRegionConverter.java (RegionFinder):
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/org/opencadc/tap/impl/QServRegionConverter.java

**Rationale:**

Needed in order to support the ADQL Geometry in Qserv.

### 9. Overwrite QueryRunner with QServQueryRunner

**Modified Behavior:**

Add QServQueryRunner in our repo which overwrites QueryRunner to support our customization related to writing async results to GCS, Sentry reporting, handling max rows, changing behaviour of some of the logging, changing the fetch size and adding the qservDataSourceName.
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/QServQueryRunner.java

**Rationale:**

The rationale for the customization mainly has to do with our use of QServ, Sentry, GCS for storing results, and some changes needed to how we want certain errors logged and reported.

### 10. Added ResultsServlet

**Original Behavior:**

Since we write results to a GCS bucket, originally the results for a UWS job were just a link to that GCS file.

**Modified Behavior:**

We've since modified this architecture to instead introduce a Results Servlet that redirects the user to the actual bucket with the results.
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/ResultsServlet.java

We also had to modify our ResultStoreImpl to point results to this servlet:
https://github.com/lsst-sqre/lsst-tap-service/blob/5f687f91ce0968e6eef9062b5204baa607f8f0c7/src/main/java/org/opencadc/tap/impl/ResultStoreImpl.java#L176

**Rationale:**

The main reason for this was because when using pyvo previously, users ran into an issue where the requests library failed to redirect the user to a URL outside the domain of the original request.

### 11. Added our own RubinRegistryServlet servlet

**Original Behavior:**

The TAP service will normally use a service registry endpoint which is located at a CADC webserver.

**Modified Behavior:**

We've modified this to introduce a small servlet that just returns an empty file and added an entry in our configmap to use this endpoint:
```
ca.nrc.cadc.reg.client.RegistryClient.baseURL = https://data-dev.lsst.cloud/api/tap/reg
```
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/RubinRegistryServlet.java

**Rationale:**

Our main rationale for this was we wanted to remove our dependency from an endpoint that is outside our control.

### 12. Sentry for reporting and metrics

**Modified Behavior:**

Added Sentry support which required modifying:
- build.gradle (Import dependencies)
- Add calls in QServQueryRunner to initialize and trace the execution of a query, as well as report errors in Sentry:
https://github.com/lsst-sqre/lsst-tap-service/commit/a23eb2544f7b6ce8341f59bb3320ad7ffb668e61

**Rationale:**

We wanted to allow the TAP Service to report errors as well as trace certain paths and their execution durations in the QueryRunner. Since we've started using Sentry for other projects it was the preferred option to use here as well.

### 13. Bootstrapping UWS database with UWSInitAction

**Modified Behavior:**

We've added the UWSInitAction class which initializes the UWS schema and tables if they do not exist:
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/UWSInitAction.java

**Rationale:**

We wanted to move control of our UWS schema to the TAP service.

### 14. Overwrite ALMATapSchemaDAO

**Modified Behavior:**

Added ALMATapSchemaDAO class which adds list of sql functions specific to our QServ back-end:
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/ALMATableServlet.java

### 15. Add implementation of AvailabilityPlugin

**Modified Behavior:**

Added TAPWebService class which is an implementation of AvailabilityPlugin and adds a validation check which runs a TAP_SCHEMA query:
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/ws/TAPWebService.java#L74

**Rationale:**

Customizing how we validate whether the service is available, by running a query to TAP_SCHEMA which validates that the service is up-and-running.

### 16. Extend SimpleJobManager class

**Modified Behavior:**

Add an extension to the SimpleJobManager, where we pass in as parameters our JobPersistence object, QueryRunner and threadpool configuration:
```java
JobPersistence jobPersist = new PostgresJobPersistence(new RandomStringGenerator(16), im, true);
final JobExecutor jobExec = new ThreadPoolExecutor(jobPersist, QServQueryRunner.class, 6);
```
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/ws/QueryJobManager.java

**Rationale:**

Mainly for customizing the QueryRunner and persistence object, also may be used to modify the number of threads used.

### 17. Add CachingFile

**Modified Behavior:**

Added reg.client.CachingFile implementation:
https://github.com/lsst-sqre/lsst-tap-service/blob/master/src/main/java/ca/nrc/cadc/reg/client/CachingFile.java

In this implementation we remove all instances of the checkpoint.

**Rationale:**

Upon adding log4j2, something was misbehaving in our logging, potentially due to a conflict with two different version of log4j, causing us to see checkpoints in the logs for certain files even though the logget configuration was set to INFO. This was a temporary workaround which we wanted to revisit after better understanding the conflicts. 

### 18. Overwrite the JobDAO class

**Modified Behavior:**

Override the JobDAO class to fix issues with async job lists (Using LAST param not showing correct results):
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/ca/nrc/cadc/uws/server/JobDAO.java

Also remove checkpoints which appear in our INFO logs.

**Rationale:**

The rationale for overwriting this was initially to add a bugfix to the async job list mentioned above. We've since also removed the checkpoints which appear in our INFO logs, potentially due to an issue caused by us introducing log4j2.

One additional reason where this may require a specific implementation in our codebase unless this can be modified via configuration in the future, is to allow synchronous queries to appear in the UWS job list. The reason for this is related to providing users with query history via this endpoint, where we may want to aggregate and display all of a user's queries there. This is currently in a PR pending further discussion on whether this is the right approach and how we plan on allowing filtering out of Firefly "system" sync queries to TAP_SCHEMA.

### 19. Binary2 support

**Modified Behavior:**

The changes required to enable Binary2 serialization can be found in the following PR: 
https://github.com/lsst-sqre/lsst-tap-service/commit/5d816bc9f9bd57053b91790462dac45f7dd51516

This utilizes the Starlink library and introduces a ResultSetWriter which implements the TableWriter<ResultSet>.
We modify the RubinTableWriter to add two different cases, one for the default VOTable which will use our ResultSetWriter implementation and one for a version that uses TABLEDATA, which can be requested by specifying the result format:

```java
case VOTABLE:
    resultSetWriter = new ResultSetWriter();

..

case VOTABLE_TD:
    voDocumentWriter = new VOTableWriter();
```

**Rationale:**

We wanted to introduce Binary2 as the default serialization of our VOTable results, as we expect queries to potentially return a large volume of data which would benefit from a more efficient serialization than the standard TABLEDATA format used by default in the TAP Service.

### 20. Added ALMATableServlet.java

**Modified Behavior:**

This class seems to match what is upstream, so it's likely that this is not needed:
https://github.com/lsst-sqre/lsst-tap-service/blob/2bdc7acad99cafa1d3ff4d00919141ff92f04340/src/main/java/org/opencadc/tap/impl/ALMATableServlet.java

## How much of this is needed in our event-based architecture?

Below is a breakdown of the modifications and whether they will still be required after transitioning to the new event-based architecture:

| ID | Modification | Required in Event Architecture? | Notes |
| --- | ------------- | ------------- | --------- |
| 1 | TableWriter (GCS Integration) | No [*] | No longer needed for temporary storage |
| 2 | Log4j2 Implementation | Yes | Still needed for Sentry integration |
| 3 | ObsCore URL Formatter | No | Formatting will happen in the qserv proxy |
| 4 | QServ ADQL Functions | Yes | Required for spatial query support |
| 5 | ALMATableServlet | No | Likely unnecessary duplication |
| 6 | Customized /capabilities endpoint | Yes | Required for path configuration unless we have a better solution |
| 7 | MaxRecValidator | No | Purpose unclear, likely unnecessary |
| 8 | QServ ADQL Functions | Yes | Required for spatial queries |
| 9 | QServQueryRunner | Yes | Changes also needed for event-based architecture |
| 10 | ResultsServlet | Yes | Required |
| 11 | RubinRegistryServlet | Yes | Required to avoid external dependencies |
| 12 | Sentry Integration | Yes | Required for error reporting |
| 13 | UWSInitAction | Yes | Required for UWS schema bootstrapping |
| 14 | ALMATapSchemaDAO | Yes | Required for QServ SQL functions |
| 15 | AvailabilityPlugin | Yes | Unless we want to use upstream Availability WebService |
| 16 | SimpleJobManager Extension | Yes | Assuming we want to specify custom Runner, Persistence and thread count |
| 17 | CachingFile | No | Hopefully unnecessary if logging issues are resolved |
| 18 | JobDAO Override | Maybe | Depends on requirements for synchronous query history |
| 19 | Binary2 Serialization | No | Will be implemented in qserv proxy |
| 20 | ALMATableServlet | No | Likely unnecessary duplication |

[*] In the event-based system We will still to generate the VOTable header which includes all fields and their metadata, as well as other additional metadata like the datalink
section meta resources.
 
## Key Modifications (tap-postgres)

The Github repository for the Postgres backed TAP service can be found at:
https://github.com/lsst-sqre/tap-postgres

While the structure is slightly different than the lsst-tap-service repository (for example our package here is named sample), the changes are for the most part similar:

https://github.com/lsst-sqre/tap-postgres/tree/master/tap/src/main/java/ca/nrc/cadc/sample

The following changes from above have not been made to this repository:

1. Binary2 support has not been added to this version
2. QServ ADQL region functions not applicable here
3. Sentry has not been added here
4. Log4j2 has not been added here

Instead, we have made some changes that do not exist in the QServ (lsst-tap-service) repository:

### 1. Added UploadManager implementation

**Modified Behavior:**

Added implementation of UploadManager for Postgres where it is supported

This works through the use of GCS where the user table is uploaded to:
https://github.com/lsst-sqre/tap-postgres/blob/master/tap/src/main/java/ca/nrc/cadc/sample/UploadManager.java

**Rationale:**

Making Upload available via Qserv has not been possible until recently, but it is possible with Postgres, so we include our implementation here, which requires uploading via GCS.

### 2. Support for Postgres ADQL Geometry

**Modified Behavior:**

Added an ObsCoreRegionConverter:
https://github.com/lsst-sqre/tap-postgres/blob/master/tap/src/main/java/ca/nrc/cadc/sample/ObsCoreRegionConverter.java

**Rationale:**

Turn the s_region columns to the different pgsphere columns that are stored in the backend database.

## Consolidation Strategy

Our goal is to minimize maintenance overhead by:
- Upstreaming modifications where possible
- Reducing implementation duplication between repositories
- Moving toward configuration-based customization rather than code overrides

## Next Steps

- Evaluate which components can be upstreamed to CADC TAP
- Evaluate functionality that would not be needed in the event-based architecture
- Consolidate common functionality between QServ and PostgreSQL implementations
- Replace custom implementations with configuration where possible
    
