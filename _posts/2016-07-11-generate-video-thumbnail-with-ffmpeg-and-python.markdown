---
published: true
title: Generate video thumbnail with FFmpeg and Python
layout: post
---
For a project I was working on, I needed to take an uploaded video and generate a square-ish thumbnail from it. So far I've had a really good experience with GStreamer, but I decided to give FFmpeg a try.

The first step was to wrap the ffmpeg executable and make it callable from Python:

{% highlight python %}
import subprocess

def ffmpeg(*cmd):
	try:
		subprocess.check_output(['ffmpeg'] + cmd)
	except subprocess.CalledProcessError:
		return False

	return True

{% endhighlight %}

Now we can make the lengthy call to ffmpeg. FFmpeg has a neat expression language that can be used when configuring video filters. For whatever reason the commas must have a backslash prepended. The code below takes care of that so that the video filter configuration stays more readable and easier to change. Change 'black' below to change the padding color and '300' to change the thumbnail size.

{% highlight python %}

def make_thumb(video_filename)
	# pad with black if W<H, crop to center if W>H, rescale to 300x300 if greater
	
	ff_filters = (f.replace(',', '\\,') for f in (
		'pad=if(lt(iw,ih),ih,iw):ih:if(lt(iw,ih),(ih-iw)/2,0):0:black',
		'crop=ih:ih:(iw-ih)/2:0'
		'scale=if(lt(iw,300),iw,300):if(lt(iw,300),iw,300)',
	))
	ff_filterstr = ','.join(ff_filters)

	thumb_path = video_filename + '.thumb.jpg'
	ffmpeg('-y', '-vf', ff_filterstr,
		'-vframes', '1', thumb_path,
		'-i', video_filename)

{% endhighlight %}

As an alternative to the unwieldy FFmpeg expression syntax, one can take the output of `ffprobe` to get the width and height, and calculate the pad/crop/scale parameters in Python:

{% highlight python %}
import json

def ffprobe(filename):
    cmd = ['ffprobe', '-of', 'json',
		'-show_format', '-show_streams',
		'-loglevel', 'quiet', filename]

    try:
        result = subprocess.check_output(cmd)
    except subprocess.CalledProcessError:
        return None

    return json.loads(result.decode('utf-8'))
	
...

(w, h), = ((s['width'], s['height'])
			for s in ffprobe(filename)['streams']
			if s['codec_type'] == 'video')
	
{% endhighlight %}