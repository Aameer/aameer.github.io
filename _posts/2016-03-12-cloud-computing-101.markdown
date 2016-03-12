---
layout: post
analytics: true
tags: [Cloud computing, EC2, Shell Scripting, boto, jwplayer, S3, ffmpeg, hls, burned watermarking, multiprocessing]
comments: true
share: true
title: "Cloud Computing 101"
date: 2016-03-12T22:14:57+05:30
---

Cloud Computing:
----------------
Cloud computing, is a kind of Internet-based computing that provides shared processing resources and data to computers and other devices on demand. It is a model for enabling ubiquitous, on-demand access to a shared pool of configurable computing resources [wiki](https://en.wikipedia.org/wiki/Cloud_computing) or as our friend google says that it is The practice of using a network of remote servers hosted on the Internet to store, manage, and process data, rather than a local server or a personal computer.

What this post aims to do:
--------------------------
But there are tones of resources online where you would find more desciption what I will be trying to achieve with this post is to build a working model/example to depeer understanding of this concept.

Things this post assumes you know:
---------------------------------
As we will try to develope a system which will perform transcoding i.e a computing task (similar to zenoders transcoding or aws's transcoding for details please refer to some [previous articles](http://aameer.github.io/articles/)) over cloud and for this we will be using some shell scripts and python library for aws called boto. Moreover we will also be using aws cli for this purpose, so without a due lets do some coding.

Sections:
---------
We will divide our objective into several parts: 

* Use a cloud infrastructure(EC2 in our case) to do some work/computing (transcoding)
* Get the inputs from source(s3 in our case) and save them to destination(again s3 in our case)
* automate the workflow

Pre requisites:
--------------
Now to accomplish the first part i.e to get the transcoding done with watermarks on ec2 we will get an ec2 instance and connect to it with the help of aws console then we will install ffmpeg and awscli on it moreover we will then copy following scripts `aws_credentials.sh` , `copy_log.sh` and `encoding_script_watermarked.sh` this ec2 instance. Also I am using a font inside this folder Aaargh/Aaargh.ttf you can use any font once you link to it properly.

we will be using [ffmpeg](https://www.ffmpeg.org/about.html) for transcoding. Before moving further and installing ffmpeg you have to be sure that you have the necessary libraries by checking if the ffmpeg build is right for example for our purposes you can install ffmpeg 
`sudo add-apt-repository ppa:mc3man/trusty-media`
`sudo apt-get update`
`sudo apt-get dist-upgrade`
`sudo apt-get install ffmpeg`
[credits to source](http://www.faqforge.com/linux/how-to-install-ffmpeg-on-ubuntu-14-04/) 

we will aso be using aws cli to copy files from and to s3
#install pip
`sudo apt-get install python-pip`
#aws interaction with s3 with awscli==1.10.8
`pip install awscli`

once you have all the necessary scripts on the ec2 instance you can create and image for this instance, image-id  from which we can use later on
The scripts will handle the actual transcoding and copying of files

Now comes the part where you automate the transcoding and get an instance up whenever you like and transcode the video we will use python and boto for this for this part we will use `ec2_instance_switch.py` script and then we can just call the python shell.

{% highlight python linenos %}

python manage.py shell
from common.ec2_instance_switch import generate_hidden_burned_watermark_screener
generate_hidden_burned_watermark_screener()

{% endhighlight %}

if everything went fine you should see your instance getting up which you can check from ec2 console too , It will copy the original file (for now assumed to be mp4) to ec2 and then burn the watermark text at desiered position with hls (adaptive streaming enabled) and then save it back on s3 and give us the link for final master index file. After the transcoding is done it will close the instance and save the logs on s3 for future refrence it will also save the time it took for different transcode versions to complete.

Code Files which you will need:

###aws_credentials.sh:

{% highlight python linenos %}

#!/bin/bash
export AWS_ACCESS_KEY_ID='your_aws_key'
export AWS_SECRET_ACCESS_KEY='your_aws_secret'
export AWS_DEFAULT_REGION='your_aws_region'

{% endhighlight %}


###copy_logs.sh:

{% highlight python linenos %}

#!/bin/bash
source ./aws_credentials.sh
# help text displayed at various times to guide user how to invoke the bash script
helptext="$(basename "$0") [-h] [-f p v] -- program to get burned in watermark hls videos with master list with ffmpeg logs
e.g mutliple word argument processor:
time bash copy_logs.sh --projectid=3631 --mothervideoid="3799" --videoid="3798" -server="your_bucket_name"
e.g with log files:
(time bash copy_logs.sh -p="3631" -m="3799" -v="3800" -s="your_bucket_name") 2>> encoding_logs.log
where:
    -h  show this help text
    -p  project id
    -s  server
    -m  mother video id
    -v  video id"

#for parsing command line arguments

#Display message incase no arguments are supplied
if [[ $# -eq 0 ]] ; then
    echo "please provide necessary arguments check help text for guidance:" >> encoding_logs.log
    echo $helptext >> encoding_logs.log
    exit 0
fi


for i in "$@"
do
case $i in
    -p=*|--projectid=*)
    INPUT_PROJECT_ID="${i#*=}"
    shift # past argument=value
    ;;
    -v=*|--videoid=*)
    INPUT_VIDEO_ID="${i#*=}"
    shift # past argument=value
    ;;
    -m=*|--mothervideoid=*)
    INPUT_MOTHER_VIDEO="${i#*=}"
    shift # past argument=value
    ;;
    -s=*|--server=*)
    SERVER="${i#*=}"
    shift # past argument=value
    ;;
    --default)
    DEFAULT=YES
    shift # past argument with no value
    ;;
    *)
    echo 'please provide correct arguments check help text for guidance:'
    echo $helptext
    exit 0
    ;;
