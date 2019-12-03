# Shard Downloader Improvements

## Overview

This document describes the changes being made to the `SSLHTTPDownloader`, a class that performs the task of downloading shards from remote web servers via
SSL HTTP. The downloader utilizes a strand (`boost::asio::io_service::strand`) to ensure that downloads are never executed concurrently. Hence, if a download is in progress when another download is initiated, the second download will be queued and invoked only when the first download is completed.

## New Features

- The ability to pause/resume downloads.
- The ability to resume downloads after a software crash.
- <span style="color:gray">*(Deferred) The ability to download from multiple servers to a single file.*</span>

## Classes

Implementing the shard downloader improvements will require making changes to the following classes:

- `SSLHTTPDownloader`

 This is a generic class designed for serially executing downloads via HTTP SSL.

- `ShardArchiveHandler`

 This class uses the `SSLHTTPDownloader` to fetch shards from remote web servers. Additionally, the archive handler performs sanity checks on the downloaded files and imports the validated files into the local shard store.

##### ShardArchiveHandler
The `ShardArchiveHandler` exposes a simple public interface:

```C++
/** Add an archive to be downloaded and imported.
    @param shardIndex the index of the shard to be imported.
    @param url the location of the archive.
    @return `true` if successfully added.
    @note Returns false if called while downloading.
*/
bool
add(std::uint32_t shardIndex, parsedURL&& url);

/** Starts downloading and importing archives. */
bool
start();
```

When a client submits a `download_shard` command via the RPC interface, each of the requested files is registered with the handler via the `add` method. After all the files have been registered, the handler's `start` method is invoked, which in turn creates an instance of the `SSLHTTPDownloader` and begins the first download. When the download is completed, the downloader invokes the handler's `complete` method, which will initiate the download of the next file, or simply return if there are no more downloads to process. When `complete` is invoked with no remaining files to be downloaded, the handler and downloader are destroyed automatically.

## Execution Concept

This section describes in greater detail how the new features will be
implemented in C++ using the `boost::asio` framework.

##### Member Variables:

The variables shown here are members of the `SSLHTTPDownloader` class and
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
To resume downloading after a pause, as an alternative to using flow control, we could also refactor the connection and download initialization logic into a separate method that gets invoked at the beginning of `do_session()` and after waking. This is the preferred approach.

### Recovery
Although `SSLHTTPDownloader` is a generic class that could be used to download a variety of file types, currently it is used exclusively by the `ShardArchiveHandler` to download shards. In order to provide resilience, the `ShardArchiveHandler` will utilize the filesystem to preserve its current state whenever there are active, paused, or queued downloads. The `shard_db` section in the configuration file allows users to specify the location of the file to use for this purpose.

##### Config File Entry
The `path` field of the `shard_db` entry will be used to determine where to store the download recovery file. In addition, a new field named `download_path` will be provided to allow users to provide a separate path for storing downloads and the state file.

```dosini
# This is the persistent datastore for shards. It is important for the health
# of the ripple network that rippled operators shard as much as practical.
# NuDB requires SSD storage. Helpful information can be found here
# https://ripple.com/build/history-sharding
[shard_db]
type=NuDB
path=/var/lib/rippled/db/shards/nudb
max_size_gb=50
```

#### Resuming Partial Downloads
When resuming downloads after a crash or other interruption, the `SSLHTTPDownloader` will utilize the `range` field of the HTTP header to download only the remainder of the partially downloaded file.

```C++
auto downloaded = getPartialFileSize();
auto total = getTotalFileSize();

http::request<http::empty_body> req {http::verb::head,
  target,
  version};

if (downloaded < total)
{
  // If we already download 1000 bytes to the partial file,
  // the range header will look like:
  // Range: "bytes=1000-"
  req.set(http::field::range, "bytes=" + to_string(downloaded) + "-");
}
else if(downloaded == total)
{
  // Download is already complete. (Interruption Must
  // have occurred after file was downloaded but before
  // the state file was updated.)
}
else
{
  // The size of the partially downloaded file exceeds
  // the total download size. Error condition. Handle
  // appropriately.
}
```

## Sequence Diagram

This sequence diagram demonstrates a scenario wherein the `ShardArchiveHandler` leverages the state persisted on the filesystem to recover from a crash and resume the scheduled downloads.

![alt_text](./interrupt_sequence.png "image_tooltip")
