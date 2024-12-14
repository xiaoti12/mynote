## linux安装预编译版本
下载解压文件
```bash
cd /usr/local
wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz
tar -xf ffmpeg-release-amd64-static.tar.xz
mv ffmpeg-xxx-amd64-static ffmpeg
```
设置环境变量
```bash
echo "export PATH=$PATH:/usr/local/ffmpeg" >> ~/.zshrc
```
## 使用
**分离音视频**：
`ffmpeg -i input_video.mp4 -vn -c:a copy output_audio.aac`

**ts视频转换为mp4**（不重新编码）：
`ffmpeg -i input_video.ts -c:v copy -c:a copy output_video.mp4`

**视频合并**：
创建文件merge.txt，运行命令`ffmpeg -f concat -i merge.txt -c copy output.mp4`

```
file 'xxx.mp4'
file 'xxx.mp4'
```

**音视频合并**：
`ffmpeg -i xxx.mp4 -i xxx.mp3 -c copy output.mp4`