esac
done

#print the argumnets read
echo "INPUT PROJECT ID  = ${INPUT_PROJECT_ID}"
echo "INPUT VIDEO ID   = ${INPUT_VIDEO_ID}"
echo "INPUT MOTHER VIDEO ID   = ${INPUT_MOTHER_VIDEO}"
echo "SERVER NAME   = ${SERVER}"

#setting the values for script as per input values
export PROJECT_ID=${INPUT_PROJECT_ID}
export VIDEO_ID=${INPUT_VIDEO_ID}
export SERVER_NAME=${SERVER}
export MOTHER_VIDEO_ID=${INPUT_MOTHER_VIDEO}

echo $AWS_ACCESS_KEY_ID
#this is the place where from you copy the original video which will be transcoded you can alter it as per your need
aws s3 cp  encoding_logs.log s3://$SERVER_NAME/$PROJECT_ID/video/$MOTHER_VIDEO_ID/$VIDEO_ID/

#stop instance
export AWS_DEFAULT_REGION='us-east-1e'
export InstanceId=`curl http://169.254.169.254/latest/meta-data/instance-id;echo`
aws ec2 stop-instances --instance-ids ${InstanceId} --region us-east-1

{% endhighlight %}

###ec2_instance_switch.py:

{% highlight python linenos %}

import boto
import time
import json
import simplejson
import datetime
from boto.s3.connection import S3Connection
from django.conf import settings

# AWS credentials
aws_key = settings.AWS_ACCESS_KEY_ID
aws_secret = settings.AWS_SECRET_ACCESS_KEY


def get_conection():
    conn = boto.connect_ec2(aws_key, aws_secret)
    return conn

def get_watermark_position(video):
    #based on the parent video watermark position choose one from below
    #1.Bottom left
    #2.Bottom center 
    #3.Bottom right
    #4.Middle left
    #5.Middle center 
    #6.Middle right
    #7.Top left
    #8.Top center 
    #9.Top right
    # you can get it for other positions too
    if video.watermark_pos == 'top':
        burned_position = '8'
    elif video.watermark_pos == 'middle':
        burned_position = '5'
    elif video.watermark_pos == 'bottom':
        burned_position = '2'
    else:
        print("This value of watermark position not known, using default")
        burned_position = '2'
    return burned_position


