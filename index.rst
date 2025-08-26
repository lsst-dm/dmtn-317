####################################################################
Technical Design for the Prompt Products Database (PPDB) in BigQuery
####################################################################

.. abstract::

   This technote presents technical requirements and design notes for the
   Prompt Products Database (PPDB) in BigQuery.

Overview
========

The Prompt Products Database (PPDB) will store Prompt Data Products, as
described in the *Data Products Definition Document* :cite:`LSE-163`, with the
goal of making the data easily accessible for scientific analysis and querying.
The data will be ingested from the Alert Production Database (APDB) with the
current plan being to host the data in Google
`BigQuery <https://cloud.google.com/bigquery>`_.
A comparison of BigQuery with other options was presented at length in
*DMTN-308* :cite:`DMTN-308`, and the result of this analysis and discussion
surrounding it was a decision to proceed with initial development using a
BigQuery-based system.

A significant amount of work has been done to implement data ingestion from the
APDB into BigQuery, as well as to set up a TAP service for querying the data.
As much as possible, this technote will attempt to make it clear which parts of
the design are still under consideration and which have already been
implemented.
Even those aspects that have been implemented are still subject to
change as the design is iterated upon and refined.

Requirements
============

Functional requirements
-----------------------

TODO: Requirements need to be cleaned up as many of them relate to the APDB,
which was conflated with the PPDB in most of these design documents.

The following requirements may be assumed based on Rubin design documents:

1. **Comprehensive Level 1 Data Access:** Provide rapid access to the complete
   history of Level 1 catalogs, including DIASource, DIAObject, SSObject, and
   DIAForcedSource, along with associated metadata and provenance
   :cite:`LSE-163` :cite:`LSE-61`.

2. **Low-Latency User Query Performance:** Support at least 20 simultaneous
   users for live queries against the Prompt Products Database, with each query
   completing within 10 seconds :cite:`LSE-61`.

3. **Public Data Availability:** Ensure that the public-facing
   view of updated Level 1 database content (DIASources, DIAObjects, SSObjects)
   is available within 24 hours of the corresponding observation or orbit
   determination :cite:`LSE-61`.

4. **Reproducible & Complex Querying:** Enable queries on all Level 1 data
   products to be reproducible over time [LDM-555, LSE-61] and support complex
   queries, including spatial correlations and time series comparisons, using
   ADQL (a superset of SQL92) :cite:`LDM-135` :cite:`LSE-61` :cite:`LDM-555`.

5. **High System Availability and Data Integrity:** The database system must
   maintain at least 98% uptime and must not lose data due to hardware/software
   failures, incorporating fault-tolerance features and avoiding extended
   offline periods for maintenance :cite:`LSE-61`. (TODO: Clarify if this is
   may originally have been referring to the APDB.)

6. **Adaptable Database Schema:** The Level 1 database schema must be
   modifiable during the lifetime of the survey without altering query results,
   and data access services must accommodate the evolution of the LSST data
   model across Data Releases :cite:`LSE-61`.

7. **Comprehensive Logging and Reporting:** Every query of LSST databases must
   be logged, including user, query text, start/end times, and results size,
   with history available to the user. Nightly Data Quality, DMS Performance,
   and Calibration Reports must be generated within 4 hours :cite:`LSE-61`.

These should be considered preliminary requirements and are subject to change.
The APDB and PPDB are used interchangeably in some design documents, in
particular within the DPDD, since these were not two distinct systems when
the documents were written.
Further clarification is needed to determine which requirements apply to
which database, though some are shared between both systems.
Finally, some of the requirements are generic ones that apply to any Rubin
database system, though they may still be relevant to the PPDB.

Non-Functional requirements
---------------------------

TODO: This section seems a bit unnecessarily long and detailed and could
probably just be subsumed into the general requirements section.

Data processing
^^^^^^^^^^^^^^^

Creating a robust and error-tolerant data processing pipeline for ingesting
data is critical to the success of the PPDB.

The following non-functional requirements may be assumed in this area:

* Robust error handling should be implemented for each step of the data
  processing pipeline.
  It should be straightforward to identify and retry failed jobs without
  causing duplicate data.

* Processes should be designed to keep up with projected data rates from the
  APDB (TODO: include projections here???). In particular, bottlenecks should
  be avoided for each process, which may involve careful monitoring and
  optimization work.

* The data ingestion pipeline should eventually be designed to handle different
  versions of the schema gracefully and seamlessly. This may involve using
  different BigQuery datasets for different versions of the schema or
  developing tools for converting from one schema version to another.
  Migration tools may also be needed to convert existing data to a new schema.

* Each step of the data processing process should be idempotent, meaning that
  if a step is retried, it will not cause duplicate data or errors.
  Implementing this is highly involved, as it requires careful tracking of
  state and ensuring that each operation can be safely retried. (In particular,
  explicit checks must be performed so that duplicate data is not inserted.)

* Each component in the system should be visible for monitoring and debugging
  purposes, allowing for easy identification of issues and performance
  bottlenecks.
  This should be achievable using logging, metrics, and monitoring tools such
  as Google Cloud Logging or custom dashboards.

The processes which have been implemented so far in the prototype system
generally do _not_ satisfy all of of these requirements; in particular,
achieving an adequate level of idempotency and robustness to failure will
require a significant amount of additional development and testing to achieve.

Database schema
===============

Public scientific databases within the Rubin Observatory are generally
considered to be part of the Science Data Model (SDM) schemas which are
maintained by the `Data Management <https://www.lsst.org/about/dm>`_ group.
Each of these schemas is managed using YAML files in the
`Felis <https://felis.lsst.io>`_ data format within the
`SDM Schemas <https://github.com/lsst/sdm_schemas>`_ repository.
The Felis file defines the schema's database objects (tables, columns, etc.)
in a human-readable way which is independent of any underlying database
technology.
Felis can instantiate these schemas in standard RDMS systems such as Postgres
and MySQL, but in other cases, such as the APDB, the schema's data model is
translated into a different underlying implementation by an external library,
which is Cassandra in that case.

Dataset and table creation
--------------------------

For the PPDB in BigQuery, the APDB's existing YAML file should be used as the
basic "source of truth" for the database schema, but Felis itself does not have
native support for BigQuery as a target database backend.
An additional library will be needed for creating BigQuery datasets and tables
from the schema file programmatically using the BigQuery API
(``google-cloud-bigquery``) or its command-line interface (``bq``).
This library should be implemented in one or more Python modules which can be
used as part of a command-line tool or script.
The schema creation library will need to handle some BigQuery-specific features
such as assignment of partitions and column clustering, which are not part of
the Felis data model.
Some additional columns not defined in the YAML schema may also need to be
to be added for table optimization and management, such as a ``GEOGRAPHY``
column for spatial queries or a derived temporal column for partitioning.
Finally, a schema version will need to be attached to the dataset, as the basic
data ingest must be done in a way that is compatible.
Only APDB data with a matching version can be ingested into the PPDB.
It be useful to encode the schema version directly into the dataset's name,
e.g., ``ppdb_v9_0_0``, and it can also be assigned as a
`label <https://cloud.google.com/bigquery/docs/labels-intro>` on the dataset
using the BigQuery API or ``bq`` command-line tool so that it can be easily
read programmatically.

