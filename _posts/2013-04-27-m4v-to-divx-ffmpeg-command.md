---
title: m4v to DIVX ffmpeg command
---
`ffmpeg -i source.m4v -acodec libmp3lame -ab 128k -vcodec mpeg4 -qscale 4 -vtag DIVX destination.avi`