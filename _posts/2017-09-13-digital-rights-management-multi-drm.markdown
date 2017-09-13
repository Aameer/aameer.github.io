---
layout: post
title: "Digital Rights Management (Multi - DRM)"
category: articles
analytics: true
tags: [DRM, multi-DRM, Playready, Widevine, Fairplay, Secure Adaptive Streaming, HLS, DASH, jwplayer, Ecnryption, AES-128]
title: "Digital Rights Management (multi - drm)"
comments: true
share: true
date: 2017-09-13T20:43:41+05:30
---

# what :question: 

> *Digital rights management (DRM) schemes are various access control technologies that are used to restrict usage of proprietary hardware and copyrighted works. DRM technologies try to control the use, modification, and distribution of copyrighted works (such as software and multimedia content), as well as systems within devices that enforce these policies* .For more details check [wiki](https://en.wikipedia.org/wiki/Digital_rights_management) :thinking_face: 

In 1983, a very early implementation of Digital Rights Management (DRM) was the Software Service System (SSS) devised by the Japanese engineer Ryuichi Moriya. Today you DRM has expanded to traditional hardware products, from Keurig's coffeemakers, Philips' light bulbs, mobile device power chargers, and John Deere's tractors. You can find it applied in 
Computer Games by publishers like Electronic Arts, Ubisoft and The Sims3 and also in IBooks, Kindle etc which use DRM scheme of one or the other kind. :robot_face: 

# why :question: 
Although use of DRM is not universally accepted. Proponents of DRM argue that it is necessary to prevent intellectual property from being copied freely, just as physical locks are needed to prevent personal property from being stolen, that it can help the copyright holder maintain artistic control, and that it can ensure continued revenue streams. Those opposed to DRM contend there is no evidence that DRM helps prevent copyright infringement, arguing instead that it serves only to inconvenience legitimate customers, and that DRM helps big business stifle innovation and competition. Furthermore, works can become permanently inaccessible if the DRM scheme changes or if the service is discontinued. :thumbsup_tone1: :thumbsdown: 

Irrespective of your view about the use of DRM if you are working in industries like entertainment and gaming which have been using DRM for a long you might still want to stick around to see what all the fuss it about and educate yourself. :nerd_face: 

# when is it going to be ready (or state now):question: 
The biggest hastle with DRM is the native support for differnet browsers on different devices. Check [this](https://drmtoday.com/platforms/) list to get an idea. So in order for you to provide your customers DRM service on most common browsers you will have to use more than one provider. A combination of Playread , Widevinew and Fairplay would cover for most part hence the term multi-DRM. :astonished: 

For more geeks pals out there check the state of [encrypted media extension here](https://caniuse.com/#search=drm) where you will also find some more resources , do check the html5rocks article by Sam Dutton.

who(service providers) and some history :question: 
* Widevine (google)
* Playready (microsoft)
* Fairplay (apple)

For this article we will focus on DASH first with widevine and playready. Fairplay uses HLS.

# How :question: 
Before moving forward skim through [this SO qs](https://stackoverflow.com/questions/822468/is-there-an-open-source-drm-solution) if you are wondering about open-source DRM solution. :crying_cat_face: 

To make our life easy we will use a multi-drm service . I recommedn ezdrm just for the level of support you will receive. It took me some time to figure out small details as not much material was available online hence quick support becomes very important in such cases. So for this article we will use ffmpeg, Bento4 and Jwplayer to get the job done you can also replace them with some alternatives after checking how the whole process works. :smile: 

Please note if I have skipped creation of any portion. It should be covered in further detail by [this base article about cloud platform](http://aameer.github.io/cloud-computing-101/).

as mentioned in the base article ffmpeg can be used for various purposes. Here we will use it for say burning subtitles and creating four different versions for adaptive streaming. Command would be like:

```bash

ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:360,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf" output_360.mp4

ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:480,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf" output_480.mp4

ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:720,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf" output_720.mp4

ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:1080,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf" output_1080.mp4

```

added `-codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut"` in above command because of [this issue](https://github.com/axiomatic-systems/Bento4/issues/85)

for creation of `dynamic_subtitle_new.ass check` the base article. Now we have 4 formats `output_1080`, `output_720`, `output_480`, `output_360`. Now comes the **benot4** into play.

# Bento4 :
> *Bento4 is a C++ class library and tools designed to read and write ISO-MP4 files. This format is defined in international specifications ISO/IEC 14496-12, 14496-14 and 14496-15. The format is a derivative of the Apple Quicktime file format, so Bento4 can be used to read and write most Quicktime files as well. Visit www.bento4.com for details.* 

you can get the binaries [here](https://www.bento4.com/downloads/). Note you will have to use python2.7 for this. I was planing to update it for python3 but was advised againt by **Gilles Boccon** (creator of bento4) in the interest of time. 

Now using bento4 we will fragment the output videos which we got from ffmpeg , command would be like 

```bash

bin/mp4fragment ffmpeg_ouput/output_360.mp4 fragmented_output/frag_360.mp4
bin/mp4fragment ffmpeg_ouput/output_480.mp4 fragmented_output/frag_480.mp4
bin/mp4fragment ffmpeg_ouput/output_720.mp4 fragmented_output/frag_720.mp4
bin/mp4fragment ffmpeg_ouput/output_1080.mp4 fragmented_output/frag_1080.mp4

```

more about [mp4fragment](https://www.bento4.com/documentation/mp4fragment/)

now I am assuming by now you have got access to http://www.ezdrm.com/. Once you have that you can use this python code to get parse values from xml response.

```python
def get_ezdrm_values(content_id=None):
    """
    Returns Ezdrm values if content_id supplied or creates new ones.
    """
    #TODO: move this in settings
    ezdrm_username ="your_user_name@domain.com"
    ezdrm_password ="your_pwd"
    if content_id:
        resp = requests.get("https://wvm.ezdrm.com/ws/LicenseInfo.asmx/GenerateKeys?U="+ezdrm_username+"&P="+ezdrm_password+"&C="+content_id)
    else:
        resp = requests.get("https://wvm.ezdrm.com/ws/LicenseInfo.asmx/GenerateKeys?U="+ezdrm_username+"&P="+ezdrm_password+"&C=")
    tree = ElementTree.fromstring(resp.content)
    ezdrm_data={}
    ezdrm_data["ContentID"] = tree.findall('.//WideVine')[0].find('ContentID').text
    ezdrm_data["Key"]= tree.findall('.//WideVine')[0].find('Key').text
    ezdrm_data["KeyHEX"]= tree.findall('.//WideVine')[0].find('KeyHEX').text
    ezdrm_data["KeyID"]= tree.findall('.//WideVine')[0].find('KeyID').text
    ezdrm_data["KeyIDGUID"]= tree.findall('.//WideVine')[0].find('KeyID').text
    ezdrm_data["KeyIDHEX"]= tree.findall('.//WideVine')[0].find('KeyIDGUID').text
    ezdrm_data["PSSH"]= tree.findall('.//WideVine')[0].find('PSSH').text
    ezdrm_data["ServerURL"]= tree.findall('.//WideVine')[0].find('ServerURL').text
    ezdrm_data["ServerGet"]= tree.findall('.//WideVine')[0].find('ServerGet').text
    ezdrm_data["ResponseRaw"]= tree.findall('.//WideVine')[0].find('ResponseRaw').text
    ezdrm_data["LAURL"]= tree.findall('.//PlayReady')[0].find('LAURL').text
    ezdrm_data["Checksum"]= tree.findall('.//PlayReady')[0].find('Checksum').text
    return ezdrm_data

```

ideally this should have be fine but I faced and issue from bento4

    Note: When I was using 'KeyIDHEX': 'cf5faXXX-XXd0-5XXa-bXX8-9de2eaXXX25' . I was getting error ERROR: Invalid argument format for --encryption-key option
    I checked the code and it was failing at line number 1043 (latest on github: https://github.com/axiomatic-systems/Bento4/blob/master/Source/Python/utils/mp4-dash.py#L1083) i.e

    1042             if len(kid_hex) != 32:
    1043                 raise Exception('Invalid argument format for --encryption-key option')

    so I changed the 'KeyIDHEX':'cf5faXXXXXd05XXabXX89de2eaXXX25' by removing the dashes and I was able to proceed ahead.



then used this command (please replace the the corresponding vales as you obtain from the above mentioned python function:

```bash

bin/mp4dash --encryption-key=KEYIDHEX:KEYHEX --playready-header=LA_URL:LAURL --widevine-header=#SERVERURL fragmented_output/frag_360.mp4 fragmented_output/frag_480.mp4 fragmented_output/frag_720.mp4 fragmented_output/frag_1080.mp4 --output-dir="outputfiles" --mpd-name="final_stream.mpd"

```

now you should have a folder `outputfiles` and which would have `final_stream.mpd` and other related video and audio files. You can host them on s3 now and serve them with **cloudfront**. Once you do that you can use **Jwplayer** to test your strem

:alembic:   
goto : https://developer.jwplayer.com/tools/stream-tester/
replace : file url with your url to your final_stream.mpd
and click widevine on chrome and playready on ie/edge and then paste corresponding values which you got from ezdrm in the function above

and then hit play. At this stage your stream must work and your video must get a license to be played. Also this stream will be adaptive.

To add hls to the workflow we just need to add one more command along with creation of DASH , More [here](https://www.bento4.com/developers/dash/encryption-and-drm/). We would also create HLS stream and hence Fairplay would also be covered more [here](https://www.bento4.com/developers/hls/)


Special thanks to Ezdrm guys especially to  **David Eisenbacher (founder ezdrm)** and **Gilles Boccon-Gibod (cretor of Bento4)** for the help. 

Some important links which were helpful in this course :
* http://www.ezdrm.com/
* https://ffmpeg.org/
* https://github.com/axiomatic-systems/Bento4
* https://drmtoday.com/platforms/
* https://tdngan.wordpress.com/2016/11/17/how-to-encode-multi-bitrate-videos-in-mpeg-dash-for-mse-based-media-players/
* https://trac.ffmpeg.org/wiki/Encode/H.264

not sure if necessary but keeping it as I remember adding this
`sudo apt-get install libavcodec-extra-54`

https://stackoverflow.com/questions/9764740/unknown-encoder-libx264/10027588#10027588
https://superuser.com/questions/908280/what-is-the-correct-way-to-fix-keyframes-in-ffmpeg-for-dash/1098329#1098329

With jwplayer I faced issue with edge and had reported to them so you can use https://bitmovin.com/mpeg-dash-hls-drm-test-player/ instead to test your steams incase they dont work on edge

Note for building ffmpeg which I have used check my stackoverflow [here] (https://stackoverflow.com/questions/35597083/how-to-build-ffmpeg-with-burn-text-on-hls-output-while-maintaining-the-aspect-ra)


sample :telescope:   
https://dyinlm8q22u8a.cloudfront.net/482/video/628/648/output/master_482_648.mpd
widevine: https://widevine-dash.ezdrm.com/proxy?pX=5E6ACA
playready: https://playready.ezdrm.com/cency/preauth.aspx?pX=6BDD75

sample ezdrm values :key: 

```python

{'ResponseRaw': '{"status":"OK","drm":[{"type":"WIDEVINE","system_id":"edef8ba979d64acea3c827dcd51d21ed"}],"tracks":[{"type":"SD","key_id":"z1+vdTzQX2qzGJ3i6iqVJQ==","key":"+lFeMZptOWNK9oPhm2IhSQ==","pssh":[{"drm_type":"WIDEVINE","data":"EhDPX691PNBfarMYneLqKpUlGghtb3ZpZG9uZSIQWWgkeECVN0KZcSWWiIhon0jj3JWbBg=="}]}]}', 'KeyIDGUID': 'z1+vdTzQX2qzGJ3i6iqVJQ==', 'KeyID': 'z1+vdTzQX2qzGJ3i6iqVJQ==', 'KeyHEX': 'fa515e319a6d39634af683e19b622149', 'PSSH': 'EhDPX691PNBfarMYneLqKpUlGghtb3ZpZG9uZSIQWWgkeECVN0KZcSWWiIhon0jj3JWbBg==', 'Checksum': '4xv5IGn1Xyk=', 'ServerURL': 'https://widevine-dash.ezdrm.com/proxy?pX=5E6ACA', 'Key': '+lFeMZptOWNK9oPhm2IhSQ==', 'LAURL': 'https://playready.ezdrm.com/cency/preauth.aspx?pX=6BDD75', 'ServerGet': 'request={"policy": "", "tracks": [ {"type": "SD"}], "content_id": "WWgkeECVN0KZcSWWiIhonw=="}', 'ContentID': 'WWgkeECVN0KZcSWWiIhonw==', 'KeyIDHEX': 'cf5faf75-3cd0-5f6a-b318-9de2ea2a9525'}

```