In addition to handling of the primary dataset(s) containing the productions
tables, it is likely that there will be additional datasets which need to be
created for various purposes such as data staging and snapshot storage.
Keeping all of the tables within the PPDB in a single dataset, even for a
specific schema version, could lead to confusion and clutter, so separating
them out into different datasets by their purpose is advisable.
These can be distinguished by naming conventions, such as
``ppdb_v{version}_staging`` for staging tables or ``ppdb_v{version}_snapshots``
for backup snapshots.
Many operations in BigQuery take advantage of
`zero-copy cloning
<https://cloud.google.com/bigquery/docs/table-clones-intro>`_ operations which
do not require duplicating the underlying data, with subsequent changes to the
cloned table being stored as deltas.
These operations are very fast and efficient, so having multiple datasets
should not cause significant overhead compared with a single one, provided that
the datasets are all in the same GCP project and region.

Extra tables
------------

In addition to the three main production tables that are replicated to the
PPDB from the APDB (``DiaObject``, ``DiaSource``, and ``DiaForcedSource``), it
may be useful to create additional tables for convenience and performance.
The ``DiaObject`` table in particular has a composite primary key consisting of
``diaObjectId`` and ``validityStartMdjTai``, which means that there will be
many records with the same ``diaObjectId`` value.
This is likely to be be confusing for a typical science user, who may expect
this ID column to be unique, and it also complicates certain types of queries,
such as nearest neighbor searches, which have typically been designed with an
expectation that there is a column with a unique identifier for each object.
To address this, a ``DiaObjectLast`` table could be created which contains only
the most recent version of each object, i.e., the record with the latest
``validityStartMdjTai`` value for each ``diaObjectId``.
This is actually present in the APDB but is not currently replicated to the
PPDB and may require separate processing and handling to keep it up to date.
This table could be alternately be constructed by selecting the records in the
``DiaObject`` where the ``validityEndMdjTai`` has a value of ``NULL``,
indicating that it is the most recent version.
In either case, users could then query directly on this table when they want to
work with unique objects, and it could also simplify certain types of other
queries.

Schema migrations
-----------------

Felis schemas may have an embedded version number which can be incremented when
the schema is changed.
The APDB schema uses semantic versions, e.g., ``v{major}.{minor}.{patch}``,
which are incremented following specific
`rules <https://github.com/lsst/sdm_schemas/blob/main/docs/APDB.md>_`.
The Cassandra implementation of the APDB uses tools in the
`dax_apdb_migrate <https://github.com/lsst-dm/dax_apdb_migrate/>_` project
for managing its schema migrations.
A similar tool may be needed for creating, managing, and performing migrations
in BigQuery.

This migration process would likely involve:

1. Creating an empty dataset for the new schema version using the
   Felis-to-BigQuery library described above.
2. Copying data from the old dataset to the new one, transforming it as needed
   to be compatible with the new schema. (This could be done using Dataflow
   jobs.)
3. Validating the data in the new dataset to ensure that it is correct and
   complete.
4. Switching downstream applications such as the TAP service to use the new
   dataset.

Significant downtime could be incurred during these migrations, and unforeseen
problems could occur, so they should be scheduled during off-hours, as well as
communicated to users in advance.

Table optimization
------------------

TODO: Add PK and FK constraints where appropriate, even though they are not
enforced by BigQuery, as they can help the query planner optimize queries.
https://cloud.google.com/bigquery/docs/best-practices-performance-compute#specify_primary_key_and_foreign_key_constraints

Overview
^^^^^^^^

Performing queries on large tables in BigQuery can be expensive, as the billing
cost is based on the amount of data scanned.
Queries on unoptimized datasets can not only be costly but also have high
latency due to the large amount of data that must be processed when no
pruning is possible.
Therefore, tables should be optimized for common query patterns, primarily by
assigning
`partitions <https://cloud.google.com/bigquery/docs/partitioned-tables>`_
and
`clustering columns <https://cloud.google.com/bigquery/docs/clustered-tables>`_
where appropriate.

Partitioning divides a large table into smaller, more manageable segments based
on a column's values.
Relatively low cardinality columns are typically used for partitioning, such as
a date or integer column.
This allows queries that filter on the partitioning column to only scan the
relevant partitions rather than the entire table, which can significantly
reduce the amount of data scanned and improve query performance.
Clustering organizes (orders) the data within each partition based on the
values of one or more columns.
This allows for more efficient filtering and searching on those columns, as
data blocks can be skipped entirely if they do not match the filter criteria.

