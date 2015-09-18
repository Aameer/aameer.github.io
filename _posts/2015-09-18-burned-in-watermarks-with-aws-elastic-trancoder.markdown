---
layout: post
category: articles
analytics: true
tags: [Watermarks, Burned Watermarks, python, Django, AWS, Elastic trancoder, Text to Image Conversion, PIL]
comments: true
share: true
title: "Burned In watermarks with AWS Elastic Trancoder"
date: 2015-09-18T16:57:00+05:30
---

Introduction:
-------------
AWS elastic transcoder allows us to create a trancoded video with burned in watermarked trancoded videos.For details about setting up aws 
elastic encoder from scratch check my post regarding [aws elastic transcoder](http://aameer.github.io/articles/aws-elastic-encoder/). Here I 
will be discussing how to get a burned in watermakred transcoded video you just have to replace `get_output_objects` function in the previous 
post and add `create_watermark_image` and `upload_watermark_image_to_s3` fucntions to your code and you are good to go.
Note you have to update the presetIds and locations and paths as per your need

Text to Image Conversion:
-------------------------
Aws Elastic trancoder doesnt allow text as watermarks so what we do is convert the text into Image using python's Image Library and then save
it on s3 and the use it as watermark image.Hope you like the post

Code:
-----
{% highlight python linenos %}

import os
from PIL import Image
from PIL import ImageDraw
from PIL import ImageFont
from io import StringIO
import textwrap
from django.core.files.uploadedfile import InMemoryUploadedFile
from common.indee_s3 import get_bucket_connection

watermark_text='Burned Watermark Text'

def upload_watermark_image_to_s3(image,project_id):
        thumb_file = InMemoryUploadedFile(image, None, 'watermark_text_img.png', 'image/png',image.size, None)
        filename = thumb_file.name
        name, ext = os.path.splitext(filename)
        bucket = get_bucket_connection()
        image.thumbnail(image.size, Image.ANTIALIAS)
        #update location as per your need
        image.save(os.path.join(os.path.dirname(__file__), '../../static/s3_temp/%s%s%s' % (project_id,'_watermark', ext)))
        #update key as per your need
        k = bucket.new_key('%s/project/poster/%s%s%s' % (project_id, project_id,'_watermark', ext))
        print('bucket key', k)
        k.set_contents_from_filename(os.path.join(os.path.dirname(__file__), '../../static/s3_temp/%s%s%s' % (project_id,'_watermark', ext)))
        bucket.set_acl('public-read', k.name)
        # update name as per your need
        print('watermark_text_image_key',k.name)
        return k.name

def create_watermark_image(text, project_id):
        #allowing one line of 30 chars and our height can accomodate two such lines thus 60 chars is the limit
        para = textwrap.wrap(text,width=35)
        W,H=(900, 100)
        #white
        text_color=(255, 255, 255,64)
        #change this valas per you font location
        font_location='/home/aameer/Documents/projects/verdanab.ttf'
        text_font = ImageFont.truetype(font_location,33)
        image = Image.new('RGBA', (W,H))
        d = ImageDraw.Draw(image)
        #padding and height for the lines breaking the text into several lines
        current_h, pad = 10, 2
        for line in para:
            w, h = d.textsize(line, font=text_font)
            d.text(((W - w) / 2, current_h), line, fill=text_color,font=text_font)
            current_h += h + pad 
        key_name = upload_watermark_image_to_s3(image,project_id) 
        return key_name

#change preset
def get_output_objects(project_id,video):
    video_id=video.id
    #get watermark image
    input_key = create_watermark_image(watermark_text,project_id)
    #set watermark_position based on watermark_id in your aws presets
    watermark_id=='bottom'
    output_objects = [
        {
            'Key': 'video/%s_1080.mp4'%(video_id),
            'PresetId': '1441956280541-11',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
            'Watermarks':[{
                'InputKey':input_key,
                'PresetWatermarkId':watermark_id
            }],
        },
        {
            'Key': 'video/%s_720.mp4'%(video_id),
            'PresetId': '1441956203492-tw2',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
            'Watermarks':[{
                'InputKey':input_key,
                'PresetWatermarkId':watermark_id
            }],
        },
        {
            'Key': 'video/%s_480.mp4'%(video_id),
            'PresetId': '1441956110883-mt2h91',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
            'Watermarks':[{
                'InputKey':input_key,
                'PresetWatermarkId':watermark_id
            }],
        },
        {
            'Key': 'video/%s_360.mp4'%(video_id),
            'PresetId': '1441956046335-g30uf',
            'Rotate': 'auto',
            'ThumbnailPattern': '',
            'Watermarks':[{
                'InputKey':input_key,
                'PresetWatermarkId':watermark_id
            }],
        }
    ]
    return (output_objects)

{% endhighlight %}
