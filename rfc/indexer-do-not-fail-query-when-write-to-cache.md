# RFC: Do not fail query when writing to cache

When querying the indexer it may have to fetch data from IPNI because it is not found in local caches. It is then written to local caches to allow subsequent requests to be served faster. Under certain conditions writing to the cache may fail. This could be because of varous reasons - e.g. the network connection is down, OOM due to the cache being full of non-expirable items or downtime due to system upgrades.

In the case when writing to the cache fails, the proposal is to return the fetched data to the client instead of raising an error. At this point, the work to fetch from IPNI has already been done, so allowing the client to proceed instead of failing (and possibly trying again) might actually alleviate pressure on the issue and help ensure service levels are maintained (even if operating in a degraded state).

The following additional tasks are also recommended:

* Ensure the error is logged, so that the issue can be debugged.
* Set up monitoring on caches to ensure the error is not masked and/or report the error to a monitoring service e.g. Sentry.
