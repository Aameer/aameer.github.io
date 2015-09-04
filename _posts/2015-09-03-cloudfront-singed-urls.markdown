---
layout: post
category: articles
analytics: true
tags: [Cloudfront,S3,Debian,python,Django,Signed Urls, Secure content distribution, avoid cloudfront hotlinking]
comments: true
share: true
title: "Cloudfront Singed Urls"
date: 2015-09-03T21:33:00+05:30
---

What is Cloudfront:
-------------------
CloudFront is a web service that speeds up distribution of your static and dynamic web content, for example, .html, .css, .php,
and image files, to end users. CloudFront delivers your content through a worldwide network of data centers called edge locations.
When a user requests content that you're serving with CloudFront, the user is routed to the edge location that provides the lowest 
latency (time delay), so content is delivered with the best possible performance.
For more details check [AWS documentation](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html).

Issue with serving private content:
-----------------------------------
Now that we know what cloudfront is used for. There is an issue with caching of the our content by cloudfront servers as cloudfront servers
dont supprot referrer checks which mean anyone with the right cloudfront url can access it which inturn mounts up to more bills for you.
Also you would face the same issue if you want to serve some private content.To avoid this issue cloudfront gives a feature which is called
signed urls i.e the url generated for your content will expire after a certain time interval which you will set. This will ensure that 
user have access to your content only to a specified amount of time.

Implementation:
---------------
In this example we will use *boto 2.37, django 1.7, python 3.4, cryptography 1.0.* Note that there is an issue from *boto 2.34* onwards which this post will
address, details of the issue are [here](https://github.com/boto/boto/issues/2854).

script to generate signed urls
{% highlight python linenos %}

import boto
from boto_patch import RSADistribution as patched_distribution
CLOUD_SERVER = settings.CLOUDFRONT_SERVER_DEV
ORIGIN_ACCESS_IDENTITY='YOUR_ORIGIN_ACCESS_IDENTITY'
KEYPAIR_ID = 'YOUR_KEY_PAIR_ID'
KEYPAIR_FILE = 'PATH_TO_PRIVATE_KEY/KEYNAME.pem'
CF_DISTRIBUTION_ID ='YOUR_CLOUDFRONT_DISTRIBUTION_ID'
#you can also save these vals in the setting and import them here just as I am doing with AWS_ACCESS_KEY_ID
from django.conf import settings

def get_signed_url(full_s3_url):
    #This fucntion gets us the cloudfront signed url
    AWS_ACCESS_KEY_ID = settings.AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY = settings.AWS_SECRET_ACCESS_KEY
    my_connection = boto.cloudfront.CloudFrontConnection(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
    distros = my_connection.get_all_distributions()
    distribution_config = my_connection.get_distribution_config(CF_DISTRIBUTION_ID)
    distribution_info = my_connection.get_distribution_info(CF_DISTRIBUTION_ID)
    my_distro = patched_distribution(
        connection=my_connection,
        config=distribution_config,
        domain_name=distribution_info.domain_name,
        id=CF_DISTRIBUTION_ID,
        last_modified_time=None,
        status='Active')

    #links will expire after 2mins, ?- at GET for detials on when CloudFront checks the Expiration Date and Time in a Signed URL see aws docs here
    # http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-urls.html
    SECS = 160
    OBJECT_URL = full_s3_url
    my_signed_url = my_distro.create_signed_url(
        OBJECT_URL,
        KEYPAIR_ID,
        expire_time=int(time.time() + SECS),
        valid_after_time=None,
        ip_address=None,
        policy_url=None,
        private_key_file=KEYPAIR_FILE)
    return my_signed_url

{% endhighlight %}

Thanks to [mezka](https://github.com/mekza). we have a patch which will override the  _sign_string.
Patch taken from [here](https://github.com/boto/boto/issues/2854#issuecomment-134584217)
Save this piece of code as *boto.py* as we are importing *RSADistribution* mentioned above from here.

{% highlight python linenos %}
from boto.cloudfront.distribution import Distribution
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes
import base64

class RSADistribution(Distribution):
    def sign_rsa(self, message):
        private_key = serialization.load_pem_private_key(self.keyfile, password=None,
                            backend=default_backend())
        signer = private_key.signer(padding.PKCS1v15(), hashes.SHA1())
        message = message.encode('utf-8')
        signer.update(message)
        return signer.finalize()

    def _sign_string(self, message, private_key_file=None, private_key_string=None):
        if private_key_file:
            self.keyfile = open(private_key_file, 'rb').read()
        return self.sign_rsa(message)

    @staticmethod
    def _url_base64_encode(msg):
        """
        Base64 encodes a string using the URL-safe characters specified by
        Amazon.
        """
        msg_base64 = base64.b64encode(msg).decode('utf-8')
        msg_base64 = msg_base64.replace('+', '-')
        msg_base64 = msg_base64.replace('=', '_')
        msg_base64 = msg_base64.replace('/', '~')
        return msg_base64

{% endhighlight %}

you have to update your s3 bucket policy as user shouldn't be able to bypass cloudfront and directly access your s3 by a simple guess
Policy will check if the referer is from the s3 then deny the request, policy would look like:

{% highlight python linenos %}

{
    "Sid": "1",
    "Effect": "Deny",
    "Principal": {
        "AWS": "*"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::your_bucket_name/*",
    "Condition": {
        "StringLike": {
            "aws:Referer": [
                "https://your_bucket_name.s3.amazonaws.com/*",
                "http://your_bucket_name.s3.amazonaws.com/*"
            ]
        }
    }
}

{% endhighlight %}

When  origin access identiy updates the bucket policy it will somthing like this:
{% highlight python linenos %}

{
    "Sid": "7",
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity CLOUDFRONT_ORIGIN_ACCESS_IDENTITY_ID"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::your_bucket_name/*"
}

{% endhighlight %}


some more refrences:
--------------------
some important links which helped are: 
[stackoverflow 1](http://stackoverflow.com/questions/2573919/creating-signed-urls-for-amazon-cloudfront),
[stackoverflow 2](http://stackoverflow.com/questions/11270254/how-to-create-a-signed-cloudfront-url-with-python),
[github 3](https://github.com/boto/boto/issues/2854?_pjax=%23js-repo-pjax-container),
[cloudfront 4](http://boto.readthedocs.org/en/latest/ref/cloudfront.html).

Your checklist:
---------------
* Create cloudfront Keypair by loging into the aws console and download the private key and save the id too.
* Update the vals meantioned above accordingly.
* add origin access identity to your cloudfront distribution.
* update vals in function accordingly.
* change distribution settings(restricted viewer access,by adding self and trusted signers).
* from s3 bucket policy stop direct linking to s3.
Hope you like the post.
