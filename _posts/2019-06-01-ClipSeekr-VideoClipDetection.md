---
layout: post
title: "ClipSeekr&trade;: Video Clip Recognition System"
description: Video Shot Detection
author: starkdg
date: 2019-06-01
custom-javascript-list: ["https://ajax.googleapis.com/ajax/libs/jquery/3.4.0/jquery.min.js","/assets/js/viewhtml.js"]
use_js: true
use_math: true
tags: [video, c++, video-recognition]
---

ClipSeekr is a real-time video clip recognition system  
designed to detect video sequences that occur in a video  
stream.
<!--more-->

## How It Works

Clipseekr works by indexing fingerprints of video clips.  
A 64-bit fingerprint is created for each frame of the clip  
from the spatial frequency information extracted from its  
discrete cosine transform.  These 64-bit integers are then  
stored in a reverse index.  This reverse index is simply a  
redis database of key-value pairs, where the key is a frame's  
fingerprint pointing to a value consisting of an ID and some  
sequence information. Unknown streams can then be monitored  
to recognized the appearance of these indexed clips.  The  
basic principle is simple.  When the number of consecutive  
frames recognized for a particular ID reaches a specified  
threshold, the clip can then be identified together with its  
timestamp in the stream.  This threshold is adjustable, but  
a good value for a 29.97 fps stream seems to be between 5  
and 10 consecutive frames.

## Code

The code can be found in the github repository here:  

[ClipSeekr](https://github.com/starkdg/clipseekr)

## Test Results

To evaluate this method, we streamed four hours of television  
and copied the commercial spots into new files for indexing.  
Altogether, there were 142 of these ad spots, 135 of which  
being unique video sequences.  In brief, only one spot failed  
to be detected outright - i.e. a "false negative" - while five  
were detected falsely - i.e. "false positives".  The rest were  
successfuly detected within seconds of the occurence in the stream.  
This would roughly make for a false posive rate of 3.3%, and a  
false negative rate of 0.01%.  The following table logs the  
results more precisely.  The first two columns mark the clips  
and the timestamps for where they actually occur in the stream.  
The next two columns indicate the clips that get recognized  
along with their timestamps.


A black font represents correct detections; a red font  
represents false positives; and blue is for false negatives.

<div class="viewport" id="includedContent"></div>
\\
\\
\

The only one that failed to be detected was a McDonald's  
commercial, called "Uber Eats".  The only thing noteworthy  
is that the frames seemed exceptionally dark in contrast.  
Perhaps not enough definition in the fingerprints.  Another  
noteworthy issue is the second detection of the spot called  
"Jack Daniels".  While the first one was a correct match,  
the second detection, even though it was a different clip,  
it shared enough of the first clip in common that the second was  
recognized as the first.  This is an inherent weakness in the  
fingerprinting system, since there is not enough temporal  
information preserved to differentiate the two in real-time.

## A few notes for further study:

* While the fingerprinting method is fairly robust to many  
distortions, it is not robust to changes in the screen format.  
In other words, many broadcast streams manipulate the screen  
format to include varying amounts of black space in the margins.  
Also, the presence of various logos and other textual occlusions  
further obfuscate the spatial information of the frames.  
Alternative fingerprinting methods can be explored for this:  
scale-invariant feature points, or feature points combined with  
region-based descriptors.

* The limited temporal information restricts the ability of the  
system to differentiate between clips that share a significant  
portion of frames in common.  In other words, two commercial  
spots are often composed from common sequences only edited  
differently. Unfortunately, the real-time nature of the problem  
prohibits a second pass of the data. Recognition decisions are  
constrained to only looking at past frames.

* Given the success of convolutional neural nets for image  
recognition tasks, it would be interesting to add in a  
recurrence property to better model a sequence of frames.  
Previous work in extracting image fingerprints from convolutional  
network models shows promise in differentiating images:

[pyConvnetPhash](https://blog.phash.org/posts/concise-image-descriptor).  

However, this is an extremely slow approach when dealing with 30  
fps streams. Recurrent Neural Networks could possibly add in  
some temporary information, but might be limited to video clips  
of a fixed set length.

Thanks you for your time in reading this post.  
Comments and suggestions are welcome.

[Comments and Suggestions](https://github.com/starkdg/clipseekr/issues)

