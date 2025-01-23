#### 下载 ####

```shell
git clone https://git.ffmpeg.org/ffmpeg.git

#编译
./configure

#make
sudo make && sudo make install

#环境变量
export PATH=$PATH:/usr/local/ffmpeg/bin
```

* 编译成功后inclode 头文件在 ```/usr/local/include```
* Ffmpeg 相关静态库在```/usr/local/lib```

#### 几个例子 ####

**list功能**

```c
#include <libavutil/log.h>
#include <libavformat/avformat.h>

int main(int argc, char const *argv[])
{

    int ret;
    AVIODirContext *ctx = NULL;
    AVIODirEntry *entry = NULL;
    av_log_set_level(AV_LOG_INFO);

    ret = avio_open_dir(&ctx, "./", NULL); //打开目录
    if (ret < 0)
    {
        av_log(NULL, AV_LOG_ERROR, "Can't Open dir:%s\n", av_err2str(ret));
        return -1;
    }
    while (1)
    {
        ret = avio_read_dir(ctx, &entry);
        if (ret < 0)
        {
            av_log(NULL, AV_LOG_ERROR, "Can't Read dir: %s\n", av_err2str(ret));
            goto __fail;
        }
        if (!entry) // 读完了
        {
            break;
        }
        av_log(NULL, AV_LOG_INFO, "%12" PRId64 " %s \n",
               entry->size,
               entry->name);
        avio_free_directory_entry(&entry);
    }
__fail:
    avio_close_dir(&ctx);

    return 0;
}

```

运行：

```shell
clang -g -o list ffmpeg_list.c `pkg-config --libs libavformat libavutil`
```



```c

//ic	media file handle
//type	stream type: video, audio, subtitles, etc.
//wanted_stream_nb	user-requested stream number, or -1 for automatic selection
//related_stream	try to find a stream related (eg. in the same program) to this one, or -1 if none
//decoder_ret	if non-NULL, returns the decoder for the selected stream
//flags	flags; none are currently defined
int av_find_best_stream	(	AVFormatContext *ic,
	enum AVMediaType 	type,
	int wanted_stream_nb,
	int related_stream,
	const struct AVCodec **decoder_ret,
	int flags 
)	
```





