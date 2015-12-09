---
layout: post
category: articles
analytics: true
tags: [Secure Adaptive Streaming, HLS, zencoder, AWS, jwplayer, HLS vs Dash]
comments: true
share: true
title: "HLS with AWS and Zencoder"
date: 2015-12-06T17:29:02+05:30
---

what and why secure adaptive streaming:
---------------------------------------
HLS i.e HTTP Live Streaming is an HTTP-based media streaming communications protocol implemented by Apple Inc. It has many advantages details 
[here](https://en.wikipedia.org/wiki/HTTP_Live_Streaming).More prominent advantages are:

* More secure than progressive download method
* Uses the network more efficiently

why HLS:
-------
Today we will be discussing HLS but there are other protocols which we can use like [MPEG-DASH](https://en.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP)
this line summarizes 'HLS for now and DASH for future' . Once Apple starts backing up DASH which is the future for secure streaming then we will see more support 
for dash and community will adopt it faster. AWS supports HLS for now but not DASH, zencoder has support for bothi.
some intresting reads on this issue:

* [jwplayer](http://www.jwplayer.com/blog/the-future-of-mobile-video-is-apple-for-now/)
* [streaminglearningcenter](http://www.streaminglearningcenter.com/blogs/dash-vs-hls-request-for-comments.html)
* [bitcodin - alternative to jwplayer](https://www.bitcodin.com/blog/2015/03/mpeg-dash-vs-apple-hls-vs-microsoft-smooth-streaming-vs-adobe-hds/)
* [our buddy stackoverflow](http://stackoverflow.com/questions/15687434/what-is-the-difference-between-hls-and-mpeg-dash)

HLS with aws and zencoder:
-------------------------
we will be using our previous code which we used in [AWS elastic encoder post](http://aameer.github.io/articles/aws-elastic-encoder/)
we will change only few functions:

fist we would need to add SegmentDuration to the inputs and make sure the segment duration is same for all the inputs as for playlist
which we will discuss later on it needs to be the same.

{% highlight python linenos %}

def get_output_objects(project_id,video_id):
    #decided to shift to width based names
    output_objects = [
        {
            'Key': '%s/video/%s/%s_360'%(project_id,video_id,video_id),
            'PresetId':'1448537835613-80001', #preset id
            'SegmentDuration':'10', #duration of .ts segments
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        },
        {
            'Key': '%s/video/%s/%s_480'%(project_id,video_id,video_id),
            'PresetId':'1448538214998-80002',
            'SegmentDuration':'10',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        },
        {
            'Key': '%s/video/%s/%s_720'%(project_id,video_id,video_id),
            'PresetId':'1448538068019-cbchx',
            'SegmentDuration':'10',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        },
        {
            'Key': '%s/video/%s/%s_1080'%(project_id,video_id,video_id),
            'PresetId': '1448538516638-700x1',
            'SegmentDuration':'10',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        }
    ]
    return (output_objects)

{% endhighlight %}

now the playlist will have all the names for the inputs all of them would be .m3u8 files which are like an idex to all the segment files 
with .ts extension

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
        }
    ]
    print('pl',playlist)
    return (playlist)


def create_transcoding_job(location,project_name,video,elastic_transcoder):
    # we are saving in the same bucket here, you can also save in another bucket
    ext = location.split('.')[-1]
    orig_key = location.split('_')[-1].split('.')[0]
    input_object = get_input_object(project_name,video.id,orig_key,ext)
    output_objects=get_output_objects(project_name,video.id)
    output_playlist = get_playlist(project_id,video.id)
    response=elastic_transcoder.create_job(pipeline_id=PIPELINE_ID,input_name=input_object,outputs=output_objects,playlists=output_playlist)
    aws_job_id =response['Job']['Id']
    print ("new created job id = %s" % aws_job_id)
    print('\n')
    video.awsencoder_state=aws_job_id
    video.save()

{% endhighlight %}

HLS with zencoder:
------------------


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
            "format": "ts"
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
            "format": "ts"
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
            "format": "ts"
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
            "format": "ts"
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

def main():
    print('timestamp[d/m/y-h:m:s]=',datetime.now().strftime("%d/%m/%Y-%H:%M:%S"),'\n')

    print('Video encoding started')
    if os.path.exists('video_encoding.lock'):
        print('video lock is present, This may be beacuse script is running already or encountered an error when it was run last time')
        sys.exit()
    else:
        open('video_encoding.lock', 'w').write('running')
    client = get_zencoder_client()
    bucket = get_bucket()
    all_videos=[] 
    for v_id in VIDEO_LIST:
        #v = Videos.objects.get(id=v_id)
        v  =  get_video_objetct(v_id)
        all_videos.append(v)
    
    print('VIDEO_LIST') 
    print(VIDEO_LIST) 
    print('Number of video identified for encoding', len(all_videos))
    print("-" * 50)
    
    for video in all_videos:
        film_id = video.id
        s3_url=video.s3_url
        zen_job_id = video.zencoder_state
        project_id = video.project.id
        print ("Film %s is to be HD transcoded" % film_id)
        print ("s3 url= %s" % s3_url)
        print ("zen_job_id= %s" % zen_job_id)
        print ("project_id= %s" % project_id)
        if not zen_job_id:
            print('zencoder job id not present')
            if s3_url:  # don't do anything with no s3_url
                print('s3_url is present')
                location = s3_url
                ext = location.split('.')[-1]
                outputs = get_outputs(project_id, film_id)
                response = client.job.create(location, outputs=outputs)
                zen_job_id = response.body['id']
                print ("new created job id = %s" % zen_job_id)
                video.zencoder_state =zen_job_id
                video.save()
                print ("updated video %s with zencoder = %s \n" %(film_id, zen_job_id))
            else:
                print('s3_url not present')
        else:
            try:
                print ("Already Zencoding started with %s" % zen_job_id)
                resp = client.job.details(zen_job_id)
                if resp.body['job']['state'] == 'finished':
                    #we cansave the response to our database depending on the need
                    print('response says encoding finished')
                    video.zencoded = 1
                    video.save()
                    print (" done processing film %s \n" % film_id)
                if resp.body['job']['state'] == 'failed':
                    video.zencoded = 0
                    video.save()
            except:
                print('some issue')

        print('-' * 25)
    print("-" * 50)
    cleanup()
    print('video encoding script ended\n\n')

{% endhighlight %}

now to stream our videos we would use [jwplayer](http://www.jwplayer.com/), you would need a paid version for that

last but not least we have to use crossdomain.xml in the root of the s3 folder form where you are serving your files for more info
please check these links: [1](http://www.jwplayer.com/blog/delivering-hls-with-amazon-cloudfront/) and [2](https://support.jwplayer.com/customer/portal/articles/1430218-using-hls-streaming)
you can test your streams here on [demojwplayer](http://demo.jwplayer.com/stream-tester/) to see if everything has been set up perfectly

some other important links:

* [3](http://www.jwplayer.com/blog/encoding-hls-with-amazon-elastic-transcoder/)
* [4](https://support.jwplayer.com/customer/portal/articles/1403679-crossdomain-file-loading)






