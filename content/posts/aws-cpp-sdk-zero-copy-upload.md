+++
title = "How to perform zero-copy S3 uploads with the AWS C++ SDK"
date = "2021-12-11"
author = "Roman Böhringer"
authorTwitter = "romanboehr"
cover = ""
tags = ["cpp"]
keywords = ["aws", "s3"]
description = ""
showFullContent = false
+++

If you're ever using the AWS C++ SDK in some constrained environment (such as Lambda functions with limited memory) or care about memory copies, you probably run into the issue of how to upload an existing buffer without copying it ([as](https://github.com/aws/aws-sdk-cpp/issues/64) [other](https://github.com/aws/aws-sdk-cpp/issues/533) [developers](https://github.com/aws/aws-sdk-cpp/issues/785) [did](https://github.com/aws/aws-sdk-cpp/issues/1430)).
You can write your own `streambuf` wrapper to do so, but if you're already using `boost` in your project, `boost::interprocess::bufferstream` is a very straightforward way to do it:
```cpp
char* buf;
std::size_t len;
const std::shared_ptr<Aws::IOStream> data = Aws::MakeShared<boost::interprocess::bufferstream>(TAG, buf, buf_len);
```
`data` can then be used as usual:
```cpp
Aws::S3::Model::PutObjectRequest request;
request.WithBucket(bucket_name).WithKey(name);
request.SetBody(data);
auto outcome = client->PutObject(request);
```
