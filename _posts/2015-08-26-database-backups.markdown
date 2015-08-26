---
layout: post
category: articles
analytics: true
tags: [Databases,S3,Debian,python,Django,shell,PostgreSQL]
comments: true
share: true
title: "Database backups"
date: 2015-08-26T12:21:45+05:30
---

Following is a bash script for taking database backups saving them on server disk , we also delete files which are older than 7 days.
For the code mentioned I have used python3.4, ubuntu 14.04, django 1.7 and database used was PostgreSQL 

`#!/bin/bash`

Specify the temporary backup directory 
`BKUPDIR="/home/aameer/dbbackup"`

Below lines are to delete any files older then 7 days
`find $BKUPDIR -type f -mtime +7 | xargs rm -Rf`

Database Name
`dbname="test"`
`dbuser="test"`
`dbpasswd="test"`

store the current date
`date=`date +"%s"``

Dump the psql database with the current date and compress it.
`PGPASSWORD=$dbpasswd /usr/bin/pg_dump -U $dbuser $dbname -f $BKUPDIR/production_live_backup_$date.$dbname.sql`

log the time
`echo 'dump for' $dbname 'taken on' `date +%c -d "$d"``

Next is runnig a cronjob which executes this script at regular intervals which creates db dbackup every four hours at mentioned times
`0 0,4,8,12,16,20 * * * bash /home/aameer/create_db_backup.sh >> /home/aameer/logs/user/create_db_backup.log`

Last piece of puzzle is getting the saved scripts from server to s3
{% highlight python linenos %}
#! /usr/local/bin/python3.4 -u
import sys
import os
import datetime
sys.path.insert(0, '/home/aameer/path_to_app')
sys.path.insert(1, '/home/aameer/path_to_app')
os.environ['DJANGO_SETTINGS_MODULE'] = 'projectname.settings'
from django.conf import settings
from boto.s3.connection import S3Connection
from boto.s3.key import Key

bucket_name = 'aameer-backups'
conn = S3Connection(aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,
                            aws_access_key_id=settings.AWS_ACCESS_KEY_ID)

bucket = conn.create_bucket(bucket_name)
os.chdir("/home/aameer/dbbackup/")
print(bucket)
for files in os.listdir("."):
    if files.endswith(".sql"):
        mpu = bucket.initiate_multipart_upload('/DB/%s' % files)
        mpu.upload_part_from_file(open(os.path.join('%s' % files)), part_num=1)
        cmpu_250 = mpu.complete_upload()
        print (cmpu_250)
        bucket.set_acl('private', cmpu_250.key_name)
        print ('upload file %s' % files)                   
{% endhighlight %}
Now you are taking the data backups at regular intervals.Hope you liked the post