By default, when a partition column is assigned, each value is assigned to an
individual partition, but partitioning can also be done using ranges of
values, which may be more appropriate for some columns.
In the latter case, a partitioning function must be defined to specify how the
values are mapped to partitions.
Clustering is hierarchical and can be assigned on up to four columns per table.
The order of the columns in the clustering definition matters, as the first
column determines the primary ordering of the data, the second column
determines the secondary ordering (within the first column's ordering), and so
on.
This scheme can allow for more efficient filtering and searching on those
columns, as data blocks can be skipped entirely if they do not match the filter
criteria.
In particular, filters on the first clustering column will be the most
efficient, with decreasing efficiency for subsequent columns.

Clustering
^^^^^^^^^^

Unoptimized spatial searches would typically require a full table scan, with
a high cost for big (multi-terabyte) tables, so spatial clustering should
be particularly beneficial.
Performance of these queries improves significant when a clustered
``GEOGRAPHY`` column representing the spatial coordinates of an object
is used instead of the numeric ``ra`` and ``dec`` values.
Experimentation has shown that a cone search using the numeric columns on a ~14
gigabyte ``DiaObject`` table results in a full table scan over all of the data.
When the same search was performed using a clustered ``GEOGRAPHY`` column
instead, the amount of scanned data was reduced to 64 MB, a reduction of over
200x.
Since spatial predicates are commonly used in astronomical database searches,
this type of optimization will be critical for ensuring good performance and
reasonable query costs.

Another approach could be using a customized pixelization scheme with HEALPix
or HTM for partitioning or clustering.
However, initial experiments indicate that BigQuery's built-in clustering does
a very good job optimizing spatial queries, at least when the spatial column is
the first clustering column.
A custom scheme would require significant additional effort to implement and
maintain, and it would add complexity to the system.
The TAP layer or some other middleware component would need to perform
translations to/from these pixel values and find the records that were
contained within them.
It is likely that user queries would need to be rewritten automatically to
include a filter on the pixel values.
Finally, the typical resolution of astronomical pixels is much too granular to
use them directly in partitioning, as the number of partitions would exceed the
maximum allowed by BigQuery, so a coarse pixel segmentation would be needed to
divide records into a manageable number of partitions (if partitioning was
being used).
Because of these downsides, and the fact that BigQuery's built-in clustering
apparently does an excellent job optimizing spatial queries, a custom
pixelization scheme for the PPDB is not recommended at this time.

Should the primary production tables be first clustered on a spatial column,
this would optimize spatial queries but degrade the performance of other types
of queries.
The ``diaObjectI`` is a good candidate for the second clustering column, as it
would help to optimize single object queries, which are also a common query
pattern.
However, testing has shown that there is a significant degradation in
performance on the single object selections when ``diaObjectId`` is the second
clustering column compared to when it is the first.
This is likely because the data is not well ordered on ``diaObjectId`` when it
is the second column, so a large amount of data still needs to be scanned.
Other optimization techniques may be considered, such as using multiple copies
of the tables with different clustering strategies to optimize different query
patterns (see below).

Partitioning
^^^^^^^^^^^^

An optimal partitioning strategy seems less clear compared with clustering
after the usage of spatial partitioning has ruled out.
Temporal partitioning is commonly used in many types of databases, but in the
case of the PPDB, it is not obvious that this would be beneficial, as most
queries are not anticipated to be time-based.
The tables do have temporal columns, such as ``validityStartMjdTai``, which is
the start time of validity for each record in a
`TAI <https://en.wikipedia.org/wiki/International_Atomic_Time>`_,
`MJD <https://core2.gsfc.nasa.gov/time/>`_ format.
A floor of this value would be equivalent to the date, which could be used for
partitioning.
This would be straightforward to implement during data ingestion, and an
additional column (hidden from users) could be added to the tables to store
this value.
Although most queries are not expected to filter on this column, it could
still be useful for certain types of queries, particularly for dataset
management and maintenance.
Temporal predicates in user queries could also be rewritten to use this column
when appropriate, which could improve performance for those queries.

One advantage of using this type of partitioning would be relatively equal data
distribution across partitions, as the number of records ingested each day
should be relatively consistent compared with other columns which may have
skewed distributions.
Even if this type of partitioning would not significantly improve the
performance of typical queries, some reasonable partition scheme will be needed
because BigQuery's performance degrades on un-partitioned tables as they grow
beyond a certain size.

Temporal partitioning is also not the only option available; other columns
could be considered, such as using object ID columns, which could potentially
be particularly helpful in improving single object searches and joins.
However, these types of columns may naturally have a skewed distribution of
values, and they have much too high cardinality to be used directly for
partitioning.
So a hashing scheme would be needed to map the values evenly to a manageable
number of partitions.
This would add complexity to the system, probably requiring some rewriting of
user queries to use the hashed/bucketed values.
But this type of partitioning may be the only way to optimize certain query
patterns, so it may be worth considering long-term.

Performance-optimized table copies
----------------------------------

Since there are significant trade-offs in performance when selecting the
partitioning and clustering columns, it may be desirable to have
performance-optimized table copies which can be used for optimizing certain
query patterns.
For instance, a ``DiaObject_byId`` table could be created from a clone of
``DiaObject``.
This would be identical in content but clustered first on ``diaObjectId``, and
then possibly a spatial column.
This should significantly improve the performance of single object queries as
the min/max statistics for each data block would be much more selective.
These table copies could be used explicitly by users in their queries and made
available via the TAP service, or they could be used implicitly via query
rewriting.
These tables could be created using BigQuery's zero-copy cloning feature, which
is very fast and efficient, and they could be updated periodically as needed,
probably when the main production tables were updated.
Since storage costs are relatively low (less than $25 / month / TB) compared to
query costs, having multiple copies of tables optimized for different query
patterns may be worthwhile to reduce the overall cost of the system.

Materialized views
------------------

Another option which could help optimize table access is using materialized
views, which are pre-computed views that can be queried like a table.
Though these do not by default use the same optimizations as the underlying
table, such as clustering columns, they can be partitioned and clustered
independently.
In particular, ``DiaObjectLast``, mentioned above as an extra table, could be
implemented as a materialized view instead of a physical table, which would
simplify the data processing and ensure that it is always up to date with the
latest data.
Since they may have different clustering columns, performance-optimized table
copies could also be implemented as materialized views, which would allow them
to be updated automatically as the underlying data changes.

Data ingestion
==============

Overview
--------

Data ingestion involves the replication of data from the APDB into the PPDB.
The APDB exports data in "replica chunks," (hereafter referred to as just
"chunks") where each chunk contains a set of records from multiple tables that
were created or modified within a certain time window, e.g., ~10 minutes, but
the exact time window is configurable and may vary.
These chunks need to be ingested into the PPDB in a way that maintains data
integrity, e.g., a replica chunk should only be ingested if all prior chunks
have already been ingested.
Later chunks may contain data which reference records in earlier chunks, so it
is important that chunks are inserted into the production system in order,
though they may be batched together for efficiency and need not be inserted
individually.

In order to satisfy this requirement, the data ingestion to BigQuery needs to
be divided into several distinct steps, including the initial export from the
APDB, upload to Google Cloud Storage (GCS), staging into temporary tables in
BigQuery, and, finally, promotion to production tables.
The staging step is done by copying data from the Parquet files into staging
tables, one per production table, where each row is associated with a chunk ID;
the data in these tables is not visible to users or queryable by them.
After the data has been staged, it should be copied from the staging tables
into the production tables, ideally as an atomic operation which ensures data
integrity and consistency.
For efficiency, the promotion process should operate on multiple chunks at a
time, inserting them in a batch operation, while ensuring that all prior chunks
have already been promoted.

Because the APDB is not externally accessible, the data ingestion process will
include multiple systems which are either on-premises or in the cloud.
The on-premises components will run on the Rubin Science Platform (RSP) at the
United States Data Facility (USDF) and are responsible for exporting data from
the APDB and then uploading it to Google Cloud Storage (GCS).
The cloud components will run on Google Cloud Platform (GCP) and are
responsible for all subsequent steps after the data has been uploaded to cloud
storage, including staging into temporary tables and promoting the data to
production.
Any components which must be accessed from both the on-premises and cloud
environments will generally be deployed in the cloud, as instead accessing USDF
resources from the cloud would be more involved and difficult to configure and
administer.

The data ingestion in its current developmental form has been implemented in
Python, heretofore using Python 3.11 but planned to upgrade to 3.12 soon.
However, the Apache Dataflow environment is constrained to use Python 3.9, and
it is unclear when, if ever, support for later versions will be added.
In practice, this mismatch should not be a major issue, because the Dataflow
jobs are configured entirely separately from Cloud Run functions, so they may
use the appropriate Python version.
This may mean that the Dataflow jobs (currently only a single job but more
may be added) would need to be implemented in a standalone fashion that does
not rely on shared libraries used in other parts of the system.

Chunk tracking
--------------

A Postgres database (hereafter referred to as the "chunk tracking database" for
lack of a better term) will be used to track the status of each chunk as it
moves through the data processing pipeline.
This database will coordinate the various processes involved in exporting,
uploading, staging, and promoting the chunks.
The chunk tracking database contains a ``PpdbReplicaChunk`` table with one row
per chunk and columns for the chunk ID, timestamp, status, local export
directory, unique ID, and status.
Because it must be accessible from both the USDF and cloud environments, it
will be hosted on a managed Postgres instance on GCP, either in Cloud SQL or
on Kubernetes using Cloud Native Postgres (CPNG).

The chunk tracking database is accessible from the RSP via an external IP
address, which is secured using a firewall limiting connections to only the
USDF's (or currently SLAC's) external IP addresses.
A VPC connector has also been deployed for additional security.
The Cloud Run functions and Dataflow jobs can access this database using its
internal IP address, which should eventually be converted to use
a private DNS name.
(The external address used for inbound USDF connections is not accessible from
within GCP.)

Export and upload
-----------------

The data ingestion process begins with exporting data from the APDB into local
Parquet files, which are currently organized into directories by chunk ID.
These chunks are then uploaded to a Google Cloud Storage (GCS) bucket under a
a specific prefix, where they can be accessed for staging the data into
temporary tables.

