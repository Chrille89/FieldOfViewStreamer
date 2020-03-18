# FieldOfViewStreamer

The FieldOfViewStreamer contains on a server side (FieldOfViewStreamer) and a client side (FieldOfViewStreamerClient).
The server side is a nodejs web socket and transform the tiled video to a pure dash stream based on the HEVC-Tiles-Merger and push the segments to the client.
If the field of view (FoV) changes on the client side, the server must load the actual visible tile in higher resolution.

To deploy the server you need the following steps.

## Install NodeJS

```
wget --no-check-certificate https://nodejs.org/dist/v6.10.2/node-v6.10.2-linux-x64.tar.xz
tar -xvf node-v6.10.2.tar.gz
```

Now you can modify the PATH-Variable to use Node:

```
nano ~/.profile
```

and add the following lines of Code:

```
export PATH="$PATH:$HOME/node-v6.10.2-linux-x64/bin"
```

Now check if node is successfully installed by typing:
```
node -v
```
Now you have installed Nodejs and NPM on your Linux-Machine.

## Start the Server

Install dependencies:

```
cd FieldOfViewStreamer
npm install
node mediaStreamer.js
```

How to create the HEVC content you can see [here](https://gitlab.fokus.fraunhofer.de/fame-wm/hevc-js-tiles-merger).

To download and merge the content you must set the location of the hevc content in the config.json file.

## Create Content

3 differents tools are used to generate the content:

1.ffmpeg to convert MP4 to YUV, https://www.ffmpeg.org/download.html
2.Kvazaar to convert YUV to HEVC tiled, https://github.com/ultravideo/kvazaar
3.MP4Box to convert the HEVC to MP4 and DASH it, https://gpac.wp.imt.fr/downloads/
4.Reintegrate Audio

**From MP4 to YUV**

```
ffmpeg -i my_video.mp4 hugefile.yuv
```

Generates a YUV from the input video. If needed, you can change here the resolution.
Note: YUV file is very large.

**From YUV to HEVC Tiled**

```
kvazaar -i hugefile.yuv --input-res=1920x1080 --input-fps FPS --bitrate=30000000 --tiles 3x3 -p <I-FRAME-PERIOD> --slices tiles --mv-constraint frametilemargin -o tiled.265 will create a tiled HEVC file. This one should be playable by ffplay
```

Note: use ffprobe on the base file to get the default bitrate. Then you can choose lower bitrates (to downgrade the video quality). Don't forget to put the input-res which is the resolution of the content.


**From HEVC to MP4**

```
mp4box -add tiled.265:split_tiles -new video_tiled.mp4 
```

creates a new MP4 file with 10 video tracks inside. This file is not playable.

**From MP4 to DASH**

Generate the DASH Content by typing:

```
mp4box -dash 1000 -profile live -out dash_tiled.mpd video_tiled.mp4 
```

Note: this DASH content can be play back using the MP4Client.

This DASH Process generate a MPD which is almost DASH Compliant (but the srd option is not well implemented yet). This MPD file is not used in our implementation. Instead, the content is described in media/media.json. See below.


**Reintegrate Audio**

```
ffmpeg -i my_video.mp4 my_video.aac
mp4box -add my_video.aac audio.mp3
mp4box -dash 1024 -profile live -out dash_tiled.mpd video_tiled.mp4
mp4box -add audio.mp3 video_tiled.mp4
mp4box -dash 1024 -profile live -out dash_tiled.mpd video_tiled.mp4

```

Now you have n+2 tracks. The last track is the audio track. If the audio track segments differ to the video segments you must change the dash duration. Use 1024 ms for each track because 1024 is the duration of a acc-coding audio frame.
You must use the correct i-frame period in the coding of the video. 
For example for the duration of each video segment of 1024 ms you need a i-frame period of ca. 29 frames. 

**Adding the new content in the media.json file**

The App loads the file media.json from the CDN-Host 'http://dash.fokus.fraunhofer.de/tests/cba/FoVStreamer/media/media.json'. 
You can directly edit this file if you have uploaded new media.

The media.json file looks like:

```json
{
    "contents": [
        //... your content here
    ]
}
```

Put your content as a JSON Object with the following properties:

```json
{
    "content_name":                    "Starwars: Hunting the Fallen",
    "content_url_scheme":              "http://dash.fokus.fraunhofer.de/tests/cba/FoVStreamer/media/starwars/starwars_3x3_dash_%bitrate%/video_tiled_dash_track%tile%_%segment%.m4s",
    "content_dash_init":               "http://dash.fokus.fraunhofer.de/tests/cba/FoVStreamer/media/starwars/starwars_3x3_dash_%bitrate%/dash_tiled_set1_init.mp4",
    "content_tracks"                    11,
    "content_segments":                 102,
    "content_base_track_number":        1,
    "content_bitrates":                 [100,1000,5000,10000,18826]
    "content_grid_size":                ["3x3"],
    "content_audio":                    true
}
```

**content_name:** is the name of the content.

**content_url_scheme and content_dash_init:** are urls to the content. You can use %bitrate% and %grid% values that will be replaced in the url according to the current grid and bitrates. %tile% and %segment% are numbers representing respectively the track number (1 should be the base track, the others video/audio tracks) and the segment number (should be 1 second each).

**content_segments:** is the number of available segments.

**content_base_track_number:** is the track number of the base track. If the content is generated using Kvazaar and MP4Box, it should be the track 1.

**content_bitrates and content_grid_size:** These values will be used to replace %bitrates% and %grid% is the url scheme.

**content_tracks** is the count of available tracks.

**content_audio** is set to true, if the video contains audio.

