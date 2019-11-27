# Shard Downloader Improvements

## Overview

This document describes the changes being made to the `SSLHTTPDownloader`, a class that performs the task of downloading shards from remote web servers via
SSL HTTP. This class utilizes a strand (`boost::asio::io_service::strand`) to ensure that downloads are never executed concurrently. If a download is in progress and another download is initiated, the second download will be queued and invoked only when the first download is completed.

## New Features

- The ability to pause/resume downloads.
- The ability to resume downloads after a software crash.
- (Deferred) The ability to download from multiple servers to a single file.

## Implementation Details

This section describes in greater detail how the new features will be
implemented in C++ using the `boost::asio` framework.

#### Member Variables:

The variables shown here are members of the ```SSLHTTPDownloader``` class and
will be used in the following code examples.

```c++
using boost::asio::ssl::stream;
using boost::asio::ip::tcp::socket;

stream<socket>>         stream_;
std::condition_variable pauseResume_;
std::atomic<bool>       isPaused_;
```

### Pausing Downloads

##### Thread 1:

A pause is initiated by setting an atomic member variable.

```c++
void SSLHTTPDownloader::pauseDownloads()
{
    isPaused_ = true;
}
```

##### Thread 2:

The pause is realized when the thread executing the download polls `isPaused_`  after this variable has been set to `true`. Polling only occurs while the file is being downloaded, in between calls to `async_read_some()`. The pause takes effect when the socket is closed and the thread sleeps on a condition variable. Any queued downloads will be postponed, as each download is invoked on the same `strand`.

```c++
void SSLHTTPDownloader::do_session()
{
    while (true)
    {
         // (Connection initialization logic)

         .
         .
         .

         // (In between calls to async_read_some):

         if (isPaused_.load ())
         {
             stream_->async_shutdown();
             pauseResume_.wait ();

             // After waking up, use "continue"
             // to start at the beginning of this
             // method. This will reconnect to the
             // server and resume the download.
             continue;
         }

         .
         .
         .

         break;
    }
}
```

### Resuming Downloads

##### Thread 1:

Resuming downloads is initiated by updating `isPaused_`, and waking any thread currently waiting on the condition variable `pauseResume_`. It's possible that no thread is waiting on `pauseResume_`, which is not an issue.

```c++
void SSLHTTPDownloader::resumeDownloads()
{
    if (!isPaused_.load ())
        return;

    isPaused_ = false;
    pauseResume_.notify_one ();
}
```

##### Thread 2:

In order to resume the download, the notified thread restarts the current routine. Rather than recursively invoking the current method, which could be abused to cause stack overflow, the routine is wrapped in a `while` loop, and the thread reconnects to the server by using `continue` to return to the beginning of the method.

```c++
void SSLHTTPDownloader::do_session()
{
    while (true)
    {
         // (Connection initialization logic)

         .
         .
         .

         if (downloadInterrupted)
         {
             // Request remainder of file using the
             // HTTP range header.
         }

         // (In between calls to async_read_some):

         if (isPaused_.load ())
         {
             stream_->async_shutdown();
             pauseResume_.wait ();

             // After waking up, use "continue"
             // to start at the beginning of this
             // method. This will reconnect to the
             // server and resume the download.
             continue;
         }

         .
         .
         .

         break;
    }
}
```

##### Note:
To resume downloading after a pause, as an alternative to using flow control, we could also refactor the connection and download initialization logic into a separate method that gets invoked at the beginning of `do_session()` and after waking.

### Recovery
Although `SSLHTTPDownloader` is a generic class that could be used to download a variety of file types, currently it is used exclusively by the `ShardArchiveHandler` to download shards. In order to provide resilience, the `ShardArchiveHandler` will utilize the filesystem to preserve its current state whenever there are active, paused, or queued downloads. A new section in the configuration file will be provided allowing users to specify the location of the file to use for this purpose.

##### Config File Entry
The `path` field of the `shard_db` entry will be used to determine where to store the download recovery file.

```bash
# This is the persistent datastore for shards. It is important for the health
# of the ripple network that rippled operators shard as much as practical.
# NuDB requires SSD storage. Helpful information can be found here
# https://ripple.com/build/history-sharding
[shard_db]
type=NuDB
path=/var/lib/rippled/db/shards/nudb
max_size_gb=50
```