The following describes the steps involved in exporting and uploading the data
from the APDB:

1. The APDB is periodically queried by a long-running "exporter" process which
   finds chunks that are ready to be exported.
2. When chunks are found, the table data is exported to Parquet files, one
   per table, in a directory for that chunk.
3. A manifest file is generated containing useful metadata about the chunk,
   such as its chunk ID, timestamp, number of rows per table, etc.
4. The information for the exported replica chunk is inserted into the chunk
   tracking database as a new record with a status of "exported."
5. A separate "uploader" process continually monitors the chunk tracking
   database to find chunks with the "exported" status, indicating that they are
   ready to be uploaded.
6. For each ready chunk, the Parquet files are uploaded to GCS under a unique
   prefix (pr "folder"), along with the manifest file containing the metadata.
7. After the upload for a particular chunk has been completed, the uploader
   publishes a Pub/Sub message with information necessary for importing the
   chunk's data, such as the GCS bucket and path, chunk ID, and timestamp.

As currently implemented, the export process serially writes each chunk to
Parquet without any multi-processing.
Similarly, the uploader also processes chunks one at a time, though the copying
of each chunk's Parquet files into cloud storage is parallelized using a thread
pool executor.
Though it is unknown at this time whether these processes will be able to keep
up with production data rates from the APDB, they would ideally be able to do
so without requiring parallelization.
Single process optimization should be aggressively pursued before either of
these processes was parallelized, as this would significantly complicate the
system, likely requiring an additional orchestration component.

Staging
-------

Once the data has been copied into a GCS bucket, a cloud-native process stages
the data into temporary tables in BigQuery.

This process involves a few steps:

1. A Cloud Run function which is subscribed to the appropriate Pub/Sub topic is
   triggered when a new chunk has been uploaded.

2. A ``stage_chunk`` function decodes the Pub/Sub message, extracts the
   necessary information (and may also access the associated manifest file if
   necessary, though the function does not, currently), and then launches an
   Apache Beam job on Google Dataflow to stage the data.

3. Based on the job parameters, the Beam job reads the Parquet files from GCS
   and copies their data into the appropriate staging tables in BigQuery.

The Dataflow jobs execute asynchronously and in parallel, as multiple chunks
can be staged at the same time without any data integrity issues.
Some experience with this system in development has shown that the total time
to launch and complete a Dataflow job is between 8 and 15 minutes and may vary
based on the size of the chunk.
While overall limits on BigQuery insert limits must be considered, it is
unlikely that the staging process will be a bottleneck, overall, given that
many chunks can be staged in parallel and the insert Operations into BigQuery
are highly optimized and performant.

The staging tables themselves are long-lived rather than temporary, as they
are continually receiving data from new chunks generated by different
Dataflow jobs.
These tables have the same schema as the production ones with the addition
of a ``apdb_replica_chunk`` column which keeps track of which chunk each
record came from.
Currently, no additional data processing is done in this step; instead, it is
envisioned that any additional columns which need to be computed or derived
will be calculated during the promotion step.

Promotion
---------

After data has been uploaded to the staging tables, a separate process promotes
the data to production tables.
This is described in more detail below.

1. A Cloud Run function called ``promote_chunks`` is run on a periodic and
   configurable schedule.
   (This could be as frequent as every 15 minutes or as infrequent as once
   per day, depending on the desired cadence for making data available to
   users.)

2. This function identifies a batch of promotable chunks in the chunk tracking
   database, which is the next continuous sequence of chunks that have been
   staged where all prior chunks have already been promoted.
   If there are no promotable chunks, the process logs this fact and then
   exits.

3. A temporary snapshot is created for each production table.
   In BigQuery, this is a "zero copy" operation, which is very fast and
   efficient and will not incur additional storage costs for bulk data copying.

4. Data is then copied from the staging tables to their corresponding temporary
   table. It is likely that during this step, the values for computed or
   derived columns will be calculated and populated.
   These may include, for instance, temporal partitioning columns or spatial
   ``GEOGRAPHY`` columns for clustering (discussed in detail later).

5. An atomic table swap is then performed to promote the temporary table to
   production. Though not currently the case, a snapshot could be created
   beforehand to allow for rollback in the event of an error.

6. The records for the chunks that were promoted are deleted from the staging
   tables so that they are not processed again.

The time cadence of the promotion process has not been finalized but it is
fully configurable as part of the Cloud Scheduler deployment.
A daily promotion job, starting in the morning or early afternoon (Chilean
time) after nightly data taking, would satisfy the formal requirement that L1
data be made available within 24 hours of acquisition.
However, some discussion has indicated that a more frequent promotion cadence
would be desirable so that data was available to scientists sooner.
The time between promotions jobs could be as frequent as every 15 minutes or
one hour, depending on the desired latency.
Experiments can be performed to determine the optimal cadence for this
process while taking into account the data rates, processing times, and desire
to make data available as soon as possible, both for end users and downstream
data processing.

Updating existing data
----------------------

Updates to existing records may occur in the APDB, which will then need to be
propagated to the PPDB (discussed in *DMTN-322* :cite:`DMTN-322` and
`DM-50190 <https://rubinobs.atlassian.net/browse/DM-50190>`_).
These may occur either as new versions of existing records which are ingested
in replica chunks, or they may be implemented as separate record-level events
(The exact mechanism for the latter type of updates is still under
consideration.).

A known example where existing data should be updated is setting the validity
end time of DiaObject records.
Records in the ``DiaObject`` table may have multiple versions, with the
``diaObjectId`` and ``validityStartMjdTai`` forming the composite primary key.
When a new ``DiaObject`` version is created and written to the chunk data, the
previous version's ``validityEndMjdTai`` value should be updated to the prior
version's validity start time.
This operation can occur as a step within the data ingestion pipeline, likely
when as part of the promotion from staging to production.

There are several other record-level update operations that may need to be
performed as well, including but not limited to:

- Re-assigning a fraction of ``DiaSource`` records to new SSObjects.

- Deletion of ``DiaObject`` records, e.g., those without matching ``DiaSource``
  records after re-assignment has been performed.

- Withdrawal of ``DiaSource`` or ``DiaForcedSource`` records through setting
  a "time withdrawn" column to a valid date.

An additional complexity is that some types of updates may need to be ordered
in time, as results of multiple operations may also depend on the order in which they are
applied.
Overall, the implementation of these updates in the PPDB will first depend on
these processes being fully defined and specified and then implemented in the
APDB, which is still ongoing work.

User access
===========

TAP service
-----------

It is planned that the PPDB will provide user access through a TAP service
(CITATION: TAP), which will allow users to query the data using
`ADQL <https://www.ivoa.net/documents/ADQL/>`_, similar to how other several
other Rubin Science Data Model (SDM) databases are currently accessed such as
the data previews.
A BigQuery TAP service was developed by Burwood Group several years ago in
collaboration with Google and is described in the `TAP BigQuery`_ document.
This service is an extension to the
`CADC TAP repository <https://github.com/opencadc/tap>`_, a Java implementation
of the server-side TAP protocol.
Connectivity is provided using the
`Simba driver <https://insightsoftware.com/drivers/bigquery-odbc-jdbc/>`_
for JDBC which can accept configuration parameters via a connection string.

