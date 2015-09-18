---
layout: post
category: articles
analytics: true
tags: [AWS,Elastic Encoder,S3, Debian, python]
comments: true
share: true
title: "Aws Elastic Encoder"
date: 2015-08-24T22:25:54+05:30
---

This script gets a list of videos(objects in my case) which havent been encoded and tries to get them transcoded from correponding pipeline on Amazon Elastic Transcoder and then print the status on console, you can even update your app based on your needs.

{% highlight python linenos %}

import os
import sys
from django.conf import settings
from datetime import datetime


LOCAL_DEV=True
DEBUG= True


if LOCAL_DEV:
    sys.path.insert(0, '/home/aameer/path_to_project')
    sys.path.insert(1, '/home/aameer/path_to_project')

os.environ['DJANGO_SETTINGS_MODULE'] = 'projectname.settings'

import django
django.setup()

#imports from app
from apps.videos.models import Videos
import boto.elastictranscoder

#debug, un comment for boto logs
#boto.set_stream_logger('boto')

#based on server
if settings.DEBUG:
    #for your s3 bucket bucket
    PIPELINE_ID = 'bucket_pipe_line_id'
    #Singapore, check from aws docs
    DEFAULT_REGION_NAME='ap-southeast-1'

print('Pipeline used for the script:')
print(PIPELINE_ID,DEFAULT_REGION_NAME,'\n')

def cleanup():
    if os.path.isfile('video_encoding.lock'):
        os.unlink('video_encoding.lock')

def awstrancoder():
    print('getting elastic trancoder...')
    try:
        elastic_transcoder= boto.connect_elastictranscoder(aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,aws_access_key_id=settings.AWS_ACCESS_KEY_ID)
        elastic_transcoder.DefaultRegionName=DEFAULT_REGION_NAME
        print('AWS elastic_trancoder connetion successful...')
        return elastic_transcoder
    except Exception as awsete:
        print('Failed to connect to AWS Transcode')
        print(awsete)
        cleanup()

def get_input_object(project_name,video_id,orig_key,ext):
    #change this funtion according to your s3 key for input video
    print('gettting obeject with key:')
    print('%s/video/%s_%s.mp4'%(project_name,project_name,orig_key))
    input_object = {
        'Key': '%s/video/%s_%s.mp4'%(project_name,project_name,orig_key),
        'Container': ext,
        'AspectRatio': 'auto',
        'FrameRate': 'auto',
        'Resolution': 'auto',
        'Interlaced': 'auto'
    }
    return (input_object)

def get_output_objects(project_name,video_id):
    #here we want 4 different version of encoded vidoes, you can change the s3 key pattern for output videos and even the bucket here
    #you can get PresetId from amazon docs
    output_objects = [
        {
            'Key': '%s/video/%s_1080.mp4'%(project_name,video_id),
            'PresetId': '1351620000001-000001',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        },
        {
            'Key': '%s/video/%s_720.mp4'%(project_name,video_id),
            'PresetId': '1351620000001-000010',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        },
        {
            'Key': '%s/video/%s_480.mp4'%(project_name,video_id),
            'PresetId': '1351620000001-000020',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        },
        {
            'Key': '%s/video/%s_360.mp4'%(project_name,video_id),
            'PresetId': '1351620000001-000040',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
        }
    ]
    return (output_objects)

def create_transcoding_job(location,project_name,video,elastic_transcoder):
    # we are saving in the same bucket here, you can also save in another bucket
    ext = location.split('.')[-1]
    orig_key = location.split('_')[-1].split('.')[0]
    input_object = get_input_object(project_name,video.id,orig_key,ext)
    output_objects=get_output_objects(project_name,video.id)
    response=elastic_transcoder.create_job(pipeline_id=PIPELINE_ID,input_name=input_object,outputs=output_objects)
    aws_job_id =response['Job']['Id']
    print ("new created job id = %s" % aws_job_id)
    print('\n')
    video.awsencoder_state=aws_job_id
    video.save()
	
def main():
    print('='*50)
    print('timestamp[d/m/y-h:m:s]=',datetime.now().strftime("%d/%m/%Y-%H:%M:%S"),'\n')
    print('Video Encoding started')
    if os.path.exists('video_encoding.lock'):
        print('video lock is present, This may be beacuse script is running already or encountered an error when it was run last time')
        sys.exit()
    else:
        open('video_encoding.lock', 'w').write('running')
    elastic_transcoder=awstrancoder()
    #all video which havent been encoded or any other filter will work
    all_videos=Videos.objects.filter(encoded=False)
    print('no of videos identified for trancoding are %s \n' % all_videos.count())
    for video in all_videos:
        project_name='TestProject'
        video_id=video.id
        if not video.awsencoder_state:
            #no aws job id
            if video.s3_url:
                #check if s3_url is present , we can directly mentin any accessible url here
                create_transcoding_job(
                	location=video.s3_url,
                	project_name=project_name,
                	video=video,
                	elastic_transcoder=elastic_transcoder
                )
        else:
            job_id = video.awsencoder_state
            job_details = elastic_transcoder.read_job(id=job_id)['Job']
            job_status = job_details['Status']
            job_outputs = job_details['Outputs']
            if job_status == 'Submitted':
            	print('Job submitted \n')
            elif job_status == 'Progressing':
            	print('Job progressing \n')
            elif job_status == 'Complete':
                print('Job Complete \n')
                video.encoded = True
                video.save()
            elif job_status == 'Canceled':
            	print('Job Canceled \n')
            elif job_status == 'Error':
            	print('Job encountered Error \n')
            else:
            	print('some unkown job status error occured \n')
        print('-' * 25)
    print('-' * 50)
    cleanup()
    print('Video Encoding script Ended \n')

if __name__ == '__main__':
    main()

{% endhighlight %}
# to make the script run contiously (after every 10 minutes) you can run a cron job like this
`*/10 * * * * /usr/local/bin/python3.4  video_aws_elastictranscoder_script.py >> video_aws_elastictranscoder_script.log`



