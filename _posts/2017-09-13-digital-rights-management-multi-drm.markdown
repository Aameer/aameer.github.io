---
layout: post
title: "Digital Rights Management (Multi - DRM)"
category: articles
analytics: true
tags: [DRM, multi-DRM, Playready, Widevine, Fairplay, Secure Adaptive Streaming, HLS, DASH, jwplayer, Ecnryption, AES-128]
title: "Digital Rights Management (multi - drm)"
comments: true
share: true
date: 2017-11-11T20:43:41+05:30
---

# what

> *Digital rights management (DRM) schemes are various access control technologies that are used to restrict usage of proprietary hardware and copyrighted works. DRM technologies try to control the use, modification, and distribution of copyrighted works (such as software and multimedia content), as well as systems within devices that enforce these policies* .For more details check [wiki](https://en.wikipedia.org/wiki/Digital_rights_management).

# History 
Remember product key (A typically alphanumerical serial number used to represent a license to a particular piece of software), it is one of the oldest and least complicated DRM protection methods for the computer games. 

In 1983, a very early implementation of Digital Rights Management (DRM) was the Software Service System (SSS) devised by the Japanese engineer Ryuichi Moriya. Today DRM has expanded to traditional hardware products, from John Deere's tractors, Philips' light bulbs and mobile device power chargers. You can find it applied in Computer Games by publishers like Electronic Arts, Ubisoft and The Sims3 and also in IBooks, Kindle etc which use DRM scheme of one or the other kind.  

# why 
Although, use of DRM is not universally accepted. Proponents of DRM argue that it is necessary to prevent intellectual property from being copied freely, just as physical locks are needed to prevent personal property from being stolen, that it can help the copyright holder maintain artistic control, and that it can ensure continued revenue streams. Those opposed to DRM contend there is no evidence that DRM helps prevent copyright infringement, arguing instead that it serves only to inconvenience legitimate customers, and that DRM helps big business stifle innovation and competition. Furthermore, works can become permanently inaccessible if the DRM scheme changes or if the service is discontinued. 

Irrespective of your view about the use of DRM if you are working in industries like entertainment and gaming which have been using DRM for a long you might still want to stick around to see what all the fuss it about and educate yourself. 

# State of DRM
The biggest hastle with DRM is the native support for different browsers on different devices. Check [this](https://drmtoday.com/platforms/) list to get an idea. So in order for you to provide your customers DRM service on most common browsers you will have to use more than one provider. A combination of Playread , Widevinew and Fairplay would cover for most part hence the term multi-DRM. 

For more geeky pals out there check the state of [encrypted media extension here](https://caniuse.com/#search=drm) where you will also find some more resources , do check the html5rocks article by Sam Dutton.

who(service providers) and some history 
* Widevine (google)
* Playready (microsoft)
* Fairplay (apple)

For this article we will focus on DASH first with widevine and playready. Fairplay uses HLS. [Groups lobbying for DRM](http://www.info-mech.com/drm_policy.html#mpaa)

# How
Before moving forward skim through [this Stackoverflow Question](https://stackoverflow.com/questions/822468/is-there-an-open-source-drm-solution) if you are wondering about open-source DRM solution. 

To make our life easy we will use [ezdrm](http://ezdrm.com) to help us with multi-drm. I recommend ezdrm because of the level of support you will receive. It took me some time to figure out small details as not much material was available online hence quick support becomes very important in such cases. 

As for this article we will use ffmpeg, Bento4 and Jwplayer to get the job done. You can also replace them with some alternatives after checking how the whole process works. So I would recommend to have some familarity with libraries like ffmpeg and bento4 before you move forward. DRM is a complex system with multiple moving parts so getting yourself familiar with ecosystem would go a long way.

* For more details on ffmpeg please check this post about [creating a cloud computing platform from scratch](http://aameer.github.io/cloud-computing-101/) or [ffmpeg docs](ffmpeg.org).
* For bento4 you can check [bento4 docs](http://bento4.com/).

Assuming familarized yourself with ffmpeg , you now know that ffmpeg can be used for various purposes. Here we will use it for say burning subtitles and creating four different versions for adaptive streaming.

# ffmpeg

{% highlight python linenos %}


ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:360,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf" ffmpeg_out_final/output_360.mp4

ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:480,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf" ffmpeg_out_final/output_480.mp4

ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:720,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf" ffmpeg_out_final/output_720.mp4

ffmpeg  -i input_file.mp4 -codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut" -profile:v baseline -level 4.0 -vf "scale=-2:1080,subtitles='/home/aameer/Documents/projects/subtitle/dynamic_subtitle_new.ass':force_style=FontName=/home/aameer/Documents/projects/font/Aaargh.ttf"ffmpeg_out_final/output_1080.mp4

{% endhighlight %}

added `-codec:v libx264 -x264opts "keyint=24:min-keyint=24:no-scenecut"` in above command because of [this issue](https://github.com/axiomatic-systems/Bento4/issues/85)

for creation of `dynamic_subtitle_new.ass check` the base article. Now we have 4 formats `output_1080`, `output_720`, `output_480`, `output_360`. Now comes the benot4 into play.

# Bento4 :
> *Bento4 is a C++ class library and tools designed to read and write ISO-MP4 files. This format is defined in international specifications ISO/IEC 14496-12, 14496-14 and 14496-15. The format is a derivative of the Apple Quicktime file format, so Bento4 can be used to read and write most Quicktime files as well. Visit www.bento4.com for details.* 

you can get the binaries [here](https://www.bento4.com/downloads/). Note you will have to use python2.7 for this. I was planing to update it for python3 but was advised againt by **Gilles Boccon** (creator of bento4) in the interest of time. Thanks for that tip Gilles!. 

Now using bento4 we will fragment the output videos which we got from ffmpeg , command would be like 

{% highlight python linenos %}

bin/mp4fragment ffmpeg_out_final/output_360.mp4 fragmented_output_final/frag_360.mp4
bin/mp4fragment ffmpeg_out_final/output_480.mp4 fragmented_output_final/frag_480.mp4
bin/mp4fragment ffmpeg_out_final/output_720.mp4 fragmented_output_final/frag_720.mp4
bin/mp4fragment ffmpeg_out_final/output_1080.mp4 fragmented_output_final/frag_1080.mp4

{% endhighlight %}

more about [mp4fragment](https://www.bento4.com/documentation/mp4fragment/)

now I am assuming by now you have got access to [ezdrm](http://www.ezdrm.com/). Once you have that you can use this python code to get parse values from xml response. You can skip this portion if you already have figured a way to get all the values which we will need from widevine, playready and fairplay. Moreover you might also want to save these values somewhere for future reference.

{% highlight python linenos %}

def get_ezdrm_values(content_id=None):
    """
    Returns Ezdrm values if content_id supplied or creates new ones.
    """
    #TODO: change these values to your values
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

{% endhighlight %}

# Streams
Now to do multi-drm we have to create two streams
* DASH : which works with Widevine and Playready more [here](https://www.bento4.com/developers/dash/encryption-and-drm/).
* HLS  : which works with Fairplay more [here](https://www.bento4.com/developers/hls/).

Ideally values from ezdrm should have be fine but I faced and issue from bento4 while creating DASH stream-
When I was using `'KeyIDHEX': '4d51279b-8886-51b2-a15f-ac8ddd0fe046'`. I was getting `error ERROR: Invalid argument format for --encryption-key option`. I checked the code and it was failing at line number 1043 check on [github](https://github.com/axiomatic-systems/Bento4/blob/master/Source/Python/utils/mp4-dash.py#L1083) i.e

{% highlight python linenos %}

    1042             if len(kid_hex) != 32:
    1043                 raise Exception('Invalid argument format for --encryption-key option')

{% endhighlight %}

so I changed the `'KeyIDHEX':'4d51279b888651b2a15fac8ddd0fe046'` by removing the dashes and I was able to proceed ahead.

So if the values for DRM are as under

{% highlight python linenos %}

#Dummy DRM values For Fairplay:

{'AssetID': '24d016eb-dcee-4197-be2f-f5b062f5c3b5', 'LicensesUrl': 'http://fps.ezdrm.com/api/licenses', 'KeyID': 'i71VddYtmfTF0I8ervjso5tfxLAjaUbL7KSIjvRGmBw=', 'KeyHex': '8BBD5575D62D99F4C5D08F1EAEF8ECA39B5FC4B0236946CBECA4888EF446981C', 'KeyUri': 'skd://fps.ezdrm.com/;24d016eb-dcee-4197-be2f-f5b062f5c3b5', 'SupportedFPSVersions': '1'}

{% endhighlight %}

{% highlight python linenos %}

#Dummy DRM values for Widevine and Playready:

{'KeyIDGUID': 'TVEnm4iGUbKhX6yN3Q/gRg==', 'Key': 'EJVl27J9nSb2vW+cadivsw==', 'PSSH': 'EhBNUSebiIZRsqFfrI3dD+BGGghtb3ZpZG9uZSIQBbN1Nh3nR0yYxl24xwDiwUjj3JWbBg==', 'ContentID': 'BbN1Nh3nR0yYxl24xwDiwQ==', 'LAURL': 'https://playready.ezdrm.com/cency/preauth.aspx?pX=6BDD75', 'KeyID': 'TVEnm4iGUbKhX6yN3Q/gRg==', 'KeyIDHEX': '4d51279b-8886-51b2-a15f-ac8ddd0fe046', 'ServerGet': 'request={"policy": "", "tracks": [ {"type": "SD"}], "content_id": "BbN1Nh3nR0yYxl24xwDiwQ=="}', 'Checksum': '/yYhM78wfEg=', 'ServerURL': 'https://widevine-dash.ezdrm.com/proxy?pX=5E6ACA', 'KeyHEX': '109565dbb27d9d26f6bd6f9c69d8afb3', 'ResponseRaw': '{"status":"OK","drm":[{"type":"WIDEVINE","system_id":"edef8ba979d64acea3c827dcd51d21ed"}],"tracks":[{"type":"SD","key_id":"TVEnm4iGUbKhX6yN3Q/gRg==","key":"EJVl27J9nSb2vW+cadivsw==","pssh":[{"drm_type":"WIDEVINE","data":"EhBNUSebiIZRsqFfrI3dD+BGGghtb3ZpZG9uZSIQBbN1Nh3nR0yYxl24xwDiwUjj3JWbBg=="}]}]}'}

{% endhighlight %}

Then the commands for creating multi-drm adaptive dash and hls stream are -

{% highlight python linenos %}
#using bento for dash
bin/mp4dash --encryption-key=4d51279b888651b2a15fac8ddd0fe046:109565dbb27d9d26f6bd6f9c69d8afb3 --playready-header=LA_URL:https://playready.ezdrm.com/cency/preauth.aspx?pX=6BDD75 --widevine-header=#EhBNUSebiIZRsqFfrI3dD+BGGghtb3ZpZG9uZSIQBbN1Nh3nR0yYxl24xwDiwUjj3JWbBg== fragmented_output_final/frag_360.mp4 fragmented_output_final/frag_480.mp4 fragmented_output_final/frag_720.mp4 fragmented_output_final/frag_1080.mp4 --output-dir="dash_drm_output" --mpd-name="final_master.mpd"

#using bento for hls 
./bin/mp4hls --output-single-file fragmented_output_final/frag_360.mp4 fragmented_output_final/frag_480.mp4 fragmented_output_final/frag_720.mp4 fragmented_output_final/frag_1080.mp4 --output-dir="hls_drm_output" --master-playlist-name="final_stream.m3u8" --output-single-file --encryption-mode=SAMPLE-AES --encryption-key=8BBD5575D62D99F4C5D08F1EAEF8ECA39B5FC4B0236946CBECA4888EF446981C --encryption-iv-mode=fps --encryption-key-format=com.apple.streamingkeydelivery --encryption-key-uri="skd://fps.ezdrm.com/;24d016eb-dcee-4197-be2f-f5b062f5c3b5" 

#notice the commas around --encryption-key-uri they are intensional

{% endhighlight %}

again this is to avoid a bento4 issue which doesn't add keys properly to stream and hence stream doesn't work.

now you should have a folder `dash_drm_output` and `hls_drm_output` which would have `final_stream.mpd` and `final_stream.m3u8` respectively in addition to some other files. You can host them on s3 now and serve them with **cloudfront**. Once you do that you can use **Jwplayer** to test your strem

Once you have hosted them you need a DRM enabled player to play it. In this case we used Jwplayer. Since it is DRM usual setup wont work for fairplay with ezdrm so we have to add some additional code to make it work which I am mentioning as under:

{% highlight javascript linenos %}

  //added these function from frontend for fairplay to work:
  extractContentId: function(initDataUri) {
    var uriParts = initDataUri.split('://', 1);
    var protocol = uriParts[0].slice(-3);
    uriParts = initDataUri.split(';', 2);
    var contentId = uriParts.length > 1 ? uriParts[1] : '';
    return protocol.toLowerCase() == 'skd' ? contentId : '';
  },
  licenseRequestMessage: function(message) {
    return new Blob([message], {type: 'application/octet-binary'});
  },
  extractKey: function(response) {
    return new Promise(function(resolve, reject) {
      var reader = new FileReader();
      reader.addEventListener('loadend', function() {
        resolve(new Uint8Array(reader.result));
      });
      reader.addEventListener('error', function() {
        reject(reader.error);
      });
      reader.readAsArrayBuffer(response);
    });
  }

{% endhighlight %}

# Closing Notes

I have added credits whereever applicable. In addition to that I would like to thank

*  **David Eisenbacher** (CEO and Co-Founder, EZDRM Inc) and others from ezdrm.
*  **Dave Otten** (CEO jwplayer) and others from Jwplayer for the help. 
*  **Gilles Boccon-Gibod** (creator of bento4) for his time and valuable inputs.
{: style="color:blue; font-size: 80%;"}

I havent tested this but in some cases you may need to run the below mentioned command for ffmpeg to work as mentioned in this article 
`sudo apt-get install libavcodec-extra-54`

Note for building ffmpeg which I have used check my stackoverflow [here](https://stackoverflow.com/questions/35597083/how-to-build-ffmpeg-with-burn-text-on-hls-output-while-maintaining-the-aspect-ra)

Few things to Keep in mind. ezdrm and Jwplayer used here are paid services. Also you might need to have apple developer registration to get FPS deployment package which is again a paid service and if you/your company doesnt have an apple developer account you will need to get DUNS number first to apply for one. For more information on DUNS check [this](https://en.wikipedia.org/wiki/Data_Universal_Numbering_System).

One more thing to notice is this setup doesnt cover IOS as fariplay supports IOS with only IOS sdk which is not covered here.

If you dont want to deal with all this complexity then you can also use services like [bitmovin](http://bitmovin.com/) where they will take care of everything for you. You just need to supply values and receive output streams.

Since this is a very complex system there is a chance I may have missed some details in that case please get in touch. There is lot more to DRM that just this but hope this post was worth your time and gave you a picture of DRM.

Thanks for your time.

Some important links which were helpful in this course

* [ezdrm site](http://www.ezdrm.com/)
* [ffmpeg docs](https://ffmpeg.org/)
* [bento4 github](https://github.com/axiomatic-systems/Bento4)
* [drm today platforms](https://drmtoday.com/platforms/)
* [Xin's blog](https://tdngan.wordpress.com/2016/11/17/how-to-encode-multi-bitrate-videos-in-mpeg-dash-for-mse-based-media-players/)
* [ffmpeg wiki](https://trac.ffmpeg.org/wiki/Encode/H.264)
* [stackoverflow,com](https://stackoverflow.com/questions/9764740/unknown-encoder-libx264/10027588#10027588)
* [superuser.com](https://superuser.com/questions/908280/what-is-the-correct-way-to-fix-keyframes-in-ffmpeg-for-dash/1098329#1098329)