A test version of this TAP service is currently deployed standalone (e.g., not
inside of an RSP environment) on GCP.
The authentication in this test environment currently uses network-level
restrictions, which would not be used in the fully integrated production
system.
In the production system, a service account key file (JSON format) will be used
to authenticate to an IAM service account which has appropriate permissions to
query the PPDB dataset(s).
The contents of this key file can be stored securely in a secret management
system and retrieved at runtime by the TAP service.

This test deployment has been accessed and used successfully to return query
results in the RSP's generic TAP service GUI and from within a Python notebook
using the `pyvo <https://pyvo.readthedocs.io/en/latest/>`_ TAP library.
Initial testing with this deployment on a small dataset, with about 15 GB of
data in the ``DiaObject`` table, has shown low latency, with results of some
simple queries returned in ~3 seconds or less.
Actual performance will depend heavily on the size of the dataset and the
complexity of the query.

However, the BigQuery TAP implementation is still a prototype.
Much work remains to be done to improve the features of this TAP service,
including but not limited to:

- Allowing runtime configuration of the BigQuery dataset mapping where
  ``ppdb``, or some other generic name utilized by users in their queries, is
  mapped to the fully qualified dataset name. (This is currently hardcoded in
  the prototype's implementation but it should be configurable in some way
  through a configuration file or some other mechanism.)
- Uploading of results to cloud storage, as the current results store is set
  to a local directory. (This is not a standard feature of the CADC TAP
  implementation but is needed for integration with the RSP.)
- Support for optimized spatial queries using the BigQuery GIS functions with
  ``GEOGRAPHY`` columns (covered in the next section).
- Integration with other Rubin extensions and modifications to the TAP service
  used on the RSP, some of which are described in *SQR-099* :cite:`SQR-099`.

Overall, the current TAP service implementation should be viewed as a proof of
concept rather than a production-ready system, and it may undergo significant
revision before it is suitable for use within an operational environment.

TAP Schema
^^^^^^^^^^

The TAP service will needs to access standard TAP_SCHEMA tables
(CITATION: TAP) in order to describe the tables and columns available for
querying, as well as to correctly parse and translate user ADQL queries.
Instead of storing the TAP_SCHEMA database in BigQuery, the prototype instead
uses a Postgres implementation, which has been tested and confirmed to work, in
particular in the generic TAP query interface of the RSP portal and in
Python notebooks.
The database is implemented in Postgres primarily for performance reasons, as
the TAP service needs to query these tables frequently with low latency, and
Postgres is better suited for this type of workload.

Eventually, tools should be developed for populating the TAP_SCHEMA tables
from the BigQuery table schema for a given schema version.
There are a few "quirks" having to do with data types which may necessitate
usage of custom tooling rather than the standard Felis (CITATION: felis)
command-line interface.
For instance, the VOTable (CITATION: VOTable) ``float`` type does not currently
map correctly to BigQuery's ``FLOAT64`` and so ``double`` should be substituted
instead, which does (TODO: Is this because of the Simba driver?).
Another similar issue is that all of BigQuery's integer types are 64-bit, so
the VOTable ``long`` type should always be used instead of ``int`` or
``short``, which would not be interpreted correctly.
Further work on the TAP service and its mapping between VOTable and BigQuery
types could resolve some of these issues so that the standard tooling could be
used for generating the TAP_SCHEMA database without any modifications

Another requirement of the TAP service is that a schema name used in the
TAP_SCHEMA data, such as ``ppdb``, must be mapped to a fully qualified dataset
name in BigQuery.
In the prototype version of the TAP service, this mapping is hardcoded into the
Java code, e.g., ``ppdb`` in TAP_SCHEMA is mapped to
``my_ppdb_project.my_ppdb_dataset``.
Ideally, this mapping would be configurable in some way, either via a
configuration file, argument to the TAP server, or environment variable.

Table uploads
^^^^^^^^^^^^^

TODO: More details could be included in this section on how user uploads could
be implemented and what limitations there may be, e.g., size limits, expiration
time, etc.

The TAP REC includes support for uploading user tables, which the BigQuery
implementation should eventually include.
This would need to be implemented as a custom backend service, as no standard
implementation exists for BigQuery.
A starting point could be the `CREATE TEMP TABLE` syntax which is natively
supported, which would at least allow usage of uploaded data within a single
query.
However, this would likely be only a temporary solution, as it does not allow
users to store their tables long-term or use them in multiple queries, e.g.,
for cross-matching.
The proper solution would likely involve creating of tables in a separate
BigQuery dataset (and possibly within another GCP project) for security and
access control reasons, as well as to avoid cluttering the main PPDB dataset
with potentially hundreds of user tables.

Spatial query support
---------------------

Spatial queries are a commonly used pattern when searching astronomical
databases, and the PPDB will need to support them efficiently.
ADQL supports geometric
`types <https://ivoa.net/documents/ADQL/20230418/PR-ADQL-2.1-20230418.html#tth_sEc3.5>`_
and `functions <https://ivoa.net/documents/ADQL/20230418/PR-ADQL-2.1-20230418.html#tth_sEc4.2>`_
for constructing spatial queries, and these will need to be mapped to BigQuery
GIS functions in the TAP service.
Some number have already been mapped, but further work is needed to ensure that
all commonly used functions are supported.
BigQuery has the capability of performing spatial queries using a spherical
rather than spheroid model, which is ideal for astronomical use cases.
The actual implementation should be check so that it uses these functions
correctly.
Most of the GIS functions accept a ``use_spheroid`` argument which can be set
to ``FALSE`` to use the spherical model.
This would be done automatically in the TAP service when rewriting ADQL queries
to BigQuery SQL.

Cone search
^^^^^^^^^^^

One of the most common spatial query patterns is a cone search, which finds
all objects within a certain radius of a given position on the sky.
In ADQL, this is typically expressed using the ``CONTAINS``, ``POINT``, and
``CIRCLE`` functions.

Below is an example of a cone search that finds all ``DiaObject`` records
with spatial coordinates that are within 1.0 degree of a given spherical
position:

.. code-block:: sql

    SELECT *
    FROM ppdb.DiaObject
    WHERE CONTAINS(
    POINT('ICRS', ra, dec),
    CIRCLE('ICRS', 186.8, 7.0, 1.0)) = 1

When the numeric (``FLOAT64``) ``ra`` and ``dec`` columns are used directly,
this type of query would typically require a full table scan, which can be
very expensive on large tables.
Work is ongoing to optimize this type of query for performance by adding
``GEOGRAPHY`` columns to relevant tables, such as ``DiaObject``, which
represent spherical points on the sky.
These ``GEOGRAPHY`` columns can then be used in the query instead of the
numeric ``ra`` and ``dec`` columns, which will allow the query to be executed
much more efficiently.
Initial testing has shown that using ``GEOGRAPHY`` columns can reduce the amount
of data scanned by up to several orders of magnitude, which can significantly
reduce query costs and latency.

Nearest neighbor search
^^^^^^^^^^^^^^^^^^^^^^^

Another common spatial query pattern is a nearest neighbor search, where
objects within a certain distance of each other are found, typically within a
limited region of the sky.

A query such as this could be expressed in ADQL as follows:

