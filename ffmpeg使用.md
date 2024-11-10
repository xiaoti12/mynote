分离音视频：

`ffmpeg -i input_video.mp4 -vn -c:a copy output_audio.aac`

ts视频转换为mp4（不重新编码）：

`ffmpeg -i input_video.ts -c:v copy -c:a copy output_video.mp4`

视频合并：

创建文件merge.txt，运行命令`ffmpeg -f concat -i merge.txt -c copy output.mp4`

```
file 'xxx.mp4'
file 'xxx.mp4'
```

音视频合并：

`ffmpeg -i xxx.mp4 -i xxx.mp3 -c copy output.mp4`
