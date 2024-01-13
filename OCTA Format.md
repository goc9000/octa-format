OCTA Format
===========

Version: 0.0.0


Summary
-------

OCTA (Online Crawling Trace Archive) is a database format for storing the complete history ("trace") of Web crawling
sessions.



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
