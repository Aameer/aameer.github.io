---
layout: post
category: articles
analytics: true
tags: [Zencoder,S3,Debian,python,Django]
comments: true
share: true
title: "Encoding with zencoder"
date: 2015-08-29T01:44:40+05:30
---
Add the video ids(django model objects/instances in example case) for which you want to run the script for.

{% highlight python linenos %}

import sys
import os
LOCAL_DEV = True
DEBUG = True

VIDEO_LIST=['3515'] #add ids for the video objects for which we want to run the zencoder

# add Project path and Lib path, example below
if LOCAL_DEV:
    sys.path.insert(0, '/home/aameer/path_to_project')
    sys.path.insert(1, '/home/aameer/path_to_project')

os.environ['DJANGO_SETTINGS_MODULE'] = 'projectname.settings'

from zencoder import Zencoder
from boto.s3.connection import S3Connection
import simplejson
from django.conf import settings
from datetime import datetime
import django
django.setup()
from apps.videos.models import Videos

def get_bucket():
    try:
        s3_conn = S3Connection(aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,
                               aws_access_key_id=settings.AWS_ACCESS_KEY_ID)
        bucket = s3_conn.get_bucket(settings.AWS_BUCKET_NAME)
        if DEBUG:
            print (" S3 connection successful")
        return bucket
    except Exception as be:
        print ("Failed to connect to S3 ")
        print (be)
        cleanup()
        sys.exit(-1)


def get_zencoder_client():
    try:
        client = Zencoder(settings.ZENCODER_API_KEY)
        if DEBUG:
            print (" Zencoder connection successful")
        return client
    except Exception as ze:
         print ("Failed to connect to Zencoder")
         print (ze)
         cleanup()
         sys.exit(-1)

# Function to destroy the lock if it exists.
def cleanup():
    if os.path.isfile('video_encoding.lock'):
        os.unlink('video_encoding.lock')


def get_video_objetct(v_id):
    try:
        video =Videos.objects.get(id=v_id)
        print ("checking for video id %s" %(v_id))
        return video
    except Exception as ve:
         print ("Failed get video id %s" %(v_id))
         print (ve)
         cleanup()
         sys.exit(-1)

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
                outputs = [{'label': '%s_320' % project_id,
                            'url': 's3://%s/%s/video/%s_320.mp4' % (settings.AWS_BUCKET_NAME, project_id, film_id),
                            "width": "320", "height": "180", "format": "mp4"},
                           {'label': '%s_854' % project_id,
                            'url': 's3://%s/%s/video/%s_854.mp4' % (settings.AWS_BUCKET_NAME, project_id, film_id),
                            "width": "854", "height": "480", "format": "mp4", "skip": {"min_size": "480x854"}},
                           {'label': '%s_1280' % project_id,
                            'url': 's3://%s/%s/video/%s_1280.mp4' % (settings.AWS_BUCKET_NAME, project_id, film_id),
                            "width": "1280", "height": "720", "format": "mp4", "skip": {"min_size": "720x1280"}},
                            {'label': '%s_1920' % project_id,
                            'url': 's3://%s/%s/video/%s_1920.mp4' % (settings.AWS_BUCKET_NAME, project_id, film_id),
                            "width": "1920", "height": "1080", "format": "mp4", "skip": {"min_size": "800x1920"}}
                           ]
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

if __name__ == '__main__':
    main()

{% endhighlight %}

To make the script run contiously (after every 10 mins )you can run a cron job like this
`*/10 * * * * /usr/local/bin/python3.4  video_aws_elastictranscoder_script.py >> video_aws_elastictranscoder_script.log`