def get_video_duration(video):
    #add the watermark and send the time for both the full length burned watermark  and also add the position for the visible watermark
    try:
        # you must have video duration in milliseconds
        video_duration_in_ms=56000
        miliseconds = int(video_duration_in_ms)
        hours, milliseconds = divmod(miliseconds, 3600000)
        minutes, milliseconds = divmod(miliseconds, 60000)
        seconds = float(milliseconds) / 1000
        burnedin_end_time = "%i:%02i:%06.3f" % (hours, minutes, seconds)
        #'0:00:52.254'
        return burnedin_end_time
    except Exception as durexecp:
        print(durexecp)

def get_video_details(video):
    try:
        #get base file
        #some_s3_link/test_file.mo4
        original_base_file = video.original_url.split('/')[-1]#test_file.mp4
        #get watermark position
        burned_position = get_watermark_position(video) 
        #get video durations
        burnedin_end_time = get_video_duration(video)
        return original_base_file,burned_position, burnedin_end_time
    except Exception as vnaexp:
        print('Some issue occured while getting video details')
        print(vnaexp)


def keyexistsonS3(key):
    conn = S3Connection(aws_secret_access_key=aws_secret,aws_access_key_id=aws_key)
    bucket = conn.get_bucket(settings.AWS_BUCKET_NAME, validate=True)
    try:
        # Will hit the API to check if it exists.
        possible_key = bucket.get_key(key)
        if possible_key:
            return True
        else:
            return False
    except Exception as e:
        return False

def transcode_request_for_ec2(watermark_text,mother_video):
    
    mother_video_id =str(mother_video.id)
    new_video = mother_video
    new_video.pk = None #django shuould set it 
    new_video.save()
    project_id = str(new_video.project.id)
    #for simple test you can just input these values, new video has most details same as old one except the watermarking
    original_base_file, burned_position, hidden_position, burnedin_end_time, h_burnedin_start_time, h_burnedin_end_time = get_video_details(new_video)
    
    video_id = str(new_video.id)

    bucket_name = settings.AWS_BUCKET_NAME 

    # Connect to EC2
    conn = get_conection()
    

    #we can add a parameter to check which formats we need in future
    BOOTSTRAP_SCRIPT = """#!/bin/bash -ex
    exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
    echo BEGIN
    date
    cd /home/ubuntu/
    pwd
    while  [  ! -f ./encoding_script_watermarked.sh ];   do      sleep 2;echo "loading script...";   done
    echo "script loaded"
    ((time bash ./encoding_script_watermarked.sh -p='%(PROJECT_ID)s' -m='%(MOTHER_VIDEO_ID)s' -v='%(VIDEO_ID)s' -f='%(FILE_NAME)s' -b="%(BURNED_END_TIME)s" -a="%(BURNED_POSITION)s" -w="%(WATERMARK_TEXT)s" -s='%(BUCKET_NAME)s') &>> encoding_logs.log; bash ./copy_logs.sh -p='%(PROJECT_ID)s' -m='%(MOTHER_VIDEO_ID)s' -v='%(VIDEO_ID)s' -s='%(BUCKET_NAME)s')
    echo END
    """

    burned_end_time= str(burnedin_end_time)

    startup = BOOTSTRAP_SCRIPT % {
        'PROJECT_ID': project_id,
        'VIDEO_ID': video_id,
        'FILE_NAME': original_base_file,
        'BURNED_END_TIME': burned_end_time,
        'BURNED_POSITION': burned_position,
        'WATERMARK_TEXT': watermark_text,
        'MOTHER_VIDEO_ID':mother_video_id,
        'BUCKET_NAME': bucket_name}

    print(startup)

    instance_reservation = conn.run_instances(
        image_id="ami-ac3640c1",# use the ami for the image which we created earlier
        instance_type="c3.xlarge",#instance type you want to use
        key_name="your_key_file",
        security_groups=["ffmpeg-test-security-group"],
        user_data=startup
    )
    instance = instance_reservation.instances[0]
    print(instance)

    #wait till the instance is not running
    while instance.state != 'running':
        print ('...instance is %s' % instance.state)
        time.sleep(10)
        instance.update()

    #tag instcance once its running
    instance_name = str(mother_video_id)+'-'+str(video_id)+'-'+str(datetime.datetime.now())
    print(instance_name)
    instance.add_tag("Name",instance_name) 

    #now the instance is running we should check for processed data in s3 after some avegare time
    
    #wait till instance has not stopped
    while instance.state != 'stopped':
        print ('...instance is %s' % instance.state)
        time.sleep(10)
        instance.update()
    
    #check if the files are present on s3, .ts files , index files and master file
    if keyexistsonS3(project_id+'/video/'+mother_video_id+'/'+video_id+'/'+'master_'+project_id+'_'+video_id+'.m3u8'):
    
        master_url_tobe_saved = 'https://'+str(settings.AWS_BUCKET_NAME)+'.s3.amazonaws.com/'+project_id+'/video/'+mother_video_id+'/'+video_id+'/'+'master_'+project_id+'_'+video_id+'.m3u8'
        print(master_url_for_hls_video)        
    else:
        #if not then process failed and we can retry and report and should never terminate the instance, also get a failuer mail
        print('some s3 files are missing something went haywire with instance %s' % instance)
        return None

