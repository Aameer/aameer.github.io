---
layout: post
title: "Secure, fast uploads for large files directly from browser to s3"
date: 2018-02-18T14:21:42+05:30
category: articles
analytics: true
tags: [AWS, Security, Multi-part Upload, S3, Cloudfront, cdn, Performance, Large file uploads (> 5GB), STS, Temporary Tokens, AWS JS SDK]
comments: true
share: true
date: 2018-02-18T20:43:41+05:30
---

# Problem
In this post we will be discussing "How to acheive high speed uploads securly". In all the cases mentioned below, we will be using multipart upload for best performance. Some points for context before we dive deeper:

1. Most of the developers are familiar with using AWS (Amazon Web Services) for uploads, you may have used aws js sdk or boto(python) or any other library. One of the most usual tasks which one would perform is uploads. For most of the use cases uploading files to a sever first and then uploading them from server to s3 will do the job. In cases where file size is not small or in cases where you want to improve performace further this approach may not be suitable.

2. Here in the second category, where your uploads can be upto 5GB. What you probably would end up doing is uploading files directly from frontend to S3 (to improve performance by saving some time). In this case security becomes of huge importance as some hacker could get the keys from your request and hence get access to your data. Using Signed urls will resolve this issue. Here you would generate signed urls on your backend and send them across to frontend to upload files.

3. Things become interesting when you want to upload large files (say 20GB). AWS has limit of 5GB on requests with signed urls. Hence you will need to figure something else. Even if you look at third party uploading services you still may face these issues:

* Either they are not fast enough.
* Most of them have support for just 5GB files sizes.
* They are costly (implementation time or money wise).

Primarily we are going to address 3rd case in this post (although it can be applied to all). As always we will start from github. So, there is a [nice discussion on github which is our starting point](https://github.com/aws/aws-sdk-js/issues/468). There are some really smart solutions mentioned here with some examples in ruby etc. I tried many approaces which were inspired from this discussion but none of them seemed to solve for everything we were looking for and even if they did we couldn't make them work seamlessly. So, we tried another solution which was suggested by our friends at Amazon. Big shout out for them especially Vinay. The approach is "creating temporary credentials with limited access to resource on backend, sending it to frontend; then use it with multipart uploads and vola! you have a working solution". We were also exploring use of cloudfront (aws CDN) for uploads which lead us to this insight (our cherry on top of the cake) that if you use accelerated uploads, it inturn uses cloudfront (aws cdn) to improve transfer speed. In case of aws js sdk using `useAccelerateEndpoint: true` and enabling it on your s3 bucket properties is all you have to do to enable it. You may be charged extra for this though.

The approximate upload speeds which we got after this implementation (even on little slower networks) was about 1GB/min.  Now let us take a look at some code :

Python (BACKEND)
----------------
{% highlight python linenos %}
import boto3
REGION_NAME='ap-southeast-1' #e.g

sts = boto3.client(
    'sts',
    aws_access_key_id=<AWS_ACCESS_KEY_ID>,
    aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>,
    region_name=REGION_NAME
)

success_sts, sts = get_sts()
Policy={"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:*","Resource":["arn:aws:s3:::<your-bucket>/<key-folder>/*"]}]}
credentials = sts.get_federation_token(
    Name=user_email, #or any unique text related to user
    Policy=policy,
    DurationSeconds=<AWS_TEMP_CREDENTIALS_EXPIRY_TIME>, 
    #minimum is 200 which is enough as token is just for start of request and necessarily need not live throughout the life of the whole upload.
)
access_key_id = credentials['Credentials']['AccessKeyId']
session_token = credentials['Credentials']['SessionToken']
secret_access_key = credentials['Credentials']['SecretAccessKey']
{% endhighlight %}

JS (FRONTEND)
-------------
{% highlight javascript linenos %}

data =get_data() //get credentials from http call to backend(e,g using ajax), say we save it in variable data

AWS.config.update({
    accessKeyId: data.access_key_id,
    secretAccessKey: data.secret_access_key,
    sessionToken: data.session_token,
    region: data.region,
    useAccelerateEndpoint: true, //for using accelerated transfers
});

var s3 = new AWS.S3();

return new Promise((resolve, reject) => {
    this.awsReq = s3.upload(params, (err, data) => {
        if (err) {
            if (err.code === 'RequestAbortedError') {
                console.log("upload aborted")
            } else {
                console.log("upload failed")
            }
            return reject(err);
        } else {
            console.log('upload successful')
        }
    }).on('httpUploadProgress', (evt) => {
        console.log(Math.floor(evt.loaded * 100 / evt.total));
    });
});

{% endhighlight %}

# References
[multipart via cloudfront](https://stackoverflow.com/questions/31687165/s3-multipart-upload-via-cloudfront-distribution-with-aws-sdk-js).  
[gist mulitpart upload](https://gist.github.com/xli/6f313d06d4b9e70bf3b0#file-s3-multipart-upload-example-step4-js).  
[aws sdk issue](https://github.com/aws/aws-sdk-js/issues/423).  
[aws sdk issue](https://github.com/aws/aws-sdk-js/issues/468).  
[aws sdk issue](https://github.com/aws/aws-sdk-js/issues/669).
[boto sts](http://boto.cloudhackers.com/en/latest/ref/sts.html).  
[boto sts federation token](http://boto.cloudhackers.com/en/latest/ref/sts.html#boto.sts.STSConnection.get_federation_token).  
[sample example](https://github.com/vinay-nadig-0042/client-side-multpart-upload-demo/tree/master).
