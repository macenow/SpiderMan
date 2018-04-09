## SpiderMan

SpiderMan is a micro multi-threading crawler framework. The basic structure of it is like this:

```
#################### class ThreadPool spawns threads ####################
           |                |                 |                |
           |                |                 |                |
           |                |                 |                |
     FetcherThread     ParserThread      SaverThread     FilterThread
    e.g. 20 threads   e.g. 5 threads    e.g. 1 thread    e.g. 1 thread
           |                |                 |                |
      f_task_queue     p_task_queue      s_task_queue          |
           |                |                 |                |
           |                |                 |                |
           |                |                 |                |
        Fetcher           Parser            Saver            Filter
#########################################################################
```

Thread ThreadPool class can spawn many threads, each thread (class `FetcherThread`/`ParserThread`/`SaverThread`/`FilterThread`) will has one corresponding worker (class `Fetcher`/`Parser`/`Saver`/`Filter`) for dealing with tasks.

Once the system starts, the instance of ThreadPool will also initiate three queues:
- `f_task_queue` for `FetcherThread`
- `p_task_queue` for `ParserThread`
- `s_task_queue` for `SaverThread`

A `FetcherThread` will pick a task from `f_task_queue`, fetch the html content and send content (e.g., page url and page html) to `p_task_queue`.

A `ParserThread` will pick a task from `p_task_queue`, parse the html content for acquiring needed data or more urls, and send content (e.g. data, page header, more urls) to `f_task_queue` for further fetching process, or to `s_task_queue` if some conditions matched (e.g., max depth reached)

A `SaverThread` will pick a task from `s_task_queue`, save the data or content to defined method, e.g., a database or to a Json file. Before saving the data, `SaverThread` will also turn to `FilterThread` (if the instance is not None) for eliminating redundancy.