def generate_hidden_burned_watermark_screener():
    #I am using a video object here for simplicity
    mother_video_id ="3627"
    watermark_text ="Ec2 test 104"
    mother_video = Videos.objects.get(id=mother_video_id)


    #generate the burned watermark video
    print('calling ec2')
    #pdb.set_trace()
    burned_video = transcode_request_for_ec2(watermark_text,mother_video)

    print('got the video')
    print(burned_video)

{% endhighlight %}

###encoding_script_watermarked.sh

{% highlight python linenos %}

#!/bin/bash

source ./aws_credentials.sh

# help text displayed at various times to guide user how to invoke the bash script
helptext="$(basename "$0") [-h] [-f p v] -- program to get burned in watermark hls videos with master list with ffmpeg
e.g mutliple word argument processor:
time bash encoding_script_watermarked.sh --projectid="3631" --videoid="3798" --videofile="3631_6.mp4" -server="your_bucket"
with log files:
9time bash encoding_script_watermarked.sh -p="3631" -m="3799" -v="3800" -f="3631_6.mp4" -b="0:00:52.254" -i="0:00:17.418" -z="0:00:17.433" -a="5" -c="6" -w="this is test watermark text" -s="your_bucket") 2>> encoding_logs.log
where:
    -h  show this help text
    -f  link to video, file name
    -p  project id
    -s  server
    -b  burned watermark end time
    -a  burned watermakr position
    -w  watermakr text
    -m  mother video id
    -v  video id"

#for parsing command line arguments
#Display message incase no arguments are supplied
if [[ $# -eq 0 ]] ; then
    echo "please provide necessary arguments check help text for guidance:" >> encoding_logs.log
    echo $helptext >> encoding_logs.log
    exit 0
fi

for i in "$@"
do
case $i in
    -f=*|--videofile=*)
    INPUT_VIDEO_FILE_KEY="${i#*=}"
    shift # past argument=value
    ;;
    -p=*|--projectid=*)
    INPUT_PROJECT_ID="${i#*=}"
    shift # past argument=value
    ;;
    -v=*|--videoid=*)
    INPUT_VIDEO_ID="${i#*=}"
    shift # past argument=value
    ;;
    -b=*|--burnedendtime=*)
    INPUT_BURNED_END_TIME="${i#*=}"
    shift # past argument=value
    ;;
    -a=*|--burnedposition=*)
    INPUT_BURNED_POSITION="${i#*=}"
    shift # past argument=value
    ;;
    -w=*|--watermarktext=*)
    INPUT_WATERMARK_TEXT="${i#*=}"
    shift # past argument=value
    ;;
    -m=*|--mothervideoid=*)
    INPUT_MOTHER_VIDEO="${i#*=}"
    shift # past argument=value
    ;;
    -s=*|--server=*)
    SERVER="${i#*=}"
    shift # past argument=value
    ;;
    --default)
    DEFAULT=YES
    shift # past argument with no value
    ;;
    *)
    echo 'please provide correct arguments check help text for guidance:'
    echo $helptext
    exit 0
    ;;