.. code-block:: sql

    SELECT o1.diaObjectId AS id1, o2.diaObjectId AS id2,
       DISTANCE(POINT('ICRS', o1.ra, o1.dec),
           POINT('ICRS', o2.ra, o2.dec)) AS d
    FROM ppdb.DiaObject o1
    JOIN ppdb.DiaObject o2
    ON o1.diaObjectId < o2.diaObjectId
    WHERE CONTAINS(POINT('ICRS', o1.ra, o1.dec),
        CIRCLE('ICRS', 186.5, 7.05, 0.5))=1
    AND DISTANCE(POINT('ICRS', o1.ra, o1.dec),
        POINT('ICRS', o2.ra, o2.dec)) < 0.5;

As with cone searches, using ``GEOGRAPHY`` columns instead of numeric ``ra``
and ``dec`` columns will allow these types of queries to be executed much more
efficiently.

The above query will not actually work correctly though on the current
``DiaObject`` table, as the ``diaObjectId`` column is not unique by itself.
Another table containing only the most recent object versions could be created
to support this type of query in a more natural way.
Or the query itself could be rewritten to use the full composite primary key
(``diaObjectId`` plus validity start) in order to ensure uniqueness of the
results.

Additional query patterns
-------------------------

In addition to spatial queries, the PPDB will need to support other common
query patterns. These will not be covered exhaustively here, but several common
ones will be considered.

Single object selection
^^^^^^^^^^^^^^^^^^^^^^^

Sometimes users will want to select a single, known object by its unique
identifier.
This is typically done using a simple ``WHERE`` clause, as in the following
example:

.. code-block:: sql

    SELECT * FROM ppdb.DiaObject WHERE diaObjectId=24624704620855420

On unoptimized tables without clustering or partitioning, this type of query
will generally result in a full table scan.
Clustering on the ``diaObjectId`` column should significantly improve the
performance of this type of query, though there will be trade-offs to consider
when choosing which columns to cluster on and in which order.
If ``diaobjectId`` was the first clustering column, single object queries would
be optimized but the performance of other types of queries might be degraded.

Table joins
^^^^^^^^^^^

Joins between tables are a common query pattern, such as joining ``DiaSource``
to ``DiaObject`` to construct a light curve.
These types of queries can be expensive, as they typically require up to a full
table scan of one or both tables.
For instance, the following query joins ``DiaSource`` to ``DiaObject`` to get
all sources for objects within a certain region of the sky:

.. code-block:: sql

    SELECT s.*, o.*
    FROM ppdb.DiaSource s
    JOIN ppdb.DiaObject o
    ON s.diaObjectId = o.diaObjectId
    WHERE CONTAINS(
        POINT('ICRS', o.ra, o.dec),
        CIRCLE('ICRS', 186.8, 7.0, 1.0)) = 1

Some experimentation has been performed with clustering both the ``DiaSource``
and ``DiaObject`` tables on the ``diaObjectId`` column in an attempt to
optimize this type of query, but this did not seem to significantly reduce data
scanning.
Further experimentation is needed to determine the optimal clustering strategy
or other techniques that could be used to reduce data scanned when joining
tables.

Table scans
^^^^^^^^^^^

Some types of queries will naturally require a full table scan, such as
filtering on a magnitude column without any spatial or ID constraints.

One example is the following query which searches for all objects where the
mean PSF flux in the r-band is between two values:

.. code-block:: sql

    SELECT * FROM ppdb.DiaObject WHERE r_psfFluxMean BETWEEN 1090.0 and 1100.0

Unless the table is clustered on this column, this would always require a full
scan.

Under certain circumstances, queries which would naturally require a full table
scan may be optimized even without clustering or partitioning on the relevant
columns.
A ``LIMIT`` query may be able to stop scanning early once the limit has been
reached, though this is not guaranteed and would depend on the query plan
chosen by BigQuery.
Block-level min and max statistics, which BigQuery stores even for un-clustered
tables, may also be used to skip over data blocks that do not contain any
matching records, though this cannot be relied upon as a consistent
optimization strategy.

Deployment and operations
=========================

Most components of the PPDB will be deployed and operated on GCP, though some
need to run on-premises at the USDF.
The PPDB could therefore be considered a hybrid cloud system, though only the
first few steps of the data ingestion process are on-premises, with all other
components running in the cloud.

USDF deployment
---------------

The two components of the system which much run on-premises at the USDF are the
exporter and uploader, as they need direct access to the APDB, which is not a
publicly accessible system.
The two processes are implemented in the ``ppdb-replication`` application
within the `Phalanx <https://phalanx.lsst.io>`_ repository, which is the
standard project used to deploy and manage RSP applications.
This application can be deployed and managed using Argo CD, which provides a
web interface for monitoring and administration.
Specific versions of the application may be deployed by referencing their
Docker image tags in the Phalanx configuration.

Compute resource requirements for these two processes are relatively modest;
the exporter additionally needs disk space for storing the exported Parquet
files before they are uploaded to GCS.
It has yet to be determined whether or not the files will be kept long-term as
an on-premises backup.
They could potentially be deleted as soon as the files are copied into GCS,
though it may be prudent to keep them for a certain period, e.g., 30 days, to
allow for recovery in the event of a catastrophic failure.
Exactly how much space these files require will depend on the data rates from
the APDB and the retention period, as well as the Parquet compression level,
but it is likely that at least 10 TB of disk space should be allocated for this
initially, with the capability to expand if necessary.

GCP deployment
--------------

All of the other components of the PPDB will run on GCP, with the exception of
the TAP service, which should eventually be able to run within an RSP
environment in the cloud or at the USDF.
The PPDB GCP project has many services which need to be deployed and
configured.
Most of these are fully managed administratively by Google using a Software as
a Service (SaaS) model, which should reduce operational burden, but all
together they will still require significant effort for their configuration and
management.

GCP services
^^^^^^^^^^^^

Required GCP services must be enabled in the project before they can be used.
These are summarized below with their purpose for the PPDB and the tasks
involved in setting them up.

.. list-table::
   :header-rows: 1

   * - **Service Name**
     - **Purpose for PPDB**
     - **Setup Tasks**
   * - BigQuery
     - data storage and querying
     - - dataset creation
       - table creation
       - permissions and roles
   * - Google Cloud Storage (GCS)
     - Parquet file storage

       Dataflow flex templates and other artifacts
     - bucket creation
   * - Cloud Run
     - Cloud functions for data processing
     - function deployments
   * - Cloud Scheduler
     - scheduling periodic tasks
     - promotion job scheduling
   * - Dataflow
     - data ingestion and processing
     - flex templates, etc.
   * - CNPG or Cloud SQL
     - chunk tracking database
     - - instance creation
       - user and database setup
       - GKE cluster setup (if CNPG)
   * - IAM
     - permissions and roles for services
     - - service accounts
       - roles and permissions
   * - virtual networking
     - secure connectivity between services
     - - VPC setup
       - firewall rules
   * - Pub/Sub
     - messaging between components
     - - topic creation
       - subscription creation

Terraform deployment
^^^^^^^^^^^^^^^^^^^^

Google projects within Rubin are generally managed using the
`idf_deploy <https://github.com/lsst/idf_deploy>`_ repository, which uses
Terraform to define project-level settings and create/deploy services, user
accounts, etc.
Additionally, the repository is also used to manage important project-level
metadata such as the billing account, budget alerts, and other organizational
policies.
This repository should eventually be used for managing the PPDB project as
well, ensuring that the configuration can be version controlled and easily
deployed, which will be particularly crucial in a disaster recovery scenario.

