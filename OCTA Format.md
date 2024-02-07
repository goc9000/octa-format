OCTA Format
===========

Version: 0.0.0


Summary
-------

OCTA (Online Crawling Trace Archive) is a database format for storing the complete history ("trace") of Web crawling
sessions.


Table of Contents
-----------------

- [Summary](#summary)
- [Design Goals](#design-goals)
- [Comparison to Alternatives](#comparison-to-alternatives)
- [Conceptual Overview](#conceptual-overview)
  * [The Base Layer](#the-base-layer)
  * [The Annotation Layer](#the-annotation-layer)
  * [The Compression/Deduplication Layer](#the-compressiondeduplication-layer)
- [Low-Level Format](#low-level-format)
  * [Canonical File-Based Implementation](#canonical-file-based-implementation)
  * [Other Implementations](#other-implementations)
  * [Versioning](#versioning)
- [Database Format](#database-format)
  * [Column Types](#column-types)
  * [Tables for the Base Layer](#tables-for-the-base-layer)
  * [Tables for the Compression/Deduplication Layer](#tables-for-the-compressiondeduplication-layer)
  * [Tables for the Annotation Layer](#tables-for-the-annotation-layer)
- [Procedures](#procedures)
  * [Procedures for Recording](#procedures-for-recording)
  * [Procedures for Annotation](#procedures-for-annotation)


Design Goals
------------

The most central property the format is designed for is it being **online**, which is to say: *the archive must be
readable at any point in time, by multiple parties, even while a crawling session is in progress, and the database must
reflect the history of the active session recorded up to the present moment.*

The online nature of the archive has several benefits:

- It allows for on-the-fly debugging for long and/or partially interactive crawling sessions, particulary coupled with
  the next point:
- Multiple parties accessing the archive allows for separation of concerns during on-the-fly debugging: the crawler can
  focus only on recording the requests, while another program is used to display the trace as crawling progresses. This
  differs from, e.g. opening the Network tab in Chrome DevTools, where the same program is doing both the crawling and
  display, and you are stuck with the design chosen by the Chrome team and the rare updates thereof.
- Since the display program is independent of the crawler, you can come back later to a recording and open it with a
  more powerful parser; or even view the trace on-the-fly with multiple display programs, each specialized for a
  particular aspect.
- Separation of the crawler/display also enables debugging if the crawler is headless and/or on a different machine to
  that of the workstation where the display program is running.
- As long as the crawler is well-behaved (i.e. updates the archive as soon as events occur), you will have a nearly
  complete trace even if the crawler crashes at some point. Other formats, e.g. JSON-based, necessarily require the
  crawler to write the output file in one go at the end, and poor programming or special conditions (e.g. out of RAM)
  may cause this step to be omitted in the event of a crash, leading to complete data loss.
- An online-first design inherently implies that the archive will allow for efficient retrieval of small parts of its
  content (e.g. a specific request in a specific session), even in very large archives. Compare this to saving a trace
  to a JSON-based format: it is almost impossible to work with the data in a specific part of the trace without parsing
  the entire JSON file and loading the data into memory.

Other design goals include:

- **Open nature**: the format must be open to the public for perusal, free to use, and forkable if anyone wishes to
  develop it in a different direction to that of the original author

- **Portability**: the format must not depend on any specific browser, crawler, company, or proprietary standard thereof

- **Tolerance for incomplete data**: crawlers can vary widely as to which details are recorded and the precision
  thereof. Some crawlers may wish to record with millisecond precision all events in a request's lifetime (e.g. start,
  arrival of the first bytes in the response, completion etc). Others may be fine with just recording the start date of
  a request, with second-level precision, or not even bother with times at all. Sometimes we want all downloaded data
  (e.g. images) to be saved, sometimes only HTML pages, sometimes no response bodies at all. The format must accomodate
  all these situations.

- **Support for binary content**: the archive must support recording binary data for response bodies and POST data at
  the very least. Some other formats do not allow binary responses at all (only HTML, JSON etc) or assume that POST data
  is necessarily text-based.

- **Support for arbitrarily large archive sizes**: it should be possible for an archive to contain any number of
  sessions, with large amounts of downloaded data, the only limitation being disk space, not RAM

- **Support for annotations**: sessions, requests and other objects in the archive can be annotated with tags, comments,
  vendor-specific data, and even the results of analyses/transformations. For instance, we could attach the unminified,
  prettified version of the code to the body of a script request.

- **Extensibility**: related to the point above, the archive must be capable of storing vendor-specific
  per-session/per-request data that is not part of the initial design

- **Support for compression and deduplication**: there is a very high potential for compressibility in a crawling trace,
  given that many HTML/JSON/JS responses are text-based and have moderate entropy, and that many other details are the
  same or nearly the same between requests (e.g. headers, URLs). When storing multiple crawling sessions for the same
  site, there is an even greater potential for deduplication, as any requested fonts, scripts, images etc. will likely
  be exactly the same between sessions and should only be stored once.

- **Support for safe concurrency**: although support for safe concurrent access is rather a property of the programs
  working with the format, not the format itself, it should still be designed so as to enable rather than impede safe
  concurrent access, at least in the most basic case of one writer vs many readers (e.g. a crawler recording a session
  while several display/analysis programs are observing). Note that scenarios requiring more complex concurrency are
  possible: we could have a crawler recording (writer), with a display program observing (reader), while another
  analysis program is e.g. unminifying JS files and adding annotations on the fly (reader+writer).


Comparison to Alternatives
--------------------------

Before comparing OCTA to well-established alternatives such as HAR, let us first note that, given that the file system
is itself a sort of database, many of the design goals mentioned in the previous section could be met by a relatively
simple, ad-hoc file-based format, whereby e.g. each session is saved in a dedicated directory, and each request therein
is dumped to a file or directory (perhaps with separate files for dumping the response body, headers, metadata etc.).
Although there is no standardized crawling archive format of this kind, it is instructive to add it as a hypothetical
alternative to compare features with.

| Feature          |    OCTA     |   Filesystem-based   |       HAR       |
|:-----------------|:-----------:|:--------------------:|:---------------:|
| Online           |     Yes     |         Yes          |       No        |
| Open             |     Yes     |         No           |       Yes       |
| Portable         |     Yes     |         Yes          |       Yes       |
| Sessions         | Many / file | Many files / session | 1 / file [^ses] |
| Incomplete data  |     Yes     |         Yes          |       Yes       |
| Binary content   |     Yes     |         Yes          |  Partial [^bin] |
| Large archive    |     Yes     |         Yes          |     No [^lrg]   |
| Annotations      |     Yes     |         Yes          | Yes (primitive) |
| Extensibility    |     Yes     |         Yes          |       Yes       |
| Compression      |     Yes     |         Yes          |   Yes/No [^cmp] |
| Deduplication    |     Yes     |       No [^ded]      |       No        |
| Safe concurrency | Yes [^co1]  |    Partial [^co2]    |       No        |

[^ses]: Requests from multiple sessions can technically be stored in a single HAR file but there will be no explicit
        separation between the sessions (only the passage of time and/or some comment/custom field detailing each
        request's source)
[^bin]: POST data is assumed to be text-only
[^lrg]: All data is stored in a single JSON file, so the more requests and content, the more problematic it is to load
        into memory and parse
[^cmp]: There is no compression support in the standard itself, but the JSON file can be kept compressed with e.g. .gz
        and there are many programs and libraries that can open compressed JSON files directly
[^ded]: In theory deduplication could be achieved, at least for content bodies, by the use of hardlinks or special file
        systems with automatic deduplication support. Both of these create major portability issues. For the former
        option, there is also the difficult problem of efficiently keeping track of unique content across the entire
        archive so that we know whether e.g. incoming content is a duplicate and should be replaced with a hardlink, or
        not. At the very least, this would require some centralized database file which rather defeats the purpose of a
        simple file-based representation of the archive.
[^co1]: OCTA is a database format and most major database backends (SQLite, MySQL etc) have decent support for
        transactions and locking
[^co2]: Some filesystems support locking, but many do not. When locking is supported, it tends to be rather simple (e.g.
        only files, not whole directory trees), which requires extra programming effort when more complicated
        transactions are needed (e.g. we could use a global lock file etc)


Conceptual Overview
-------------------

The content in an OCTA archive can be seen as fitting in one of three functional layers:

1. **The base layer**: The core data we are interested in: sessions, HTTP requests, responses
2. **The annotations layer**: Tags, comments, analyses attached to base layer objects.
3. **The compression/deduplication layer**: Extra data (hashes etc) that enables the compression and deduplication
   functionality.

The latter two layers are entirely optional. A crawler need not add annotations, nor even support their handling in
general. Similarly, a simple crawler does not need to bother with deduplication or compression - another program can
process and deduplicate the archive later (or on the fly).

### The Base Layer

Data in the base layer is structured according to this hierarchy:

    Archive
      Sessions
        Tabs
          Requests
            Request data
            Response data

- The **archive** is a singleton entity representing the entire archive database. It contains any number of crawling
  sessions.
- A **session** object represents an entire crawling session, during which a site was accessed and navigated through. A
  session usually has a start and end date, and features only one site (other than the dependencies it pulls), though
  this is not enforced by the format.
- A **tab** object roughly corresponds to a tab open in a browser, wherein various URLs and pages may be opened during
  a crawling session (in Chrome parlance they are called "pages", but this is an ambiguous term). In a simple crawling
  session, most of the action will happen in a single tab, but in a more complex scenario, the page may open extra tabs
  for e.g. downloads, `target="_blank"` links etc.

  If crawling is done through a simple script instead of an instrumented browser, there will be no tabs as such in
  reality, but a "fake" tab object must still be created, as per the hierarchy.

  Tabs are also used to represent hidden service pages such as for extensions, etc., though generally, a crawler will
  not bother with capturing data from these.

- A **request** object represents an HTTP request over its lifetime, together with the response received, if any.

  Data stored about the request includes:

  - The tab from which it originated
  - The date/time when it was sent
  - A sequence number (indicating this is the Xth request issued)
  - The HTTP method (`GET`, `POST` etc)
  - The URL
  - The request headers
  - Any POST data that was sent

  Stored response information includes:

  - The request's ultimate fate (completed, failed due to network errors, aborted etc.); some conditions may result in
    no response being received at all, or just the headers etc.
  - The response's HTTP code and associated extra text (if a response arrived at all)
  - The response headers
  - The response body
  - If the request failed at the network/browser level, an error code and explanatory text
  - Timing information for events such as:
      - When the response started arriving (headers etc)
      - When the response arrived in full (body included)
      - When the request was deemed failed / aborted

#### External IDs

All the major objects in the database (sessions, tabs, requests etc) feature an optional field by which an
**external ID** may be set, which differs from the internal ID that is only used within the database. The external ID is
a string in a vendor-specific format that allows one to refer to objects in the archive in a stable way that survives
database reorganizations, consolidations, merging etc. Objects in two archives that have the same external ID should
generally be considered to be the same for the purpose of operations such as merging and comparison.

Note that external IDs are only unique within the scope of the immediate parent. Thus, session IDs are unique per
archive, but e.g. tab IDs are only unique within a session. The same tab ID can be reused for tabs appearing in
different sessions.

### The Annotation Layer

All important objects in the archive can have any number of annotations attached. This includes sessions, tabs and
requests, but also sub-objects such as headers and response bodies.

There are three core kinds of annotations:

- **Tags**: Tags allow objects to be easily grouped and searched according to some property they have in common, which
  is marked by the tag. For instance, in an archive containing many sessions, all sessions for a specific portal could
  have a tag like `portalXYZ` attached, and be easily retrieved by a query by that specific tag.

  Any number of tags may be attached to an object.

  Tags are defined as having a user-readable label and a machine-readable identifier. The latter acts as a sort of
  mandatory external ID. During merging, comparison etc., tags with the same identifier will be considered equivalent
  even if they have a different label in different archives.

- **Comments**: Comments are text snippets with an author and a date that can form a conversation around an object in
  the archive (if there are multiple people looking at the archive). In a single-user case, they can be used for storing
  research notes or TODOs.

- **Custom fields**: These are key-value pairs that can store any content not covered by this standard. The key is a
  string (usually a fully qualified domain name) that describes the type of the value, and the value is a binary blob
  whose interpretation is up to any program that can handle the specified type.

  For instance, an automated JavaScript deminifier would attach its analysis to the appropriate response body object
  with a key like e.g. `org.vendorXYZ.deminifier.deminified_body` and a value consisting of the deminified JavaScript
  content.

  As a rule, programs that do not handle a custom field type should just leave it as-is. Basic operations on the archive
  (moving etc.) will also preserve the fields of an object. The standard does not cover more complex situations like
  whether fields of the same type should be overwritten or merged during a merge, etc. It is best if complex operations
  on content with custom fields is handled by a program that understands the meaning of all said fields.

All annotations to an object also feature an **annotation date** and a reference to the **actor** that created the
annotation, the latter being either a person (described by their name) or a program. For instance, a comment could refer
to a "John Smith" as its author, while a deminified body custom field would be authored by a "XYZ deminifier" as its
actor.

Actors are defined in a registry contained in the archive. Like tags, actors have both a user-readable label and a
machine-readable identifier. Between archives, actors with the same identifier are considered to be equivalent.

### The Compression/Deduplication Layer

A key feature of the way compression and deduplication are implemented in the format is that they are **opt-in** on a
per-object basis. Some objects may be compressed, others not. Some instances of a piece of content may be deduplicated
while still allowing for duplicates of that content to exist. Simple programs may write content without having to handle
compression and deduplication at all, whereas aware programs can make use of the format's relevant features so as to
efficiently do compression and deduplication on-the-fly.

#### Compression

Currently compression is only supported for response bodies and POST data. Each response body can be stored using one of
these supported compression methods:

- **no compression**: the data is stored as-is, uncompressed
- **DEFLATE**: the data is compressed as per the standard DEFLATE algorithm introduced by Zip and used in `gzip`, the
  standard `gzip` HTTP compression method, etc.

Parties capable of compression may still want to leave certain objects uncompressed, particularly incompressible data
such as JPEGs, where attempting to compress it may actually increase the size.

#### Deduplication

Deduplication is supported for the following types of entities:

- URLs [^url]
- Response bodies
- POST data
- Request HTTP header names
- Request HTTP header values
- Response HTTP header names
- Response HTTP header values
- HTTP status messages
- Request failure descriptions

The OCTA format provides support for deduplication in two ways:

- Being a database format, all objects and sub-objects are entries in a table under some given ID. Deduplicating e.g. a
  response body is a trivial matter of making two requests refer to the same ID in the "response bodies" table.
- All deduplicable entities are provided with a "hash" field where a hash of the content can be stored by an aware
  program. Before trying to add a piece of content, another aware program can check whether there is already some
  content with a matching hash, and use the ID of the existing content instead.

The benefits of deduplication can be massive. In one experiment, a rudimentary filesystem-based archive counting several
tens of gigabytes was reduced to a single SQLite database file less than 200MB in size (and with far superior querying
facilities to boot).

[^url]: Not only are URLs frequently repeated, but modern sites also feature data URLs that can be very large.
        Deduplicating these is a must.


Low-Level Format
----------------

OCTA is a *database format* as opposed to a *file format*, and thus describes the structure of a database as opposed to
the binary structure of a file. That said, this standard does specify a "canonical", "default" database implementation
that allows one to store an OCTA archive as a single file.

### Canonical File-Based Implementation

In the canonical implementation, an OCTA archive can be stored as a SQLite database file (version 3 minimum), with any
of these extensions:

- `.db`, `.sqlite`, `.db3` (the typical SQLite database extensions)
- `.octa`
- `.octa.db` etc.

Additionally, the database must contain a table named `meta` with the following columns:

| Name  | Type | Nullable | Other Properties |
|:------|:-----|:--------:|:-----------------|
| key   | TEXT |   No     | PRIMARY KEY      |
| value | TEXT |   Yes    |                  |

And at least the following two entries:

| `key`   | Contents of `value`                           |
|:--------|:----------------------------------------------|
| type    | The exact string `org.atmfjstc.octa_format`   |
| version | The version of the format used (e.g. `0.1.0`) |

The other tables in the database are described by the OCTA database format proper in the sections below.

### Other Implementations

Other methods of implementing the archive format in a database include:

- Storing the archive as a dedicated database on a SQL server like MySQL, PostgreSQL, Microsoft SQL Server etc. The
  format makes use only of basic SQL features that are common to all these dialects.
- Embedding the archive in an existing database. It is acceptable and recommended to alter the table names (e.g. by
  adding a prefix like `octa_` or `archive_` etc.) so that they are grouped together when inspecting the database.

### Versioning

Versioning is particularly important when OCTA archives are shared in SQLite form, as this is when it is most likely
that a program designed for a particular version will encounter an archive built using an earlier version of the format.

The versioning scheme observed is: *major* `.` *minor* `.` *patch* , where:

- *major* is advanced when the format undergoes breaking changes or is otherwise fundamentally rewritten. There is no
  expectation of backward compatibility between major versions.

  As an exception, advancing from major version 0 to 1 does entail backward compatibility - it only signifies
  progressing past the alpha / rapid development stage.

- *minor* is advanced when significant, but non-breaking changes are made. A program should be able to handle an OCTA
  archive of an earlier *minor* version as long as the *major* version is still the same.

- *patch* is advanced when the designed functionality remains the same but a bug or oversight has been fixed

Note that earlier versions may be missing columns or tables vs later versions. A program should be aware of this when
assembling its SQL queries, as most SQL implementations cannot automatically "fill in defaults" when it comes to whole
tables or column definitions.


Database Format
---------------

We arrive finally at the database format proper.

### Column Types

Despite most SQL dialects offering similar *functionality* regarding data types, the exact *syntax* for specifying those
data types still differs by quite a lot between dialects. This is particularly relevant for SQLite, which has a much
looser typing system, where complex types such as sets, dates etc. all resolve to either integer, float, string or
blob (as seen [here](https://www.sqlite.org/datatype3.html)). As OCTA is intended to work with many SQL dialects, we
cannot specify columns using a "standard" syntax as we do for tables etc. - because there isn't any -, instead we need
to cover the exact syntax for SQLite, MySQL and other major dialects. Since OCTA only makes use of a handful of distinct
types, we will cover them all there, together with all their translations to major dialects, and specify them only as
"abstract" data types in the sections to follow.

| "Abstract" type | SQLite  | MySQL        | PostgreSQL | MS SQL Server           | Notes                                           |
|:----------------|:--------|:-------------|:-----------|:------------------------|:------------------------------------------------|
| serial          | INTEGER | INT(10)      | integer    | INT                     | Int used as surrogate ID for tables             |
| integer         | INTEGER | INT(10)      | integer    | INT                     | General-purpose integer (HTTP codes etc)        |
| boolean         | INTEGER | TINYINT(1)   | boolean    | BIT                     | "True or false" value                           |
| timestamp       | TEXT    | DATETIME(3)  | timestamp  | DATETIME2               | UTC timestamp with millisecond precision [^utc] |
| string(N)       | TEXT    | VARCHAR(N)   | varchar(N) | NVARCHAR(N)             | Text of limited length (usually < 1000) [^utf]  |
| text            | TEXT    | LONGTEXT     | text       | NTEXT or NVARCHAR(MAX)  | Text data of any length [^len]                  |
| bytes(N)        | BLOB    | VARBINARY(N) | bytea      | VARBINARY(N)            | Binary data of limited length                   |
| blob            | BLOB    | LONGBLOB     | bytea      | IMAGE or VARBINARY(MAX) | Binary data of any length                       |


[^utc]: All timestamps stored in an OCTA database should be assumed to be referenced to UTC (if the dialect does not
        provide a way of specifying this explicitly)
[^utf]: Unless otherwise noted, all text data should be assumed to be UTF-8 encoded (or any other Unicode variant
        available for the dialect).
[^len]: In practice, nearly all SQL engines have a limit of 1 or 2 GB for binary or text values, and thus an OCTA
        database cannot store e.g. response bodies that are larger than that after compression. While the format could
        be adjusted to account for this possibility, this is not a priority as downloading such huge files during
        crawling is not a typical or recommended scenario.

### Tables for the Base Layer


#### Table: `sessions`

##### Columns

| Name        | Type        | Nullable | Other Properties                  |
|:------------|:------------|:--------:|:----------------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT [^ain] |
| external_id | string(200) |   Yes    |                                   |
| start_time  | timestamp   |   Yes    |                                   |
| end_time    | timestamp   |   Yes    |                                   |


[^ain]: Some SQL dialects have a different way of defining an auto-incrementing surrogate ID column. For PostgreSQL, for
        instance, its native `serial` datatype should be used instead.

##### Indexes

| Column(s)   | Other Properties | Notes                                                                    |
|:------------|:-----------------|:-------------------------------------------------------------------------|
| external_id | UNIQUE           | Mandate external ID uniqueness and allow finding sessions by external ID |
| start_time  |                  | For efficiently sorting sessions by date                                 |
| end_time    |                  | Find sessions that are still open                                        |


#### Table: `tabs`

##### Columns

| Name        | Type        | Nullable | Foreign Key To | Other Properties           |
|:------------|:------------|:--------:|:---------------|:---------------------------|
| id          | serial      |   No     |                | PRIMARY KEY, AUTOINCREMENT |
| session_id  | serial      |   No     | sessions.id    |                            |
| external_id | string(200) |   Yes    |                |                            |
| type        | string(24)  |   Yes    |                |                            |
| time_open   | timestamp   |   Yes    |                |                            |
| time_closed | timestamp   |   Yes    |                |                            |
| parent_id   | serial      |   Yes    | tabs.id        |                            |

Note: unless otherwise specified, all foreign key references should be defined with `ON DELETE CASCADE` and
`ON UPDATE CASCADE`.

Column notes:

- The `type` column allows distinguishing between "real" tabs and background pages and the like. The following values
  are defined:

  | Value           | Meaning                             |
  |:----------------|:------------------------------------|
  | page            | Real, visible tab                   |
  | background_page | Background page for extensions etc. |
  | service_worker  | Self-explanatory                    |
  | shared_worker   | Self-explanatory                    |

  Other browser-specific types may also occur in practice (the OCTA format does not mandate that this enumeration is
  closed).

  The value can also be `NULL` if unknown or unimportant. By default, one should assume that a tab is "real" unless
  otherwise specified.

- The `parent_id` identifies the tab that opened this one, if known. Note that a value of `NULL` does not imply that the
  tab was not opened by another, as some crawlers cannot or will not record this information.

##### Indexes

| Column(s)               | Other Properties | Notes                                                                            |
|:------------------------|:-----------------|:---------------------------------------------------------------------------------|
| session_id, external_id | UNIQUE           | Mandate external ID uniqueness per session and allow finding tabs by external ID |


#### Table: `requests`

##### Columns

| Name                  | Type        | Nullable | Foreign Key To   | Other Properties           |
|:----------------------|:------------|:--------:|:-----------------|:---------------------------|
| id                    | serial      |   No     |                  | PRIMARY KEY, AUTOINCREMENT |
| tab_id                | serial      |   No     | tabs.id          |                            |
| external_id           | string(200) |   Yes    |                  |                            |
| sequence_no           | integer     |   Yes    |                  |                            |
| method                | string(24)  |   Yes    |                  |                            |
| url_id                | serial      |   Yes    | urls.id          |                            |
| post_data_id          | serial      |   Yes    | bodies.id        |                            |
| time_started          | timestamp   |   Yes    |                  |                            |
| is_navigation         | boolean     |   Yes    |                  |                            |
| fetch_type            | string(64)  |   Yes    |                  |                            |
| response_arrived      | boolean     |   Yes    |                  |                            |
| time_response_arrived | timestamp   |   Yes    |                  |                            |
| http_code             | integer     |   Yes    |                  |                            |
| status_text_id        | serial      |   Yes    | status_texts.id  |                            |
| body_id               | serial      |   Yes    | bodies.id        |                            |
| is_failed             | boolean     |   Yes    |                  |                            |
| failure_text_id       | serial      |   Yes    | failure_texts.id |                            |
| is_complete           | boolean     |   Yes    |                  |                            |
| time_finished         | timestamp   |   Yes    |                  |                            |

Column notes:

- `method`: This contains the HTTP method (e.g. `GET`, `POST` etc.), normally in uppercase, but OCTA does not mandate
  this. Unusual/custom HTTP methods can also be specified.
- `is_navigation`: True if this is a navigation request, i.e. one that is expected to cause a new document to be loaded
  in the tab. By contrast, requests for stylesheets, scripts, AJAX etc. are non-navigation requests and should have
  `false` for this field. `NULL` should be used if the nature of the request is unknown/unspecified.
- `fetch_type`: Further indication as to the destination/meaning of this request, as per the constants defined in Chrome
  [here](https://chromedevtools.github.io/devtools-protocol/tot/Network/#type-ResourceType):

  | Value              |
  |:-------------------|
  | document           |
  | stylesheet         |
  | image              |
  | media              |
  | font               |
  | script             |
  | texttrack          |
  | xhr                |
  | fetch              |
  | prefetch           |
  | eventsource        |
  | websocket          |
  | manifest           |
  | signedexchange     |
  | ping               |
  | cspviolationreport |
  | preflight          |
  | other              |

  Other values can also be stored.

- `response_arrived`: True if any part of the response (even headers) ever arrived. Similary, `time_response_arrived`
  marks the time when the response started arriving (headers were received), as opposed to when the response body
  was completely received (which may well never occur)
- `status_text_id`: A reference to a deduplicated table of status texts (the optional text that may follow the HTTP
  status code; sometimes this contains extra info on why e.g. a resource could not be accessed). Empty statuses should
  never be stored, rather `NULL` should be used instead.
- `is_failed`: True if the request failed due to a network error or was aborted. This usually means that there will be
  no response as such at all (unless it was aborted while the response body was still being received). The request will
  **not** be considered failed in this sense if an error-indicating HTTP code (400, 500 etc) was received.
- `failure_text_id`: A short text (usually just an error code string) providing extra information on why a request
  failed. This allows distinguishing between aborted requests and those that encountered e.g. network errors.
- `is_complete`: True if the request is complete due to either succeeding (i.e. the response body being fully received)
  or failing due to network errors or being aborted. A false value should be used as an indication that the crawler
  process stopped recording before the fate of this request could be ascertained.
- `time_finished`: The time when the request completed (either by receiving the entire response body, or failing).

Note that it is acceptable to have a request object with no URL or method defined, even though no real-life request can
exist without these. This is to allow crawlers to "preallocate" a request object and its ID as soon as a request is
detected, allowing them to fill in the rest of the details later (if e.g. doing URL deduplication, this might take some
time). A display/processing program should skip processing such "empty" requests if it encounters them.

##### Indexes

| Column(s)               | Other Properties | Notes                                                                            |
|:------------------------|:-----------------|:---------------------------------------------------------------------------------|
| tab_id, external_id     | UNIQUE           | Mandate external ID uniqueness per tab and allow finding requests by external ID |
| tab_id, sequence_no     |                  | Efficiently sort requests through time (within a tab)                            |
| tab_id, time_started    |                  | Efficiently sort requests through time (within a tab)                            |
| tab_id, is_complete     |                  | Find still-pending requests (within a tab)                                       |

Note: more indexes are likely to be added here in future versions of the format, as more efficiency requirements are
revealed in practice.


#### Table: `request_headers`

This table stores the HTTP request headers for each request, as deduplicated key/value pairs.

##### Columns

| Name            | Type   | Nullable | Foreign Key To           | Other Properties           |
|:----------------|:-------|:--------:|:-------------------------|:---------------------------|
| id              | serial |   No     |                          | PRIMARY KEY, AUTOINCREMENT |
| request_id      | serial |   No     | requests.id              |                            |
| header_name_id  | serial |   No     | request_header_names.id  |                            |
| header_value_id | serial |   No     | request_header_values.id |                            |

**Important note** regarding HTTP headers that can take multiple values: it is possible to add multiple entries for a
request with the same `header_name_id`, thus defining multiple values for a header. However, there is no explicit
ordering between entries in this table, thus this can only be done for HTTP headers where the ordering is not
significant between the multiple values (e.g. `Accept:`). If the ordering is significant, only one entry should be
added for the header, with a value featuring a comma-separated list as per the HTTP spec.

##### Indexes

| Column(s)                        | Other Properties | Notes                                                                   |
|:---------------------------------|:-----------------|:------------------------------------------------------------------------|
| request_id                       |                  | Find the headers of a particular request                                |
| header_name_id, header_value_id  |                  | Efficiently find all headers of a given type or a given key-value combo |
| header_value_id                  |                  | Efficiently find all headers featuring a given value                    |


#### Table: `request_header_names`

##### Columns

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| name        | string(200) |   No     |                            |

Note that for the purposes of deduplication, programs can query by `name` directly to check for duplicates, as the field
data is small enough to allow for direct indexing.

##### Indexes

| Column(s) | Other Properties | Notes                         |
|:----------|:-----------------|:------------------------------|
| name      |                  | Index header names by content |


#### Table: `request_header_values`

##### Columns

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| value       | text        |   No     |                            |
| hash_sha256 | bytes(32)   |   Yes    |                            |

The `hash_sha256` column is part of the deduplication layer and contains the SHA-256 hash of the header value, after
encoding as UTF-8. The hash enables deduplication by allowing for programs to look up a value by content, even for very
large values that do not allow for direct indexing.

The hash is of course optional - unhashed values will simply not participate in the deduplication process, and likely
duplicates of them will accumulate in the table. A program can of course also look for duplicates using a linear
non-indexed scan, but this is not recommended.

##### Indexes

| Column(s)   | Other Properties | Notes                             |
|:------------|:-----------------|:----------------------------------|
| hash_sha256 |                  | Index header values by value hash |


#### Table: `response_headers`

This table works just like `request_headers`, but stores the headers for the HTTP response associated with a request.

##### Columns

| Name            | Type   | Nullable | Foreign Key To            | Other Properties           |
|:----------------|:-------|:--------:|:--------------------------|:---------------------------|
| id              | serial |   No     |                           | PRIMARY KEY, AUTOINCREMENT |
| request_id      | serial |   No     | requests.id               |                            |
| header_name_id  | serial |   No     | response_header_names.id  |                            |
| header_value_id | serial |   No     | response_header_values.id |                            |

##### Indexes

| Column(s)                        | Other Properties | Notes                                                                   |
|:---------------------------------|:-----------------|:------------------------------------------------------------------------|
| request_id                       |                  | Find the response headers for a particular request                      |
| header_name_id, header_value_id  |                  | Efficiently find all headers of a given type or a given key-value combo |
| header_value_id                  |                  | Efficiently find all headers featuring a given value                    |


#### Table: `response_header_names`

This table works just like `request_header_names`, but provides a separate deduplication scope for the names of HTTP
response headers.

##### Columns

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| name        | string(200) |   No     |                            |

##### Indexes

| Column(s) | Other Properties | Notes                         |
|:----------|:-----------------|:------------------------------|
| name      |                  | Index header names by content |


#### Table: `response_header_values`

This table works just like `request_header_values`, but provides a separate deduplication scope for the values of HTTP
response headers.

##### Columns

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| value       | text        |   No     |                            |
| hash_sha256 | bytes(32)   |   Yes    |                            |

##### Indexes

| Column(s)   | Other Properties | Notes                             |
|:------------|:-----------------|:----------------------------------|
| hash_sha256 |                  | Index header values by value hash |


#### Table: `bodies`

This table stores the content for both HTTP response bodies and request POST data alike.

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| content     | bytes       |   Yes    |                            |
| size        | integer     |   Yes    |                            |
| compression | string(16)  |   Yes    |                            |
| hash_sha256 | bytes(32)   |   Yes    |                            |

Column notes:

- `content`: The content of the response/POST data is stored here, either compressed or uncompressed. Note that even
  uncompressed, text-based responses are still stored as bytes. The text encoding should be retrieved from the
  `Content-Type` HTTP header, or assumed to be UTF-8 otherwise.

  Note that the `content` can be NULL if the crawler does not need or want to store the response data (for reasons of
  space, privacy etc). In this case, the `size` and/or `hash_sha256` fields can still be filled in so as to provide at
  least some essential information regarding the response.

- `size`: The size of the response, in bytes. This always refers to the *uncompressed* size. The field can be NULL in
  which case the size can be computed from the `content` if available.

- `hash_sha256`: Similarly to `size`, the hash is always computed on the *uncompressed* content.

- `compression`: The following values are supported:

  | Value        | Meaning                                                                    |
  |:-------------|----------------------------------------------------------------------------|
  | uncompressed | `content` contains the uncompressed response as-is                         |
  | (NULL)       | Same as `uncompressed`                                                     |
  | deflate      | `content` contains the response compressed using DEFLATE (see notes below) |

**Note** regarding DEFLATE compression: the DEFLATE algorithm is well-known and beyond the scope of this document, being
instead described in e.g. [RFC 1951](https://datatracker.ietf.org/doc/html/rfc1951). For easier compatibility with other
libraries and languages, the `content` field does *not* store a raw DEFLATE stream, but rather adds zlib-specific
headers and trailers as per [RFC 1950](https://datatracker.ietf.org/doc/html/rfc1950).

##### Indexes

| Column(s)   | Other Properties | Notes                           |
|:------------|:-----------------|:--------------------------------|
| hash_sha256 |                  | Index responses by content hash |


#### Table: `urls`

This table stores URLs (which may be quite large, e.g. if they are data URLs).

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| url         | text        |   No     |                            |
| hash_sha256 | bytes(32)   |   Yes    |                            |

As usual, `hash_sha256` is the hash of the URL value, after UTF-8 encoding.

##### Indexes

| Column(s)   | Other Properties | Notes                      |
|:------------|:-----------------|:---------------------------|
| hash_sha256 |                  | Index URLs by content hash |


#### Table: `status_texts`

As for header names, duplicates can be checked for using the value field directly, no need for a hash.

##### Columns

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| value       | string(250) |   No     |                            |

##### Indexes

| Column(s) | Other Properties | Notes                         |
|:----------|:-----------------|:------------------------------|
| value     |                  | Index status texts by content |


#### Table: `failure_texts`

Functions similarly to `status_texts`.

##### Columns

| Name        | Type        | Nullable | Other Properties           |
|:------------|:------------|:--------:|:---------------------------|
| id          | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| value       | string(250) |   No     |                            |

##### Indexes

| Column(s) | Other Properties | Notes                          |
|:----------|:-----------------|:-------------------------------|
| value     |                  | Index failure texts by content |


### Tables for the Compression/Deduplication Layer

There are currently no tables specifically dedicated to the compression/deduplication functionality. All such functions
are achieved through the `hash_sha256` and `compression` columns described previously.


### Tables for the Annotation Layer


#### Table: `referenced_objects`

This table provides a layer of indirection that is necessary to handle some of the complications resulting from there
being many different types of objects that can be annotated: sessions, tabs, requests, bodies etc.

In this table, and only in this table, there is a foreign key for each of the object types that allow annotations. The
type-specific surrogate IDs are all associated with another unique "type-agnostic" annotatable object ID, and it is the
latter that the tags, comments etc. are attached to. Thus we avoid having to repeat the foreign key set for each type
of annotation (as well as the merging etc. complications that result).

Notes:

- Not all objects in the database need to have an entry here, only those for which an annotation exists
- The table is subject to an invariant (not enforced at the SQL level): for each row, exactly one of the foreign keys
  must be non-NULL.
- Other solutions could have been chosen, e.g. an inheritance pattern whereby the sessions, tabs etc. tables are
  considered subtypes of an "annotatable object" superclass and the tables are structured accordingly. However the
  chosen solution has the advantage that the annotation layer tables etc. can be completely omitted without any change
  being necessary to the main layer definitions.

##### Columns

| Name                   | Type   | Nullable | Foreign Key To            | Other Properties           |
|:-----------------------|:-------|:--------:|:--------------------------|:---------------------------|
| id                     | serial |   No     |                           | PRIMARY KEY, AUTOINCREMENT |
| session_id             | serial |   Yes    | sessions.id               |                            |
| tab_id                 | serial |   Yes    | tabs.id                   |                            |
| request_id             | serial |   Yes    | requests.id               |                            |
| url_id                 | serial |   Yes    | url.id                    |                            |
| body_id                | serial |   Yes    | bodies.id                 |                            |
| request_header_val_id  | serial |   Yes    | request_header_values.id  |                            |
| response_header_val_id | serial |   Yes    | response_header_values.id |                            |

##### Indexes

| Column(s)               | Other Properties | Notes                          |
|:------------------------|:-----------------|:-------------------------------|
| session_id              | UNIQUE           | Enforce uniqueness             |
| tab_id                  | UNIQUE           | Enforce uniqueness             |
| request_id              | UNIQUE           | Enforce uniqueness             |
| url_id                  | UNIQUE           | Enforce uniqueness             |
| body_id                 | UNIQUE           | Enforce uniqueness             |
| request_header_val_id   | UNIQUE           | Enforce uniqueness             |
| response_header_val_id  | UNIQUE           | Enforce uniqueness             |


#### Table: `actors`

This table is an index of all the parties that authored annotations in the archive. The column `label` contains a
human-readable name or description, whereas `identifier` contains the unique, machine-readable ID, usually in FQDN
format.

Note that due to the unique constraint on the identifier, deduplication for actor objects is mandatory. It is best if a
program checks and creates its necessary actors ahead of time instead of during each tagging operation.

##### Columns

| Name                   | Type        | Nullable | Other Properties           |
|:-----------------------|:------------|:--------:|:---------------------------|
| id                     | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| identifier             | string(200) |   No     |                            |
| label                  | string(200) |   No     |                            |

##### Indexes

| Column(s)               | Other Properties | Notes                                       |
|:------------------------|:-----------------|:--------------------------------------------|
| identifier              | UNIQUE           | Search by identifier and enforce uniqueness |


#### Table: `tags`

This table is an index of all the tags that are *defined* in the archive (as opposed to the tag *assignments* which are
in another table).

The same considerations as for `actors` apply here.

##### Columns

| Name                   | Type        | Nullable | Other Properties           |
|:-----------------------|:------------|:--------:|:---------------------------|
| id                     | serial      |   No     | PRIMARY KEY, AUTOINCREMENT |
| identifier             | string(200) |   No     |                            |
| label                  | string(200) |   No     |                            |

##### Indexes

| Column(s)               | Other Properties | Notes                                       |
|:------------------------|:-----------------|:--------------------------------------------|
| identifier              | UNIQUE           | Search by identifier and enforce uniqueness |


#### Table: `object_tags`

This table contains the list of tags attached to annotatable objects.

Note that there is no ordering defined between the tags attached to an object (other than the tagging time).

##### Columns

| Name                   | Type      | Nullable | Foreign Key To            | Other Properties           |
|:-----------------------|:----------|:--------:|:--------------------------|:---------------------------|
| id                     | serial    |   No     |                           | PRIMARY KEY, AUTOINCREMENT |
| tag_id                 | serial    |   No     | tags.id                   |                            |
| actor_id               | serial    |   No     | actors.id                 |                            |
| time_tagged            | timestamp |   No     |                           |                            |
| item_id                | serial    |   No     | referenced_objects.id     |                            |

##### Indexes

| Column(s)               | Other Properties | Notes                                       |
|:------------------------|:-----------------|:--------------------------------------------|
| time_tagged             |                  | Search by date                              |


#### Table: `comments`

##### Columns

| Name                   | Type      | Nullable | Foreign Key To            | Other Properties           |
|:-----------------------|:----------|:--------:|:--------------------------|:---------------------------|
| id                     | serial    |   No     |                           | PRIMARY KEY, AUTOINCREMENT |
| comment                | text      |   No     |                           |                            |
| actor_id               | serial    |   No     | actors.id                 |                            |
| time                   | timestamp |   No     |                           |                            |
| item_id                | serial    |   No     | referenced_objects.id     |                            |

##### Indexes

| Column(s)               | Other Properties | Notes                                       |
|:------------------------|:-----------------|:--------------------------------------------|
| time                    |                  | Search by date                              |


#### Table: `custom_annotations`

This table contains the custom annotations (analyses, vendor-specific data etc.) that can be attached to an object.
Central to each entry are the `type` (a FQDN-like identifier) and the `value` blob whose interpretation, of course,
depends on the type.

Notes:

- It is possible in principle to attach multiple annotations of the same `type` to a given object (but they will not be
  ordered, other than by time)
- Annotations can have the value NULL

##### Columns

| Name                   | Type        | Nullable | Foreign Key To            | Other Properties           |
|:-----------------------|:------------|:--------:|:--------------------------|:---------------------------|
| id                     | serial      |   No     |                           | PRIMARY KEY, AUTOINCREMENT |
| type                   | string(200) |   No     |                           |                            |
| value                  | bytes       |   Yes    |                           |                            |
| actor_id               | serial      |   No     | actors.id                 |                            |
| time                   | timestamp   |   No     |                           |                            |
| item_id                | serial      |   No     | referenced_objects.id     |                            |

##### Indexes

| Column(s)               | Other Properties | Notes                                       |
|:------------------------|:-----------------|:--------------------------------------------|
| type                    |                  | Search by annotation type                   |
| time                    |                  | Search by date                              |


Procedures
----------

The following section outlines a series of procedures for basic operations on the archive. They are only suggestions -
any implementation is acceptable as long as it results in the correct data structure and is safe against concurrent
access.


### Procedures for Recording

#### Opening a Session

- INSERT a row in the `sessions` table
  - Optional: set the `external_id` field appropriately during creation
  - Optional but highly recommended: set the `start_time` field during creation
- Remember and use the autogenerated `id` of the created session for future operations on it

#### Closing a Session

- Optional, but highly recommended: UPDATE the row in the `sessions` table and set the `end_time` appropriately
- Note that once the `end_time` field is set, there is a strong expectation from other programs that no further changes
  will be made to the session other than annotations.

#### Recording a Tab Being Opened

- Note that a tab object must be created even if the crawler is a simple script instead of an instrumented browser
- INSERT a row in the `tabs` table, with the appropriate `session_id`
  - Optional but highly recommended: set the `time_open` field during creation
  - Optional: set the `external_id`, `type`, `parent_id` fields during creation if the information is available
    - The information can also be added later but it is usually available immediately if available at all
- Remember and use the autogenerated `id` of the created tab for future operations on it
- Optional but highly recommended: if tabs are not created in response to captured "tab opening" events, but rather
  created specifically, especially if creating a "default" tab, then an alternate procedure is recommended:

  - Lock the database exclusively
  - Check whether the default/specific tab already exists (if so, just use its `id`)
  - Otherwise, create the tab as per the normal procedure
  - Unlock the database

  This ensures safe concurrency if multiple threads in the crawler are likely to try to create a specific tab at the
  same time.

#### Recording a Tab Being Closed

- Optional, but highly recommended: UPDATE the row in the `tabs` table and set the `time_closed` appropriately
- As for sessions, once the `time_closed` field is set, there is a strong expectation from other programs that no more
  requests will be added to the tab

#### Recording a Just-Starting Request

- If the exact start time of the request is not available from the capture infrastructure, make a note of the current
  time before starting any other operation
- Ensure that the parent tab object is created and that you have its `id`
- Record the URL of the request using the `Recording a URL` procedure, and obtain the URL object `id`
- If applicable, record the POST data using the `Recording POST data` procedure, and obtain the `id` for the entry
- INSERT a row in the `requests` table with the appropriate `tab_id`
  - Ensure that these fields are set during creation: `method`, `url_id`, `post_data_id`
  - Optional: set these fields during creation if the information is available: `external_id`, `is_navigation`,
    `fetch_type`
  - Optional but highly recommended: set `time_started` (using the time noted previously if necessary), `sequence_no`
  - Optional but highly recommended: set the state fields `response_arrived`, `is_failed`, `is_complete` to false so as
    to reflect the initial situation
- Remember and use the autogenerated `id` of the created request for future operations on it
- Record the request HTTP headers using the `Recording Request Headers` procedure
  - If desired, one can start a transaction so that both the request row and its headers are added atomically. However
    note that even under normal conditions, the headers at the start of the request are not definitive. A viewer program
    should expect the header list to change with time after the request object has been created.

#### Recording a Just-Starting Request, with Preallocation

If it is important that a request object be added as soon as possible (especially so that we can get its `id`, or as a
signal to other threads), an alternate procedure can be performed to take advantage of the preallocation feature:

- Skip the recording of the URL and POST data for now
- INSERT a row in the `requests` table as before, with NULL for the `method`, `url_id` AND `post_data_id`
- The request `id` is now available
- Record the URL and POST data objects and obtain their `ids`
- UPDATE the request row so as to set the `method`, `url_id` and `post_data_id`
- Record the headers as before

#### Recording a URL, with Deduplication

- Obtain a SHA-256 hash of the URL value encoded as UTF-8
- Within a transaction (or with a database lock):
  - Look for an entry in the `urls` table with a matching `sha256_hash`
    - If found, just return the corresponding `id` and use it in the future
    - You can also manually check that the content in `url` matches if you do not trust the hash, but a collision is
      exceedingly unlikely
  - If not found, INSERT a new row in the `urls` table, setting the `url` and `hash_sha256` fields appropriately
  - Remember and use the autogenerated `id` of the created URL for future operations on it

Note: it is acceptable to use an in-memory hash-to-URL cache to avoid having to query the database every time, but note
that there can be many unique URLs over a session, and the cache itself still needs to be safeguarded against concurrent
access from multiple threads. It may be easier to just defer to the database.

#### Recording a URL, without Deduplication

A simpler procedure, for crawlers that emphasize simplicity or speed:

- INSERT a new row in the `urls` table, setting the `url` field appropriately. No hash will be set and duplicates will
  occur if the URL is added again.
- Remember and use the autogenerated `id` of the created URL for future operations on it

#### Recording POST Data, with Deduplication and Maybe Compression

- Convert the data to a bytes representation if it is rendered as text, using UTF-8 or the encoding specified in the
  HTTP headers
- Obtain the SHA-256 hash of the content
- Optional, if using compression:
  - Compress the data using standard DEFLATE
  - Check whether the compressed data is actually smaller (if the original is high entropy, compression may be
    ineffective)
    - If the compression is not worth it, leave the content uncompressed
- Using a similar check as for recording URLs, INSERT a new row in the `bodies` table, ensuring that `content`, `size`
  and `compression` are set appropriately during creation
- Remember and use the autogenerated `id` of the created POST data body for future operations on it
- The procedure without deduplication is similar to that for URLs

#### Recording POST Data without Explicit Content

If for privacy or space reasons we do not wish to store the POST data itself, but rather just its size, hash, or mere
fact that it was there, an alternate procedure is available:

- Optional: Compute the SHA-256 hash of the content, if we wish to record it
- INSERT a new row in the `bodies` table, with NULL `content` and setting `size` and/or `hash_sha256` if desired
- Optional: Deduplication can still be applied as per the previous procedures, if the SHA-256 hash is available for both
  the incoming and existing body entries
- Remember and use the autogenerated `id` of the created POST data body for future operations on it

#### Recording Request Headers, in a Once-Only Scenario

- For each header name + value pair:
  - Record the header name as per the procedure below and obtain its `id`
  - Record the header value as per the procedure below and obtain its `id`
- For each header name + value ID pair, INSERT a row into the `request_headers` table, filling in `request_id`,
  `header_name_id` and `header_value_id` appropriately
- Optional: One may want to perform the above insertions in a transaction together with creating the row in `requests`,
  if we want to avoid monitors reading the request object before the headers have been set
- Note that this procedure is only valid if the headers are only written once. More advanced crawlers may receive
  further headers subsequently and wish to update the list. A procedure for this situation is listed in a subsequent
  section (which must be applied for both the first and subsequent updates)

##### Subprocedure: Adding a Header Name, with Deduplication

- In a transaction:
  - Check whether a row with the same `name` exists in the `request_header_names` table (the index will help with this)
    - If true, just return and use the `id` of the existing row
  - Otherwise, INSERT a row in the `request_header_names` table with the appropriate `name` field
  - Remember and use the autogenerated `id` of the created header name for future operations on it

- If not using deduplication, the row can just be INSERTed directly and its `id` used as before
- An in-memory cache may be useful here if the synchronization issues are handled

Note: it is recommended to normalize header names (e.g. by lowercasing or titlecasing) before registering them but the
standard does not enforce this, nor should programs necessarily expect this to be the case in any encountered archive.
Deduplication utilities should consider header name normalization as a potential operation during their processing.

##### Subprocedure: Adding a Header Value

The procedure is analogous to that for inserting a new URL, using the table `request_header_values` and the column
`value` respectively.

#### Recording/Updating Request Headers in a Multi-Update Scenario

- In a transaction:
  - Obtain the list of new (name_id, value_id) pairs as before
  - If using deduplication:
      - SELECT from the `request_headers` table for the appropriate `request_id` so as to get the list of already
        inserted headers
      - Remove the already-existing headers from the list of pairs to be added
  - If not using deduplication:
      - DELETE from the `request_headers` table all entries for the appropriate `request_id`
  - INSERT rows into the `request_headers` table for the incoming (name_id, value_id) pairs

#### Recording a Response Arriving

- UPDATE the corresponding row in `requests` so as to update the fields: `response_arrived` (set to true), `http_code`
- Optional but highly recommended: also set the `time_response_arrived` field
- Optional: set the `status_text_id` field if there is any extra HTTP status text
  - To obtain an `id` for the status text add it via a procedure similar to that for registering a HTTP header name
    (using the `status_texts` table and the `value` column respectively)
- Record the headers of the response, using a procedure analogous to that for the request headers (except using the
  `response_*` tables instead)

#### Recording a Fully Arrived Response

- Obtain an ID for the response body data, using the same procedure as for recording POST data
- UPDATE the corresponding row in `requests` so as to update the fields: `is_failed` (set to false), `is_complete` (set
  to true), `body_id`
- Optional but highly recommended: also set the `time_finished` field
- Note that once `is_complete` is set, there is a strong expectation from other programs that no further changes will
  be made to the request object (other than annotations)

#### Recording a Request Failure

- UPDATE the corresponding row in `requests` so as to update the fields: `is_failed` (set to true), `is_complete` (set
  to true)
- If the failure occurred before any response arrived, also set `response_arrived` to false
- Optional but highly recommended: also set the `time_finished` field
- Optional: set the `failure_text_id` field if there is any failure text
  - To obtain an `id` for the failure text add it via a procedure similar to that for registering a HTTP header name
    (using the `failure_texts` table and the `value` column respectively)


### Procedures for Annotation

#### Adding a Comment

- Before doing anything else, make a note of the current time so that the comment time is accurate (preparatory
  operations may take a bit of time)
- Ensure that the corresponding actor has been registered as per the procedure in a following section, and that you
  have its `id`
- Ensure that a typeless annotatable object reference has been created for the object you are trying to comment on, as
  per the procedure in a following section, and that you have its `id`
- INSERT a row into the `comments` table with the appropriate `comment`, `actor_id`, `time` and `item_id`

#### Registering an Actor

- In a transaction:
  - Check whether a row already exists in the `actors` table with the same `identifier`
    - If so, just return and use the `id` of the already defined actor
  - Otherwise, INSERT a row into the `actors` table with the appropriate `identifier` and `label`
  - Remember and use the autogenerated `id` of the actor for future operations on it

#### Obtaining an Annotatable Object Reference

- In a transaction:
  - Check whether a row already exists in the `referenced_objects` table for that object. Depending on the type of the
    entity you are annotating, you will use one and only one of the following conditions:

    | Entity Type                      | Predicate                    |
    |:---------------------------------|:-----------------------------|
    | Session                          | `session_id=...`             |
    | Tab                              | `tab_id=...`                 |
    | Request (or associated response) | `request_id=...`             |
    | URL                              | `url_id=...`                 |
    | Response body or POST data       | `body_id=...`                |
    | Request header value             | `request_header_val_id=...`  |
    | Response header value            | `response_header_val_id=...` |

  - If found, just return and use the `id` of the already created reference
  - Otherwise, INSERT a row into the `referenced_objects` table with a single column set, as per the entity type
    (`session_id`, `tab_id` etc.)
  - Remember and use the autogenerated `id` of the reference for future operations on it

#### Adding a Tag

- Before doing anything else, make a note of the current time so that the tagging time is accurate (preparatory
  operations may take a bit of time)
- Ensure that the corresponding actor has been registered and that you have its `id`
- Ensure that the corresponding tag type has been registered and that you have its `id`. The procedure is analogous
  to that for registering an actor (using the table `tags`)
- Ensure that a typeless annotatable object reference has been created for the object you are trying to comment on, and
  that you have its `id`
- INSERT a row into the `object_tags` table with the appropriate `tag_id`, `actor_id`, `time_tagged` and `item_id`

#### Adding a Custom Annotation

- Before doing anything else, make a note of the current time so that the annotation time is accurate (preparatory
  operations may take a bit of time)
- Ensure that the corresponding actor has been registered and that you have its `id`
- Ensure that a typeless annotatable object reference has been created for the object you are trying to comment on, and
  that you have its `id`
- INSERT a row into the `custom_annotations` table with the appropriate `type`, `value`, `actor_id`, `time` and
  `item_id`
- Optional: for certain annotation types, you may want to do this in a transaction, and also check for already existing
  annotations, so that they may be deleted or replaced if appropriate. E.g. an analysis annotation may need to be
  replaced if the analysis is repeated at a later date.
