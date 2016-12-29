---
layout: post
category: articles
analytics: true
tags: [Secure Adaptive Streaming, HLS, zencoder, AWS, jwplayer, Ecnryption, AES-128, Netflix, AWS Elastic Transcoder]
title: "Government/Netflix level Encryption"
comments: true
share: true
date: 2016-12-17T13:31:45+05:30
---

what and why Encryption:
------------------------
We have always been facinated by encryption, be it books like [The Da Vinci Code](https://www.goodreads.com/book/show/968.The_Da_Vinci_Code) or movies like [The Imitation Game](http://www.imdb.com/title/tt2084970/) which is about [Alan turing](https://en.wikipedia.org/wiki/Alan_Turing) breaking [Enigma](https://en.wikipedia.org/wiki/Enigma_machine). It has been around us, for decades now. In cryptography, encryption is the process of encoding messages or information in such a way that only authorized parties can access it. [read more on wiki](https://en.wikipedia.org/wiki/Encryption).

Now why is it relevant for anyone?. Matthew Green is an Assistant Research Professor of Computer Science at the Johns Hopkins University explaing that in this [Talk](https://www.youtube.com/watch?v=M6qoJNLIoJI&t). You can also check Andy Yen's building of encrypted mail program [talk](https://www.ted.com/talks/andy_yen_think_your_email_s_private_think_again). Adam Spencer gave this hilarious [talk](https://www.ted.com/talks/adam_spencer_why_i_fell_in_love_with_monster_prime_numbers) about Prime numbers which are very important in cryptography.

Today what we will try to learn is to encrypt and decrypt HlS streams with [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard). AES is a specification for encryption even used by Government intelligence organisations like [NSA](https://en.wikipedia.org/wiki/National_Security_Agency) and companies like [Netflix](https://www.netflix.com).

Some good courses about cryptography if you want to dive a little deeper:
[Stanford](https://www.coursera.org/learn/crypto), [Udacity](https://www.udacity.com/course/applied-cryptography--cs387). If you are looking for some books then [Applied Cryptography by Bruce](https://www.amazon.com/Applied-Cryptography-Protocols-Algorithms-Source/dp/0471117099/ref=sr_1_4?s=books&ie=UTF8&qid=1426511280&sr=1-4&keywords=bruce+schneier) is among the famous ones.

Now Let us write some code. We will use aes-128 encryption for encrypting our videos using services like amazon elastic transcoder and zencoder. So for this article I am assuming that you have some idea about AWS Elactic Transcoder, Zencoder, AES-128 Encryption. If you haven't worked on hls before, I would recommend to go through this [article](http://aameer.github.io/articles/hls-with-aws-and-zencoder/) where I have tried to cover it along with other relevant detials.

Encryption with AWS Elastic Transcoder:
---------------------------------------

we will be using our previous code which we used in [HLS with AWS and Zencoder](http://aameer.github.io/articles/hls-with-aws-and-zencoder/)
we will  just change a playlist function i.e the one which creates playlist which inturn is fed to transcoding job creation function

{% highlight python linenos %}

def get_playlist(project_id,video_id):
    #generate playlist
    #here wew are generating the playlist notice the final index filename has master in it and it points to all the other .m3u8 files
    playlist = [
        {
           "Format":"HLSv3",#HLSv3|HLSv4|Smooth
           "Name":str(project_id)+'/video/'+str(video_id)+'/master_'+str(project_id)+'_'+str(video_id),
           "OutputKeys":[
              '%s/video/%s/%s_360'%(project_id,video_id,video_id),
              '%s/video/%s/%s_480'%(project_id,video_id,video_id),
              '%s/video/%s/%s_720'%(project_id,video_id,video_id),
              '%s/video/%s/%s_1080'%(project_id,video_id,video_id)
           ]
        },
        "HlsContentProtection":{
           'Method':'aes-128', #This value is written into the method attribute of the EXT-X-KEY metadata tag in the output playlist.
           'KeyStoragePolicy':'WithVariantPlaylists',#Specify whether you want Elastic Transcoder to write your HLS license key to an Amazon S3 bucket.
        }
    ]
    return (playlist)

{% endhighlight %}

Few things you should keep in mind before using elastic transcoder for encryption

* Creating and associating AWS KMS Key ARN with your pipeline. Note the region for AWS KMS Key ARN and pipeline should be same.
* Use one of these input formats 3gp,asf,avi,divx,flv,mkv,mov,mp4,mpeg,mpeg-ps,mpeg-ts,mxf,ogg,ts,vob,wav,webm,mp3,m4a,aac as these are the ones supported by aws elastic transcoder
* Opting to store keys on s3 or using no playlist option in which case license acquisition url is needed. We are here going with former approach.


Encryption with Zencoder:
-------------------------
Here the only section which will change is the output

for the zencoder one we will follow the [Encoding with zencoder post](http://aameer.github.io/articles/encoding-with-zencoder/). we will 
again change some fucntions here:

{% highlight python linenos %}

def get_outputs(project_id,video_id):
    print('getting outputs')
    bucket_name = 's3://%s/'%(settings.AWS_BUCKET_NAME)
    print(bucket_name)
    outputs = [
        {
            "label": '%s_360'%(video_id),
            "audio_bitrate": 128,
            "audio_sample_rate": 44100,
            "base_url": bucket_name + '%s/video/%s/'%(project_id,video_id),
            "filename": '%s_360.m3u8'%(video_id),
            "format": "aac",
            "public": 1,
            "type": "segmented",
            "video_bitrate": 720,
            "width": 640,
            "height": 360,
            "format": "ts",
            "encryption_method":"aes-128-cbc",
            "encryption_key": None,#not compatible with rotation
            "encryption_key_rotation_period": 100,
            "encryption_key_url_prefix": None,
            "encryption_iv": None,
            "encryption_key_url":None
        },
        {
            "label": '%s_480'%(video_id),
            "audio_bitrate": 128,
            "audio_sample_rate": 44100,
            "base_url": bucket_name + '%s/video/%s/'%(project_id,video_id),
            "filename": '%s_480.m3u8'%(video_id),
            "format": "aac",
            "public": 1,
            "type": "segmented",
            "video_bitrate": 1200,
            "width": 854,
            "height": 480,
            "format": "ts",
            "encryption_method":"aes-128-cbc",
            "encryption_key": None,#not compatible with rotation
            "encryption_key_rotation_period": 100,
            "encryption_key_url_prefix": None,
            "encryption_iv": None,
            "encryption_key_url":None
        },
        {
            "label": '%s_720'%(video_id),
            "audio_bitrate": 160,
            "audio_sample_rate": 44100,
            "base_url": bucket_name + '%s/video/%s/'%(project_id,video_id),
            "filename": '%s_720.m3u8'%(video_id),
            "format": "aac",
            "public": 1,
            "type": "segmented",
            "video_bitrate": 2100,
            "width": 1280,
            "height": 720,
            "format": "ts",
            "encryption_method":"aes-128-cbc",
            "encryption_key": None,#not compatible with rotation
            "encryption_key_rotation_period": 100,
            "encryption_key_url_prefix": None,
            "encryption_iv": None,
            "encryption_key_url":None
        },
        {
            "label": '%s_1080'%(video_id),
            "audio_bitrate": 160,
            "audio_sample_rate": 44100,
            "base_url": bucket_name + '%s/video/%s/'%(project_id,video_id),
            "filename": '%s_1080.m3u8'%(video_id),
            "format": "aac",
            "public": 1,
            "type": "segmented",
            "video_bitrate": 3500,
            "width": 1920,
            "height": 1080,
            "format": "ts",
            "encryption_method":"aes-128-cbc",
            "encryption_key": None,#not compatible with rotation
            "encryption_key_rotation_period": 100,
            "encryption_key_url_prefix": None,
            "encryption_iv": None,
            "encryption_key_url":None
        },
        {
            "base_url": bucket_name + '%s/video/%s/'%(project_id,video_id),
            "filename": 'master_%s_%s.m3u8'%(project_id,video_id),
            "public": 1,
            "streams": [
                {
                    "source": '%s_1080'%(video_id),
                    "path": '%s_1080.m3u8'%(video_id)
                },
                {
                    "source": '%s_720'%(video_id),
                    "path": '%s_720.m3u8'%(video_id)
                },
                {
                    "source": '%s_480'%(video_id),
                    "path": '%s_480.m3u8'%(video_id)
                },
                {
                    "source": '%s_360'%(video_id),
                    "path": '%s_360.m3u8'%(video_id)
                }
            ],
            "type": "playlist"
        }
    ]
    return outputs

{% endhighlight %}

Things to note with respect to Zencoder

* Note if you dont mention the key,iv and url zencoder will produce them and place them in s3 bucket where it outputs the stream.

Decryption with Jwplayer:
-------------------------
What good is encryption if we cant decrypt it at the destination. For decryption the destination should have access to the keys which we have used or which aws or zencoder have produced in this case the s3 bucket. As The keys are written in manifest itself they will call for decrypting different segments whenever they are accessed. So you wont have to change much here to test our code.

Hence to stream our videos we would use [jwplayer](http://www.jwplayer.com/). To decrypt using JWPlayer you will need [enterprise plan](https://www.jwplayer.com/pricing/) as basic verison doesnt have this feature.

With access to keys and enough knowledge to decrypt the content can still be retrieved, but as decryption with keys in addition to stiching of streamns together isnt straight forward task so your content is safer than what it was before.

Thanks for your time.