Currently, the PPDB cloud configuration is managed by a set of ad hoc
development scripts in the
`ppdb-scripts <https://github.com/lsst-dm/ppdb-scripts>`_
repository.
Porting these to Terraform, as well as adding the additional configuration
needed for a production deployment, will require a significant level of
developer effort and knowledge in devops engineering and GCP services.
An individual with significant expertise in this area will be needed to lead
this effort.

Even after the project is managed in Terraform, some aspects may be better
managed outside of it.
This could include the creation of BigQuery tables, which will likely be done
using a custom tool which can read the schema from a file and create the tables
programmatically.
Additionally, Cloud Run functions will likely be deployed in GitHub Actions
where new versions are pushed automatically when new code is merged to the
main branch (This is the approach currently used by other Rubin projects.)
Which components are managed by Terraform and which are managed by other
tools still needs to be determined.

Backup and recovery
-------------------

The PPDB will need to implement a backup and recovery strategy to ensure data
integrity and availability in the event of a failure or data loss.
BigQuery provides built-in features for data backup and recovery, including
table snapshots and point-in-time recovery.
Table snapshots allow for creating a read-only copy of a table at a specific
point in time, which can be used for recovery in the event of accidental data
deletion or corruption.
Point-in-time recovery allows for restoring a table to a specific point in
time within the last 7 days.
These features can be used to implement a backup and recovery strategy for
the PPDB, ensuring that data is protected and can be recovered in the event
of a failure or data loss.

Table snapshots may be stored in a separate dataset or project for additional
protection against data loss and for de-cluttering the main dataset.
An automated process can be implemented using a scheduled Cloud Run function or
job to create snapshots of the production tables at regular intervals, such as
every day or week.
If the promotion process was run on a daily basis, then it would be prudent to
create snapshots of all the production tables before it ran.
Snapshots can be retained for a configurable amount of time, e.g., 30 days,
after which they are automatically deleted.
Creating snapshots frequently should not incur significant additional costs
compared with storing the original tables, because they are logical copies and
do not require duplicating the underlying data.
Restoring data from a snapshot can be done using the BigQuery console,
command-line interface, or API.
These operations are generally fast and efficient, as they also do not require
copying data but instead create a logical view of the data at the specified
point in time.

The Parquet files exported from the APDB and uploaded to GCS could also be
retained as an additional backup mechanism.
These files could be kept in GCS for a configurable period to allow for
recovery in the event of a catastrophic failure that affected both the BigQuery
production dataset and the snapshots (unlikely but possible).
These files could also be stored indefinitely as a long-term archive in cloud
storage, though this would incur additional storage costs.
If this were the case, the objects should at least be configured with
``Object Lifecycle Management <https://cloud.google.com/storage/docs/lifecycle>`_
rules to transition them to colder storage classes, such as ``NEARLINE``,
``COLDLINE``, or ``ARCHIVE``, after a certain period of time to reduce costs.
Parquet files could also be stored on-premises as an additional protection
mechanism; this would require significant disk space as the data volume grew
over time.
The exact data volume has yet to be determined, but it is likely that tens of
terabytes could be generated in operations over the course of a year.

Finally, the chunk tracking database (the Postgres component of the PPDB) will
also need to be backed up, as it is a critical component of the data processing
pipeline.
If a managed Cloud SQL instance is used, automated backups can be enabled
using the built-in features of Cloud SQL (preferable option).
Or if a CNPG instance were used, a custom backup and recovery process would
need to be implemented, such as using ``pg_dump`` from a script to create
logical backups of the database at regular intervals and then storing them in
GCS.

Monitoring and Logging
----------------------

Monitoring and logging will be critical for operating the PPDB in a production
environment.
For the on-premises components running at the USDF, Grafana can be used with
Loki to collect and visualize logs and metrics for the ``ppdb-replication``
application (This is the approach used by many existing RSP applications
running at the USDF.).
In particular, it will be important to monitor whether these processes are
able to keep up with the data rates from the APDB and to alert operators
when they fall behind or even fail.
System resource usage may also need to be monitored to ensure that there is
sufficient CPU, memory, and disk space available for these processes to run
smoothly.

On GCP, there are several standard monitoring and logging services that can be
used to track the health and performance of the various components.
The Logs Explorer in Cloud Logging can be used to collect and view logs from
all GCP services, including Cloud Run, Pub/Sub, Dataflow, etc.
Most of the other services also have their own built-in monitoring and logging
features in the console which can be used to track their performance and
health.
Cloud Run has built-in monitoring and logging that log function activations,
errors, and performance metrics.
Similarly, Dataflow jobs have built-in monitoring and logging feature which are
accessible from the GCP console.
All the other services either have their own monitoring and logging features or
are accessible from the general Cloud Monitoring and Logging service (or both).

While the built-in GCP services are quite useful, they may not be sufficiently
integrated or targeted for use in a production environment.
Custom dashboards can be created in Cloud Monitoring and populated from various
sources to provide a unified dashboard for the PPDB components, showing key
metrics, alerts, etc. in a more user-friendly and concise manner.
An additional option which would provide an even greater level of customization
is a cloud dashboard application such as
`Looker Studio <https://cloud.google.com/looker-studio>`_ which could include
information from multiple GCP services and would be ideally suited for querying
BigQuery datasets to generate custom reports and visualizations.

Source-code repositories
========================

The PPDB even in its current prototype form involves a large number of source
code repositories.
These are summarized below along with their purpose and location.

.. list-table::
   :header-rows: 1

   * - **Repository**
     - **Purpose**
     - **Notes**
   * - `dax_ppdb <https://github.com/lsst/dax_ppdb>`_
     - Python interfaces for the PPDB
     - Additional tooling for BigQuery is planned.
   * - `dax_apdb <https://github.com/lsst/dax_apdb>`_
     - Python interfaces for the APDB
     - Used by ``dax_ppdb`` for accessing the APDB
   * - `dax_ppdbx_gcp <https://github.com/lsst-dm/dax_ppdbx_gcp>`_
     - GCP-specific extensions for ``dax_ppdb``
     - Encapsulates the GCP dependencies and tools for ``dax_ppdb``
   * - `ppdb-cloud-functions
       <https://github.com/lsst-dm/ppdb-cloud-functions>`_
     - Cloud Run functions for data processing on GCP
     - May be split into one repo per function in the future
   * - `ppdb-scripts <https://github.com/lsst-dm/ppdb-scripts>`_
     - Ad hoc dev scripts and config, primarily for GCP
     - Will be ported to Terraform or Python tools
   * - `phalanx <https://github.com/lsst-sqre/phalanx>`__
     - RSP application deployment framework
     - Deployment and management of the ``ppdb-replication`` application
   * - `idf_deploy <https://github.com/lsst/idf_deploy>`_
     - RSP configuration and deployment using Terraform
     - A PPDB project needs to be added to this repo for production.
   * - `tap-bigquery <https://github.com/lsst-dm/tap-bigquery>`_
     - TAP service for querying the PPDB
     - Work ongoing to add features and optimizations
   * - `opencadc/tap <https://github.com/opencadc/tap>`_
     - Base TAP service used by ``tap-bigquery``
     - Also extended by other Rubin TAP services (Qserv, etc.)

Conclusions
===========

TODO: Convert bulleted lists in this section to numbered lists once their
contents and ordering are finalized.

Strengths
---------