esac
done
{
#print the argumnets read
echo "INPUT VIDEO FILE  = ${INPUT_VIDEO_FILE_KEY}"
echo "INPUT PROJECT ID  = ${INPUT_PROJECT_ID}"
echo "INPUT VIDEO ID   = ${INPUT_VIDEO_ID}"
echo "INPUT MOTHER VIDEO ID   = ${INPUT_MOTHER_VIDEO}"
echo "SERVER NAME   = ${SERVER}"

#for dynamically creating .ass file
echo "INPUT BURNED WATERMARK END TIME     = ${INPUT_BURNED_END_TIME}"
echo "INPUT BURNED WATERMARK POSITION     = ${INPUT_BURNED_POSITION}"
echo "INPUT WATERMARK TEXT  = ${INPUT_WATERMARK_TEXT}"

#setting the values for script as per input values
export PROJECT_ID=${INPUT_PROJECT_ID}
export VIDEO_ID=${INPUT_VIDEO_ID}
export SERVER_NAME=${SERVER}
export INPUT_LINK=${INPUT_VIDEO_FILE_KEY} 
export MOTHER_VIDEO_ID=${INPUT_MOTHER_VIDEO}

#for dynamically creating .ass file
export BURNED_END_TIME=${INPUT_BURNED_END_TIME}

export BURNED_POSITION=${INPUT_BURNED_POSITION}
export WATERMARK_TEXT=${INPUT_WATERMARK_TEXT}

#creating .ass file dynamically for generating different watermarks and it also gives on control on styling as mentioned below
cat << EOF > "dynamic_subtitle_"$VIDEO_ID".ass"
[Script Info]
; Script generated by FFmpeg/Lavc57.24.103
ScriptType: v4.00+
PlayResX: 384
PlayResY: 288

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, SecondaryColour, OutlineColour, BackColour, Bold, Italic, Underline, StrikeOut, ScaleX, ScaleY, Spacing, Angle, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
Style: Default,Arial,16,&H99999999,&H99999999,&H0,&H0,0,0,0,0,100,100,0,0,1,0,0,$BURNED_POSITION,10,10,10,0

[Events]
Format: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text
Dialogue: 0,0:00:00.00,$BURNED_END_TIME,Default,,0,0,0,,$WATERMARK_TEXT
EOF

echo "copying original from s3 ..." >> encoding_logs.log

#download the file from s3 and parse it for name, for now assuming that the file is mp4
(time aws s3 cp s3://$SERVER_NAME/$PROJECT_ID/video/$INPUT_LINK . ) &>> encoding_logs.log

#set downloaded file as the input for transcoding
export INPUT_VIDEO_FILE=$INPUT_LINK 

#check if all the necessary arguments are present, note spacing is important in bash ( as Bash uses spaces to tokenise scripts)
if [[ PROJECT_ID != "" && VIDEO_ID != "" && INPUT_VIDEO_FILE != "" && SERVER_NAME != "" ]]
then

#starting notifications
echo "The ffmpeg encoding is starting \n"
echo "encoding for $PROJECT_ID for video id $VIDEO_ID has beengg started using input file $INPUT_VIDEO_FILE  \n" >> encoding_logs.log
#for -profile:v details are here- https://trac.ffmpeg.org/wiki/Encode/H.264, for other options check, ffmpeg -h
echo "temp directory for video to be saved in s3_temp/your_bucket/$PROJECT_ID/videos/$VIDEO_ID/" >> encoding_logs.log
#make directory if not present 
mkdir -p "s3_temp/your_bucket/$PROJECT_ID/videos/$VIDEO_ID/"

#360 version
echo ffmpeg -i $INPUT_VIDEO_FILE -profile:v baseline -level 4.0 -vf "scale=-2:360,subtitles='dynamic_subtitle_'$VIDEO_ID'.ass':force_style='FontName=Aaargh/Aaargh.ttf,PrimaryColour=&H664c4c4c"  -start_number 0 -hls_time 10 -hls_list_size 0 -f hls s3_temp/your_bucket/$PROJECT_ID/videos/$VIDEO_ID/$VIDEO_ID"_360_.m3u8"

echo "starting encoding ..." >> encoding_logs.log

#360 version 
(time ffmpeg -i $INPUT_VIDEO_FILE -profile:v baseline -level 4.0 -vf "scale=-2:360,subtitles='dynamic_subtitle_'$VIDEO_ID'.ass':force_style='FontName=Aaargh/Aaargh.ttf,PrimaryColour=&H664c4c4c"  -start_number 0 -hls_time 10 -hls_list_size 0 -f hls s3_temp/your_bucket/$PROJECT_ID/videos/$VIDEO_ID/$VIDEO_ID"_360_.m3u8" && echo "360 version:" >> encoding_logs.log ) 2>> encoding_logs.log &

#480 version 
(time ffmpeg -i $INPUT_VIDEO_FILE -profile:v baseline -level 4.0 -vf "scale=-2:480,subtitles='dynamic_subtitle_'$VIDEO_ID'.ass':force_style='FontName=Aaargh/Aaargh.ttf,PrimaryColour=&H664c4c4c"  -start_number 0 -hls_time 10 -hls_list_size 0 -f hls s3_temp/your_bucket/$PROJECT_ID/videos/$VIDEO_ID/$VIDEO_ID"_480_.m3u8"  && echo "480 version" >> encoding_logs.log) 2>> encoding_logs.log &

#720 version
(time ffmpeg -i $INPUT_VIDEO_FILE -profile:v baseline -level 4.0 -vf "scale=-2:720,subtitles='dynamic_subtitle_'$VIDEO_ID'.ass':force_style='FontName=Aaargh/Aaargh.ttf,PrimaryColour=&H664c4c4c"  -start_number 0 -hls_time 10 -hls_list_size 0 -f hls s3_temp/your_bucket/$PROJECT_ID/videos/$VIDEO_ID/$VIDEO_ID"_720_.m3u8" && echo "720 version" >> encoding_logs.log ) 2>> encoding_logs.log &

#1080 version
(time ffmpeg -i $INPUT_VIDEO_FILE -profile:v baseline -level 4.0 -vf "scale=-2:1080,subtitles='dynamic_subtitle_'$VIDEO_ID'.ass':force_style='FontName=Aaargh/Aaargh.ttf,PrimaryColour=&H664c4c4c"  -start_number 0 -hls_time 10 -hls_list_size 0 -f hls s3_temp/your_bucket/$PROJECT_ID/videos/$VIDEO_ID/$VIDEO_ID"_1080_.m3u8" && echo "1080 version" >> encoding_logs.log ) 2>> encoding_logs.log &

echo "The ffmpeg encoding has ended\n" >> encoding_logs.log

#creating a master playlist for hls videos for adaptive streaming 
cat << EOF > "s3_temp/your_bucket/"$PROJECT_ID/videos/$VIDEO_ID/master_$PROJECT_ID"_"$VIDEO_ID".m3u8"
#EXTM3U
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=1128000,RESOLUTION=640x360,CODECS="avc1.42001e,mp4a.40.2"
${VIDEO_ID}_360_.m3u8
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=1803000,RESOLUTION=854x480,CODECS="avc1.42001f,mp4a.40.2"
${VIDEO_ID}_480_.m3u8
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=3138000,RESOLUTION=1280x720,CODECS="avc1.42001f,mp4a.40.2"
${VIDEO_ID}_720_.m3u8
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=4865000,RESOLUTION=1920x1080,CODECS="avc1.420028,mp4a.40.2"
${VIDEO_ID}_1080_.m3u8
EOF

#uploading files to s3
echo "uploading processed files to s3 ..." >> encoding_logs.log
(time aws s3 cp s3_temp/$SERVER_NAME/$PROJECT_ID/videos/$VIDEO_ID/ s3://$SERVER_NAME/$PROJECT_ID/video/$MOTHER_VIDEO_ID/$VIDEO_ID/ --recursive) 2>> encoding_logs.log


else

echo "Please provide the necessart arguments"
echo $helptext >> encoding_logs.log

fi
} || {

echo "some error occured while runing transcoding bash script"

}

{% endhighlight %}

So thus you have now trancoded the video over cloud . Congratulations you have completed your cloud computing 101 lesson !
