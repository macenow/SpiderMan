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

A `ParserThread` will pick a task from `p_task_queue`, parse the html content for acquiring needed data or more urls, and send content (e.g. data, page header, more urls) to `f_task_queue` for further fetching process, or to `s_task_queue` if some conditions matched (e.g., max depth reached). Every time when a task is going to be added to `f_task_queue`, `FilterThread` (if the instance is not None) will always check for redundancy

A `SaverThread` will pick a task from `s_task_queue`, save the data or content to defined method, e.g., a database or to a Json file. Before saving the data

### Make a quick run

```python
from spider_man.workers import Fetcher, Parser, Saver, Filter
from spider_man.thread_pool import ThreadPool

if __name__ == "__main__":
    url = "https://www.jetbrains.com/help/pycharm/meet-pycharm.html"
    fetcher = Fetcher()
    parser = Parser(max_deep=2)
    saver = Saver(pipe='test_result')  # save results to file "test_result"
    filter = Filter(bloom_capacity=1000)

    spider = ThreadPool(fetcher, parser, saver, filter, fetcher_num=10)
    spider.run(url, None, priority=0, deep=0)
```

### Customization

You can inherit the 3 workers (class `Fetcher`/`Parser`/`Saver`) and override their corresponding methods (`fetch()`/`parse()`/`save()`) to provide a customized crawler and get the data you want.

for example:

```python
class MyFetcher(Fetcher):
    def __init__(self, url: str, jison: Jison = None, db=None,
                 max_repeat: int = 3, sleep_time: int = 0):
        super().__init__(max_repeat, sleep_time)
        self.jison = jison
        self.url = url.lower()
        self.db = db

    def change_db(self, db):
        self.db = db

    def fetch(self, url: str, data: dict, session):
        if data.get('save'):
            return 1, data, (200, '', '')

        if data.get('type') == 'field_page' or data.get('type') == 'group_page':
            response = session.get(url, timeout=(3.05, 10))
            return 1, data, (response.status_code, response.url, response.text)

        elif data.get('type') == 'var_page':
            var_type = data.get('var_type')
            if var_type == 'Multi' or var_type == 'Choice':
                session.get(url)
                response = session.get(f'https://{self.url}.', timeout=(3.05, 10))
                return 1, data, (response.status_code, response.url, response.text)
            else:
                return 1, data, (200, '', '')

        else:
            raise Exception(f'Invalid data type: type={data.get("type")}')
```