The following summarizes some of the strengths of using BigQuery for the PPDB:

- On-premises data processing for exporting the data from the APDB and
  then uploading to GCS seems fairly robust based on some limited operational
  experience on various data volumes and schema versions.
  Given a baseline export of replica chunks every 10 minutes, the components
  should be able to keep up with the data rates even if they run as
  single-threaded processes.
- GCP's web console, command-line interface, and APIS provide a robust and
  flexible framework on which the PPDB can be built.
  Initial rapid development has been possible using these tools and the results
  are promising.
  The BigQuery console provides a user-friendly interface for querying and
  managing datasets, and the command-line interface and APIs allow for
  automation and integration with other tools and systems.
  Authentication and authorization mechanisms are handled seamlessly by
  Google's IAM service, which should also provide a robust framework for
  managing the permissions and roles for users and services.
- The cloud-centric architecture should provide a highly scalable and robust
  framework for data ingestion and processing.
  Provided that certain
  `quotas <https://cloud.google.com/bigquery/quotas>`_
  are not exceeded, the system should be able to handle the expected data rates
  from the APDB, as, in particular, Dataflow can scale up to many simultaneous
  workers to process data in parallel.
  The staging strategy should allow data to be ingested simultaneously and
  asynchronously from multiple replica chunks, which can be out of order when
  inserted into the staging tables.
- Logging and monitoring tools of GCP even in their default/generic form
  provide a good starting point for operating the system in production.
  Custom dashboards can be created to provide a unified view of the system's
  health and performance.
- BigQuery has been shown to have relatively low query latency, even though
  there is overhead for dispatching work to multiple nodes and aggregating the
  results.
  Latencies from test queries have generally been in the range of a few seconds
  to under a minute.
  Initial results are promising, and since BigQuery is designed to scale to
  petabytes of data, the requirements of the PPDB (probably several hundred
  terabytes of table data at the most) should be well within its capabilities.
- Initial experiments with table optimizations and, in particular, spatial
  clustering have shown that common query patterns can be optimized to require
  much less than a full table scan, which should significantly reduce overall
  query costs.
  Other optimization techniques such as partitioning and performance-optimized
  table copies can be investigated and implemented as needed to optimize the
  most common query patterns.

Risks and limitations
---------------------

- The overall complexity of the system is and will be quite high, requiring
  many different components and services operating in concert.
  This complexity increases the operational burden and potential for failures,
  bugs, and outages.
  While a goal of achieving the least amount of complexity possible should be
  pursued, much of this is irreducible given the requirements of the system and
  its implementation details.
  During operations, it is likely that significant effort will be needed to
  monitor and manage the system, respond to alerts, and troubleshoot issues.
- The BigQuery TAP service, while functional as a prototype, will still require
  significant effort to achieve a production-ready system.
  `Rubin-specific features <https://sqr-099.lsst.io/>`_
  that have been added to the CADC TAP service will need to be incorporated,
  in particular those for authentication, authorization, and table data
  writing.
  In theory, the TAP service may implement various optimizations using, but
  but implementing them in a way that avoids "hard coding" Rubin-specific
  knowledge will be challenging (if this is a goal).
  Generic extension or plugin mechanisms are not a well-supported feature of
  the TAP library, so implementing all of these features in a clean and
  maintainable way could be challenging.
- Keeping the data in-sync with the APDB presents various challenges.
  Any update to the APDB data which would affect science queries will need to
  be propagated to the PPDB in a timely manner.
  This includes not only new data being ingested but also updates to existing
  data, such as deletions/retractions or updates to existing records.
  Implementing these in the PPDB is blocked behind these features first being
  added to the APDB, which is a separate project with its own priorities
  and schedule.
  Additionally, schema migrations in the APDB will require a corresponding
  set of updates to the PPDB, which may require significant downtime that could
  be hard to estimate or control.
- The financial cost of running the PPDB in a production environment is
  uncertain and depends on a large number of factors, including the table data
  volumes and the types and volume of queries.
  While the on-demand pricing model provides nearly limitless flexibility and
  scalability in terms of servicing user queries, it also means that costs
  could be unpredictable and potentially very high should the system be used
  heavily or inefficiently.

Future work
-----------

- Finalization of the data ingestion pipeline should be achieved as soon as
  possible to facilitate data ingestion from the APDB into the PPDB once survey
  operations begin, even if the database is not immediately available for user
  queries.
  This task will include redeploying and updating the ``ppdb-replication``
  application at the USDF and should also involve implementation of any
  additional data transformations needed to convert the data into a suitable
  form for eventual use in science queries.
- The data ingestion pipeline should be made robust to failures, errors, and
  outages.
  Each step should ideally be idempotent (not currently the case) so that if it
  fails, the pipeline can rerun on the same input data without raising errors
  or causing data corruption or duplication.
  Error-handling and reporting also needs attention so that the causes of
  failures can be diagnosed and addressed, without an overwhelming number of
  log messages and alerts being generated.
- The TAP service should be further developed and tested to bring it to a
  production-ready state.
  In particular, integration with Rubin-specific authentication and
  authorization systems is a critical requirement that must be considered
  and addressed.
  Performance optimizations are also a high priority, as unoptimized queries
  could result in excessively high financial costs as well as increased
  latency for users.
  More planning and design work is needed around integration with the RSP and
  Rubin-specific TAP extensions, and customizations should ideally be added
  in a way where the BigQuery TAP service does not hard-code Rubin-specific
  knowledge and features, at least not in a way which is non-configurable.
  One example of this is that the mapping of the schema name in TAP Schema to
  the actual BigQuery dataset name is currently hardcoded into the BigQuery
  TAP service, which will be cumbersome and inflexible in the long term.
- Tooling is needed for automating the creation and management of multiple
  datasets and tables along with their schema versions so that manual setup is
  minimized.
  There should also be tools to manage schema migrations, as these will
  inevitably occur over time as the APDB schema evolves.
- Cost control and monitoring will be critical for operating the PPDB in a
  production environment, as depending on the types and volume of user queries,
  query costs could become excessive.
  Project-level budgets and alerts should be set up to notify operators when
  costs exceed certain thresholds.
  Assuming on-demand pricing,
  `custom quotas <https://cloud.google.com/bigquery/docs/custom-quotas>`_
  should be assigned so that the system does not exceed its (monthly) budget.
  Ideally, limits on individual queries should also be implemented to avoid
  particular users from flooding the system with excessively large queries
  which could incur a high financial cost.
  (More details on strategies for cost estimation and control will be presented
  in a subsequent tech note.)
- Significant effort will be needed to deliver the system on time and to
  operate it in production, particularly in the area of devops and GCP
  configuration and management using Terraform.
  Coordination with other Rubin teams is also needed to fully integrate the
  BigQuery TAP service with the RSP and Rubin-specific extensions.

.. _LDM-135: https://ldm-135.lsst.io
.. _LDM-555: https://ldm-555.lsst.io
.. _LSE-163: https://lse-163.lsst.io
.. _LSE-61: https://lse-61.lsst.io
.. _DMTN-308: https://dmtn-308.lsst.io
.. _SQR-099: https://sqr-099.lsst.io

..
    Named links:
.. _TAP Bigquery: https://docs.google.com/document/d/1CigD-6xJgTWZV82HUEYZBCxhgwfpOHfu8ABwmjLxZ4E

References
==========

.. bibliography::
