---
date: '2025-05-26T10:00:32Z'
title: 'S3 Object Storage for MetaCentrum Workflows'
thumbnail: '/img/s3-fr/big_data_gpt.png'
description: "Storing and Handling Big Data"
tags: ["Jan Pospíšil, Vladimíra Kimlová, Dominik Zappe", "User Experience", "S3", "FR CESNET", "ZCU"]
colormode: true
draft: false
---

# Storing and Handling Big Data in S3 Object Storage for MetaCentrum Workflows

> 💡 **Note** 
> This blog post is brought to you directly by our users as part of our ongoing effort to share knowledge and real-world experience across the community. 
> The content you’re about to read is the result of a project supported through the FR CESNET (Development Fund).
> Final report (in Czech) is available at http://hdl.handle.net/11025/61737


Big data is only as useful as your ability to access and manage it.
When you're working with terabytes of information on MetaCentrum, object storage like S3 becomes essential.
Let’s dive into how to handle, store, and streamline your data pipelines with S3.

## Available Tools for Big Data in S3

For handling big data in S3, we recommend the following tools:
- **boto3**: A Python library for interacting with S3, ideal for scripting and automation.
- **s5cmd**: A high-performance command-line tool for S3, designed for large data transfers and operations.

## Best Practices for Big Data Transfers

### Fewer Big Files Over Many Small Files

When transferring large datasets, it is more efficient to use fewer big files rather than many small ones.
This reduces overhead and speeds up the transfer process.

When your data consists of many small files, consider packing them into a single large file before transfer.

### Optimal Chunk Size

When transferring large files, the upload or download process is divided into chunks, known as **multipart** uploads or downloads.
The size of these chunks can significantly impact transfer speed.

The optimal chunk size depends on the size of the files you are transferring and the network conditions.
There is no one-size-fits-all solution, but a good starting point is to set the chunk size to `file_size / 1000`.

### Cluster Choice

Some clusters offer better network interfaces than others.
When transferring large files, it is important to choose a cluster with a good network interface.

You can check the possible clusters and their network interfaces on the [official website](https://metavo.metacentrum.cz/pbsmon2/nodes/physical) of the MetaCentrum.

### Hard Disk Speed

Our research has shown that the speed of the hard disk does not have a significant impact on transfer speed.
When transferring large files, the network interface is usually the bottleneck, not the hard disk speed.

### Utilize Compression

You can compress files before transferring them to reduce the time and resources needed for the transfer.
The choice of compression algorithm depends on the type of files you are transferring, but we recommend using the `zstandard` algorithm for its balance between compression ratio and decompression speed.

For more information about compression algorithms, you can check this [comparison](https://quixdb.github.io/squash-benchmark/).

## Storing and Accessing Big Data in S3

When storing and accessing big data in S3, it is important to use the right tools and techniques.

### Using boto3

`boto3` is a Python library that allows you to interact with S3 storage.
You can use it in your Python scripts to automate data transfers and manage your S3 buckets.

Example scripts to upload and download files using `boto3` can be found below:

```python
import boto3
from boto3.s3.transfer import TransferConfig

# The following credentials and endpoint are examples, replace them with your own or get them from a secure source
access_key = "ABCD****************"
secret_key = "wxyz************************************"
endpoint_url = "https://s3.clX.du.cesnet.cz"
region_name = "us-east-1"

mb = 1024 ** 2

# Adjust the multipart threshold and chunk size as needed
config = TransferConfig(multipart_threshold=50 * mb, multipart_chunksize=50 * mb, use_threads=True)

s3 = boto3.client(
    's3',
    aws_access_key_id=access_key,
    aws_secret_access_key=secret_key,
    endpoint_url=endpoint_url,
    region_name=region_name
)

# Assuming you have a bucket and a file to upload/download
bucket_name = "bucket-name"
local_path = "/path/to/local/file"
s3_path = "desired/s3/path/to/file"

# Upload the file to S3
s3.upload_file(local_path, bucket_name, s3_path, Config=config)
# Download the file from S3
s3.download_file(bucket_name, s3_path, local_path, Config=config)
```

### Using s5cmd

`s5cmd` is a high-performance command-line tool for S3, designed for large data transfers and operations.

For example installation instructions and usage see the [official repository](https://github.com/peak/s5cmd) of the tool.

## MetaCentrum and S3 Integration

Integrating S3 with your MetaCentrum workflows is as easy as using the above scripts and tools right in your MetaCentrum job files.
You can stage in and out your data using `boto3` or `s5cmd` commands directly in your job scripts.

## Conclusion

Handling big data in S3 object storage is crucial for efficient workflows in MetaCentrum.
By following best practices for data transfer, utilizing the right tools, and understanding how to manage your data effectively, you can streamline your processes and make the most of your big data sets.
