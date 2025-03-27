+++
author = "Andreas Lay"
title = "Distributed locking using Google Cloud Storage (or S3)"
date = "2025-03-27"
description = "A simple distributed locking mechanism"
tags = ["gcp", "Python"]
categories = ["Python", "Platform"]
ShowToc = true
TocOpen = true
+++

## The Need for Distributed Locks

When you run into situations where you want to prevent two pieces of code ‒ possibly running on differen machines ‒ from running concurrently, you need a distributed lock. An easy solution to implement such a lock is to leverage a cloud storage service like Google Cloud Storage (GCS) or Amazon S3. [Here you can find the full code example implementing this kind of lock.](https://gist.github.com/layandreas/7e2fb6a44c17bb61ff45107c856cfb0f)

A concrete example would be to prevent multiple [dbt incremental models](https://docs.getdbt.com/docs/build/incremental-models#configure-incremental-materializations) from running in parallel as this could result in adding duplicate rows.

## Google Cloud Storage: Conditional Operations

If you are using Google Cloud Storage you can simple create a lock by uploading a file to your storage bucket
and use [if_generation_match=0](https://cloud.google.com/python/docs/reference/storage/latest/generation_metageneration#using-ifgenerationmatch):

> _As a special case, passing 0 as the value for if_generation_match makes the operation succeed only if there are no live versions of the blob._

This means we can try uploading a file to storage while setting `if_generation_match=0`: If the file does not yet exist the upload will succeed and our process will hold the lock. Any other process attempting to upload a file with the same name ‒ i.e. trying to acquire the same lock ‒ will fail to do so.

Using the GCS Python client library it looks like this:

```python
from google.cloud import storage

bucket_name = "my-storage-bucket"
client = storage.Client()
bucket = client.bucket(bucket_name)
lock_blob = bucket.blob(lock_name)
# This will fail if the file already exists as we set
# `if_generation_match=0`
lock_blob.upload_from_string("lock", if_generation_match=0)
```

This approach is suitable for applications where latency when acquiring / releasing the lock is not of concern.

## Writing a RemoteLock Class

Now that we've learnt about `if_generation_match=0` let's write a simple interface to use this mechanism as lock. We'd like to use the lock as a context manager:

```python
with RemoteLock(bucket_name="my_bucket", lock_name="my_lock") as lock:
     # This code path can only run once at any given time, no parallel
     # executions are possible
     ...

```

We need to implement:

- The locking mechanism itself:
  - Acquiring the lock (=conditionally writing the file)
  - Releasing the lock (=deleting the file)
- A retry mechanism in case locking fails
- A timeout after which the program stops trying to acquire the lock

Here's a Python class implementing all of the above:

```python
import logging
import time
from typing import Any

from google.cloud import storage

logger = logging.getLogger(__name__)


class RemoteLock:
    """
    A class to handle distributed locking using Google Cloud Storage. While
    the lock is acquired by the calling process other processes will not be
    able to acquire the lock and will wait until they acquire it or until
    the timeout is reached.
    Parameters
    ----------
    bucket_name : str
        The name of the cloud storage bucket.
    lock_name : str
        The name of the lock file.
    lock_timeout_seconds : int, optional
        The timeout in seconds to wait for acquiring the lock (default is 60).
    Methods
    -------
    __enter__():
        Enters the runtime context related to this object.
    __exit__(exc_type, exc_val, exc_tb):
        Exits the runtime context related to this object.
    lock() -> bool:
        Attempts to acquire the lock.
    unlock() -> None:
        Releases the lock.
    wait_for_lock(timeout: int) -> None:
        Waits until the lock is acquired or the timeout is reached.
    Usage
    -----
    >>> with RemoteLock(bucket_name="my_bucket", lock_name="my_lock") as lock:
    >>>     # Critical section of code
    >>>     pass
    """

    def __init__(
        self, bucket_name: str, lock_name: str, lock_timeout_seconds: int = 60
    ) -> None:
        self.bucket_name = bucket_name
        self.lock_name = lock_name
        self.lock_timeout_seconds = lock_timeout_seconds
        self.client = storage.Client()
        self.bucket = self.client.bucket(bucket_name)
        self.lock_blob = self.bucket.blob(lock_name)

    def __enter__(self) -> "RemoteLock":
        self.wait_for_lock(self.lock_timeout_seconds)
        return self

    def __exit__(
        self,
        exc_type: BaseException | None,
        exc_val: BaseException | None,
        exc_tb: Any,
    ) -> None:
        self.unlock()

    def lock(self) -> bool:
        logger.debug("Acquiring lock: {}".format(self.lock_name))
        try:
            # Using if_generation_match=0 allows us to use cloud storage as a
            # locking mechanism as the upload will fail if the blob already
            # exists. We also don't have write race conditions if another
            # process simultaneously tries to acquire the lock
            # (upload the file).
            self.lock_blob.upload_from_string("lock", if_generation_match=0)
            logger.info(f"Lock successfully acquired: {self.lock_name}")
            return True
        except Exception as e:
            logger.debug(f"Cannot acquire lock {self.lock_name}: {e}")
            return False

    def unlock(self) -> None:
        logger.info(f"Releasing lock: {self.lock_name}")
        self.lock_blob.delete()
        logger.info(f"Lock successfully released: {self.lock_name}")

    def wait_for_lock(self, timeout: int) -> None:
        start_time = time.time()
        logger.info(f"Waiting for lock: {self.lock_name}")
        while not self.lock():
            if time.time() - start_time > timeout:
                raise TimeoutError(
                    f"Could not acquire lock {self.lock_name} on bucket "
                    f"{self.bucket_name} within the specified timeout. "
                    "This means another process is holding the lock. "
                    "If you think the other process failed to release the lock "
                    "you can manually release it by deleting the lock file "
                    f"on the storage bucket {self.bucket_name}."
                )
            time.sleep(5)
```

[Link to gist](https://gist.github.com/layandreas/7e2fb6a44c17bb61ff45107c856cfb0f)

## Other Storage Backends

Since August 2024 [Amazon S3 supports conditional writes](https://aws.amazon.com/about-aws/whats-new/2024/08/amazon-s3-conditional-writes/), so you could replace the GCS backend with S3 in the example above to get the same locking mechanism.
