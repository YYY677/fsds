# ffmpeg and Rendering Pre-Recorded Lectures

## Exporting from Quarto

To create the outputs we need to export the Quarto presentations to PNG files. This can be done using [decktape](https://github.com/astefanutti/decktape), which is a node (hisssss!) application. To install:

```bash
npm install -g decktape
decktape
```

Then to extract:

```bash
export LECTURE='1.1-Getting_Oriented'
mkdir $LECTURE
decktape --screenshots --screenshots-directory $LECTURE --screenshots-size 1280x720 --screenshots-format png --headless true -p 500   http://localhost:4200/lectures/$LECTURE.html $LECTURE.pdf
```

## Using ffmpeg

### Orientation

[Programming Historian](https://programminghistorian.org/) has a useful [Introduction to ffmpeg](https://programminghistorian.org/en/lessons/introduction-to-ffmpeg#basic-structure-and-syntax-of-ffmpeg-commands). It doesn't cover this use case but it is nonetheless a nice orientation to the basic structure of ffmpeg commands. See also: [Demystifying ffmpeg](https://github.com/privatezero/NDSR/blob/master/Demystifying_FFmpeg_Slides.pdf), the [ffmpeg Presentation](https://docs.google.com/presentation/d/1NuusF948E6-gNTN04Lj0YHcVV9-30PTvkh_7mqyPPv4/present?ueb=true&slide=id.g2974defaca_0_231) and [ffmpeg Wiki](https://trac.ffmpeg.org/wiki/WikiStart) alongside [the documentation](https://www.ffmpeg.org/ffmpeg.html). 

### Creating the Intro

#### Creating a Background

See [this answer](https://stackoverflow.com/questions/36962458/making-a-green-screen-background-in-ffmpeg) on Stack Overflow:

```bash
export INTROTIME=15
export FRAMERATE=30
export FRAMESIZE="1280x720"
ffmpeg -f lavfi -i color=color=white:s=$FRAMESIZE -t $INTROTIME -r $FRAMERATE intro_bg.mp4
```

#### Adding Text/Image Overlays

To add a copyright/datestamp at the start and/or end [this StackOverflow thread](https://stackoverflow.com/questions/17623676/text-on-video-ffmpeg) contains a lot of useful information.

#### Text Position

```bash
ffplay -vf "drawtext=font='Amethyst':text='Stack Overflow':fontcolor=white:fontsize=44:box=1:boxcolor=black@0.5:boxborderw=5:x=(w-text_w)/2:y=(h-text_h)/2" intro_bg.mp4
```

This next one positions the text bottom-right:

```bash
ffplay -vf "drawtext=text='Jon Reades':font='Times New Roman':x=(main_w-text_w-10):y=(main_h-text_h-10):fontsize=62:fontcolor=black:box=1:boxcolor=red@0.5:boxborderw=5" intro_bg.mp4
```

#### Adding Time

Now let's bring it in between 2 and 5 seconds:

```bash
ffplay -vf "drawtext=font='Amethyst':text='Stack Overflow':fontcolor=white:fontsize=44:box=1:boxcolor=black@0.2:boxborderw=5:x=(w-text_w)/2:y=(h-text_h)/2:enable='between(t,2,5)" intro_bg.mp4
```

Fading text in is much more complex, to the point [where there is a web site to help](http://ffmpeg.shanewhite.co/):

```bash
export TEXT="CASA"
export STARTIN=0.25
export FADEIN=1.5
export FADEOUT=0.75
export SHOWLEN=3+$STARTIN+$FADEIN
export STOPAT=$SHOWLEN+$FADEOUT
ffmpeg -i intro_bg.mp4 -filter_complex "[0:v]drawtext=font='Amethyst':text='$TEXT':fontcolor=black:fontsize=64:x=(w-text_w)/2:y=(h-text_h)/2:alpha='if(lt(t,$(($STARTIN))),0,if(lt(t,$(($STARTIN+$FADEIN))),(t-$(($STARTIN)))/$(($FADEIN)),if(lt(t,$(($SHOWLEN))),1,if(lt(t,$(($STOPAT))),($(($FADEOUT))-(t-$(($SHOWLEN))))/$(($FADEOUT)),0))))'" out.mp4 # -pix_fmt=yuv420p 
```

#### Fading in a Logo

Here's a [relevant answer](https://stackoverflow.com/questions/50515822/ffmpeg-overlay-image-with-fade-in-and-fade-out) from Stack Overflow:

```bash
# Works, no fading
ffmpeg -i intro_bg.mp4 -i CASA_logo.png -filter_complex "[0:v][1:v] overlay=W-w:H-h" -pix_fmt yuv420p intro.mp4

# Works, with fade-in
ffmpeg -i intro_bg.mp4 -loop 1 -i CASA_logo.png -filter_complex "[1]fade=in:st=0:d=1:alpha=1[logo]; [0][logo]overlay=W-w:H-h:shortest=1" -c:v libx264 intro.mp4

# Works, with fade-in and fade-out
ffmpeg -i intro_bg.mp4 -loop 1 -i CASA_logo.png -filter_complex "[1:v]fade=in:st=0.25:d=2:alpha=1,fade=out:st=4:d=1:alpha=1 [logo]; [0:v][logo] overlay=W-w:H-h:shortest=1" -c:v libx264 -pix_fmt yuv420p intro.mp4

# Works, with fade-in and fade-out 
# and correct positioning (horizontally-centred 
# and 4/5ths of way down)
ffmpeg -i intro_bg.mp4 -loop 1 -i CASA_logo.png -filter_complex "[1:v]fade=in:st=0.25:d=2:alpha=1,fade=out:st=4:d=1:alpha=1 [logo]; [0:v][logo] overlay=x=main_w/2-overlay_w/2:y=main_h*4/5-overlay_h/2:shortest=1" -c:v libx264 -pix_fmt yuv420p intro.mp4
```

Integrating that with the above:

```bash
export TEXT="CASA"
export STARTIN=0.25
export FADEIN=1.5
export FADEOUT=0.75
export SHOWLEN=3+$STARTIN+$FADEIN
export STOPAT=$SHOWLEN+$FADEOUT
ffmpeg -i intro_bg.mp4 -loop 1 -i CASA_logo.png -filter_complex " [1:v]fade=in:st=$(($STARTIN)):d=$(($FADEIN)):alpha=1,fade=out:st=$(($SHOWLEN)):d=$(($FADEOUT)):alpha=1[logo]; \
  [0][logo] overlay=x=main_w/2-overlay_w/2:y=main_h*4/5-overlay_h/2:shortest=1, \
  drawtext=font='Amethyst':text='$TEXT':fontcolor=black:fontsize=64:\
  x=(w-text_w)/2:y=(h-text_h)/2:\
  alpha='if(lt(t,$(($STARTIN))),0,if(lt(t,$(($STARTIN+$FADEIN))),(t-$(($STARTIN)))/$(($FADEIN)),if(lt(t,$(($SHOWLEN))),1,if(lt(t,$(($STOPAT))),($(($FADEOUT))-(t-$(($SHOWLEN))))/$(($FADEOUT)),0))))'" -c:v libx264 -pix_fmt yuv420p  intro.mp4
```

### Finalising the Intro

By default we're using [Barlow](https://fonts.google.com/specimen/Barlow/) for the title of the video since that's what we use in Quarto. And note that here we've managed to set up the background colour on the fly so now there's no need for `intro_bg.mp4`:

```bash
export YEARTEXT="2023/24"
export COURSETEXT1="Foundations of"
export COURSETEXT2="Spatial Data Science"
export FONT="Barlow"
export FONTCOLOR="4f3d57"
export STARTIN=0.15
export FADEIN=0.6
export FADEOUT=0.6
export SHOWLEN=0.75+$STARTIN+$FADEIN
export STOPAT=$SHOWLEN+$FADEOUT
export INTROLEN=$STOPAT+0.05
export FRAMERATE=30
export FRAMESIZE="1280x720"

ffmpeg -loop 1 -i CASA_logo.png -t $(($STOPAT)) -filter_complex "\
  color=white:s=$FRAMESIZE [bg]; \
  [0:v]fade=in:st=$(($STARTIN)):d=$(($FADEIN)):alpha=1,  fade=out:st=$(($SHOWLEN)):d=$(($FADEOUT)):alpha=1 [logo]; \
  [bg][logo] overlay=x=main_w/2-overlay_w/2:y=main_h/2+overlay_h*0.125:shortest=1, \
  drawtext=font='$FONT':text='$YEARTEXT':fontcolor=$FONTCOLOR:fontsize=24:x=(w-text_w)/2:y=(h-text_h)/2:alpha='if(lt(t,$(($STARTIN))),0,if(lt(t,$(($STARTIN+$FADEIN))),(t-$(($STARTIN)))/$(($FADEIN)),if(lt(t,$(($SHOWLEN))),1,if(lt(t,$(($STOPAT))),($(($FADEOUT))-(t-$(($SHOWLEN))))/$(($FADEOUT)),0))))', \
  drawtext=font='$FONT':text='$COURSETEXT1':fontcolor=$FONTCOLOR:fontsize=48:x=(w-text_w)/2:y=(h/3-text_h/1.5):alpha='if(lt(t,$(($STARTIN))),0,if(lt(t,$(($STARTIN+$FADEIN))),(t-$(($STARTIN)))/$(($FADEIN)),if(lt(t,$(($SHOWLEN))),1,if(lt(t,$(($STOPAT))),($(($FADEOUT))-(t-$(($SHOWLEN))))/$(($FADEOUT)),0))))', \
  drawtext=font='$FONT':text='$COURSETEXT2':fontcolor=$FONTCOLOR:fontsize=48:x=(w-text_w)/2:y=(h/3+text_h/1.5):alpha='if(lt(t,$(($STARTIN))),0,if(lt(t,$(($STARTIN+$FADEIN))),(t-$(($STARTIN)))/$(($FADEIN)),if(lt(t,$(($SHOWLEN))),1,if(lt(t,$(($STOPAT))),($(($FADEOUT))-(t-$(($SHOWLEN))))/$(($FADEOUT)),0))))'" \
  -c:v libx264 -pix_fmt yuv420p -tune stillimage intro.mp4
```

### Generating the Movie

For my particular application there is a useful example of generating a _very_ high-quality output (i.e. very large file) from static input images on [StackOverflow](https://stackoverflow.com/a/73073276/4041902); however, note that the output isn't widely compatible either so it's more about using this is a framework for thinking about the parameters and options.

#### Base Case (1 Output)

This assembles separate audio and video tracks into one output:

```bash
ffmpeg -r 30 \
  -loop 1 -i 2.3-Python_the_Basics_1_1280x720.png \
  -i 2.3-1.m4a \
	-c:v libx264 -tune stillimage -shortest -pix_fmt yuv420p \
	-b:a 64k \
	out1.mp4

ffmpeg -r 30 \
  -loop 1 -i 2.3-Python_the_Basics_2_1280x720.png \
  -i 2.3-2.m4a \
	-c:v libx264 -tune stillimage -shortest -pix_fmt yuv420p \
	-b:a 64k \
	out2.mp4

ffmpeg -r 30 \
  -loop 1 -i 2.3-Python_the_Basics_3_1280x720.png \
  -i 2.3-3.m4a \
	-c:v libx264 -tune stillimage -shortest -pix_fmt yuv420p \
	-b:a 64k \
	out3.mp4
```

#### Combining Multiple Outputs

We need to repeat the above a+v merge for *every* slide in the deck and *then* merge all of the output videos together into one long video. This approach does *not* re-encode the data so it’s probably faster and less prone to creating artefacts; however it also generates errors and seems to leave a black frame at the start.

```bash
$ cat list.txt
file 'out1.mp4'
file 'out2.mp4'
file 'out3.mp4'
ffmpeg -f concat -safe 0 -i list.txt -c copy out4.mp4
```

The black frame at the start can be trimmed as follows:

```bash
ffmpeg -ss 00:00:01.250 -i out4.mp4 -t 10:00:00 -c:v copy -c:a copy out5.mp4
```

And it looks like I could do the same [at the end](https://shotstack.io/learn/use-ffmpeg-to-trim-video/) using negative numbers.

From [Stack Overflow](https://stackoverflow.com/a/53857401/1600439): 

```bash
ffmpeg -i input-a.mp4 -i input-a.mp3 -i input-b.mp4 -i input-b.mp3 -filter_complex "[0:v][1:a][2:v][3:a]concat=n=2:v=1:a=1[v][a]" -map "[v]" -map "[a]" output.mp4
```

But see [this example](https://trac.ffmpeg.org/wiki/Concatenate) (which makes a bit more sense).

#### The Cleaner Approach

This code runs, but it never completes because the loops never end on the PNG files:

```bash
ffmpeg -r 30 \
  -loop 1 -i 2.3-Python_the_Basics_1_1280x720.png \
  -loop 1 -i 2.3-Python_the_Basics_2_1280x720.png \
  -loop 1 -i 2.3-Python_the_Basics_3_1280x720.png \
  -i 2.3-1.m4a \
  -i 2.3-2.m4a \
  -i 2.3-3.m4a \
  -filter_complex "[0:v][3:a] [1:v][4:a] [2:v][5:a] concat=n=3:v=1:a=1 [vvv][aaa]" \
  -map "[vvv]" -map "[aaa]" \
  -c:v libx264 -tune stillimage -pix_fmt yuv420p \
  -c:a aac -b:a 64k \
  output.mp4
```

I *could* address this by setting `-t` on every PNG file. For that to work I'd need something like this to get the duration of the m4a file in seconds:

```bash
ffprobe -sexagesimal -show_entries format=duration 2.3-1.m4a
```

This produces:

```
[FORMAT]
duration=0:00:41.937000
[/FORMAT]
```

### Previous Approaches

```bash
ffmpeg \
  -r 30 -t 50 -loop 1 -i 1.1-Getting_Oriented_19_1280x720.png \
  -r 30 -t 30 -loop 1 -i 1.1-Getting_Oriented_20_1280x720.png \
  -i test.m4a -i test2.m4a \
  -filter_complex "[0:v:0][2:a:0][1:v:0][3:a:0] concat=n=2:v=1:a=1[outv][outa]" \
  -map "[outv]" -map "[outa]" \
  -c:v libx264 -tune stillimage -preset ultrafast -pix_fmt yuv420p \
  -c:a aac -b:a 64k \
  out7.mp4
```

**Something** happening here and in other ones where I try to merge PNG images into one video: the preview goes black and if you fast forward or rewind in that area you get no picture. I think this problem disappears if the `-t` option is longer than the audio file for input>0. But then you get video with no audio past the point where the recording stopped. So you kind of need to set the time to the *exact* length of the related audio file.

### Adding Filters

I think I’ll need to this later: [see docs](https://trac.ffmpeg.org/wiki/FilteringGuide).

Ooooh, and a [cheatsheet](https://gist.github.com/martinruenz/537b6b2d3b1f818d500099dde0a38c5f)

Consider also adding filters on audio and video channeles:

- atadenoise denoising [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-atadenoise)
- blend for blending one layer into another (watermarking?) [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-blend-1)
- chromakey for green-screening [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-blend-1) (also has useful thing for overlaying on a static black background)
- colorize is  [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-colorize)
- colortemperature is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-colortemperature)
- coreimage to make use of Apple's CoreImage API [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-coreimage-1)
- crop is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-crop)
- dblur for directional blur could be fun on intro/outro [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-dblur)
- decimate is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-decimate-1)
- displace (probably a bad idea but...) is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-displace)
- drawtext (for writing in date/year) is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-drawtext-1) (requires libfreetype)
- fade (to fade-in/out the input video) is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-fade)
- frames per second is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-fps-1) not sure how it differs from [framerate](https://www.ffmpeg.org/ffmpeg-filters.html#toc-framerate)
- Gaussian blur is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-gblur)
- Hue/saturation/intensity is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-huesaturation)
- Colour adjustment is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-Color-adjustment)
- Loop is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-loop)
- Monochrome is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-monochrome) and could be used with colourisation on live video, for instance
- Normalise is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-normalize) for mapping input histogram on to output range
- Overlay is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-overlay-1) and will be useful for adding an intro/outro
- Perspective correction i s[here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-perspective)
- Scale is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-scale-1) to rescale inputs.
- Trim is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-trim)
- Variable blur i s[here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-varblur) and could be useful for background blurring behind a talking head.
- Vibrance to increase/change saturation is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-vibrance)
- Vstack as faster alternative to Overlay and Pad is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-vstack)
- Xfade to perform cross-fading between input streams is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-xfade)
- Zoom and pan is [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-zoompan)

Currently available video sources are [here](https://www.ffmpeg.org/ffmpeg-filters.html#toc-Video-Sources)

<a rel="me" href="https://mapstodon.space/@jreades">Mastodon</a>
