---
title: Concurrent access to Models
---

Applications need to be aware of the concurrency issues in access
Jena models. API operations are not thread safe by default. Thread
safety would simple ensure that the model data-structures remained
intact but would not give an application consistent access to the
RDF graph. It would also limit the throughput of multi-threaded
applications on multiprocessor machines where true concurrency can
lead to a reduction in response time.

For example, supposed an application wishes to read the name and
age of a person from model. This takes two API calls. It is more
convenient to be able to read that information in a consistent
fashion, knowing that the access to the second piece of information
is not being done after some model change has occurred.

Special care is needed with iterators. In general, Jena's
iterators do *not* take a copy to enable safe use in the presence
of concurrent update. A multi-threaded application needs to be
aware of these issues and correctly use the mechanisms that Jena
provides (or manage its own concurrency itself). While not zero,
the application burden is not high.

There are two main cases:

-   Multiple threads in the same JVM.
-   Multiple applications accessing the same persistent model
    (typically, a database).

Transactions are provided by persistent models: see
the [TDB documentation](/documentation/tdb/tdb_transactions.html)
and the [SDB documentation](/documentation/sdb/index.html)
and for details.

This note describes the support for same-JVM, multi-threaded
applications.

## Locks

Locks provide critical section support for managing the
interactions of multiple threads in the same JVM. Jena provides
multiple-reader/single-writer concurrency support (MRSW).

The pattern general is:

    Model model = . . . ;
    model.enterCriticalSection(Lock.READ) ;  // or Lock.WRITE
    try {
        ... perform actions on the model ...
        ... obey contract - no update operations if a read lock
    } finally {
        model.leaveCriticalSection() ;
    }

Applications are expected to obey the lock contract, that is, they
must not do update operations if they have a read lock as there can
be other application threads reading the model concurrently.

## Iterators

Care must be taken with iterators: unless otherwise stated, all
iterators must be assumed to be iterating over the data-structures
in the model or graph implementation itself.  It is not possible to
safely pass these out of critical sections.

## SPARQL Query

SPARQL query results are iterators and no different from other
iterators in Jena for concurrency purposes. The default query
engine does not give thread safety and the normal requirements on
an application to ensure MRSW access in the presence of iterators
applies. Note that Jena's query mechanism is itself multi-threaded.
If the application is single threaded, no extra work is necessary.
If the application is multi-threaded, queries should be executed
with a read lock.

Outline:

      Model model = ... ;
      String queryString = " .... " ;
      Query query = QueryFactory.create(queryString) ;
      model.enterCriticalSection(Lock.READ) ;
      try {
        try(QueryExecution qexec = QueryExecutionFactory.create(query, model)) {
            ResultSet results = qexec.execSelect() ;
            for ( ; results.hasNext() ; )
            {
                QuerySolution soln = results.nextSolution() ;
                RDFNode x = soln.get("..var name..") ;
            }
        }
      } finally { model.leaveCriticalSection() ; }

Updates to the model should not be performed inside the read-only
section. For database-backed models, the application can use a
transaction. For in-memory models, the application should collect
the changes together during the query processing then making all
the changes holding a write lock.

Jena Locks do not provide lock promotion - an application can not
start a "write" critical section while holding a "read" lock
because this can lead to deadlock.

## Compatibility

The actually interface is called `Lock` and has implementations
including `LockMRSW`.

For compatibility with previous versions of Jena, there is a class
`ModelLock`.
