---
title: "Exploring NLP Parsed Audit Documents"
date: 2017-11-25T00:00:00+00:00
draft: false
---

Inspiration
========
This project came about for these reasons:

 1.  Make a positive impact on the world.
    *  Enable understanding through exploration of hard data as opposed to adhering to a particular belief system.
    *  A first project to gain some experience working with real world open data.
 2.  Learn some of the technologies used by TravelPerk so I could hit the ground running. 
 3.  Professional Development - gain experience with some of the latest technologies.

The previous version of the site included out-of-the-box Postgres text search functionality, already very powerful.  Only documents through mid 2014 were included in the [.zip file available for download](https://archive.org/details/usinspectorsgeneral) from archive.org.  Searching through the ~35,000 audit documents for the names of my favorite politicians was fun but not not particularly insightful.  The goal become how to glean information from such a vast corpus without a huge investment in building and tuning and AI system.

Final Product
=============

After some probing I came across [NLTK](http://www.nltk.org/) - a Python library that may be used to parse raw text and reveal it's structure.  Of particular interest was [Named Entity Recognition](http://www.nltk.org/book/ch07.html) (see chapter 5).  Since the context is set in this case as Audit Documents any named entity within a document would be of interest.  Combined with the inspiration that [simple frequency distribution](http://www.nltk.org/book/ch01.html) often yields insight a plan was hatched to explore the audit document corpus based on frequency of Named Entities.  After some more thought the plan was refined as follows:

 1.  Extract all named entities from each audit document along with their frequency within that document.
 2.  Across all audit documents determine what the most frequently occuring named entities are.  Provide an ordered list to the user of the most N frequently occuring named entities across audit documents (each document contributes a count of 1 towards a named entity if that named entity occurs more than K times in the document).
 3.  Allow the scope to be limited to a user selected set of years so the user can see how subject matter changes across years.
 4.  Enable exploration through refinement.  After a user selects a named entity the scope is then limited only to Audit Documents that contain that named entity.  

The above exploration process allows the user to find a small number of audit documents who all contain the set of named entities the user is interested in.  It turns out this will result in a set of documents related to a specific concern.  Here is an example search across Audit Documents in 2017:

Landing Page
-------------------

![Named Entity Landing](/images/NE_landing_s_e_s.png)

Click the "Named Entity Exploration" Button...

Start of Named Entity Exploration
----------------------------------------------


![All Documents](/images/NE_start_s_e.png)

Click the "Veterans" button...

Documents containing "Veterans"
-----------------------------------------------


![Documents with "Veterans"](/images/NE_veterans_only_e_s.png)

Click the "VISN" button...

Narrow scope for documents also containing "VISN"
------------------------------------------------------------------------


![Documents also with "VISN"](/images/NE_veterans_VISN_e_s.png)

Final Result
----------------
Including:

 *  Medical Center
 *  OSC: Acronym for the [U.S. Office of Special Council](https://osc.gov/Pages/about.aspx), who investigate whistle blower complaints.


<img src="/images/NE_veterans_end_e.png" width=100%/>

The result is a consistent set of Audit Documents investigating the performance of Medical Centers providing care for veterans across various cities.

---

Offline Processing
==================

As mentioned before, the archive at archive.org only had up through 2014.  Obtaining more recent reports required using the [software that created the archive in the first place](https://github.com/unitedstates/inspectors-general#inspectors-general).  This software has knowledge of what Inspector General websites exist and how to search each one for audit documents.  Each audit document is downloaded, converted to raw text (often poorly, a topic for another post), and parsed for basic meta information (title, publication date, etc).  A directory with these files is created for each report with the overall directory structure reflecting the publication year and which website the report came from.  Running the tool resulted in a total of 51,295 audit documents to process.  The metadata would be used to obtain title, publication date, and source url while the raw text would be parsed for named entities and indexed by Postgres for text search.  Only the distilled data from this processing would be included on the site, to view the document the origin url would be provided to the end user.

Parsing each audit document using NLTK turned out to be the bottleneck, up to 10 seconds or so for a particularly large text.  To finish in a reasonable amount of time it became clear that all 4 of my raging CPU cores (Phenom II X4 965) would need to be fully utilized.  Python offers many options for concurrent execution, including asynchronous programming within a thread (supported by the language) and [multithreading and multiprocessing](https://docs.python.org/3.6/library/concurrency.html).  In a previous [article](http://randalmoore.me/asynchronous-programming/) I explained the differences between these.  In this case since the tasks are CPU bound and taking into consideration the [Python Global Interpreter Lock](https://wiki.python.org/moin/GlobalInterpreterLock) the best option was to use multi processing.  

[This file](https://github.com/RandyMoore/mySiteDjango/blob/master/my_site_django/upload_audit_docs.py) populates the Postgres database with metadata, text search, and named entity data for each audit document.  It searches through a directory structure at a given root and processes the files associated with each audit document if that data isn't already in the database.  The documents are processed in parallel, one per core of the host machine.

The write of results to Postgres was originally written to occur in the same task after processing the document but this resulted in Django layer concurrency errors.  It would have been complicated and inefficient to coordinate the DB as a shared resource so I opted instead to use a queue.  Each document processing task places the result into a queue.  An additional DB dedicated process consumes from the queue and writes each result to the database sequentially.  The final code was surprisingly simple:

Top of file:

```python
db_queue = None # declared as global to be shared between processes
```
    
In \_\_main\_\_ create the DB queue and start it's process:

```python
db_queue = multiprocessing.Manager().Queue() # Special queue provided for IPC use
multiprocessing.Process(target=save_to_db).start() # DB dedicated process
```

In process_documents() the results are placed in the queue:

```python
db_queue.put((doc, named_entities))
```
    
In save_to_db() we loop forever consuming from the queue until None is encountered:

```python
while True: 
    doc_tuple = db_queue.get()  # blocks if queue is empty
    <exit function if None found>
    <save to DB>        
```    

In \_\_main\_\_ all cores of the host machine are loaded up using a multiprocessing Pool and once they are finished None is placed in the DB queue to be found by the DB task causing it to exit:

```python
with multiprocessing.Pool() as p: # by default creates as many processes as available cores.
    p.map(func=process_documents, iterable=files) # Processes to process documents

db_queue.put(None)
```

With this my machine was steadily maxed at 100% processor utilization, and took close to a full day to process all of the documents.  A fair amount of electricity was used but at least some neat Python tricks were explored and cool words such as "corpus" learned :)
