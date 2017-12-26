# Turn a collection of images into a video

Author: Robert Nikutta

Version: 2017-12-21

The images in this example are of JPG type

Adapted steps from https://dlo.me/archives/2015/07/26/making-a-time-lapse-using-ffmpeg-and-imagemagick/

## Resize and crop

```
cd pics
mkdir resized
for FILE in `ls *.JPG`; do mogrify -resize 1280x720^ -gravity center -crop 1280x720+0+0 +repage -write resized/$FILE $FILE; done
```

`center` crops the central 1280x720 pixels. `south` the bottom ones, `north` the upper ones etc. See http://www.imagemagick.org/Usage/crop/#crop_tile_centered for more.

## Make vid with ffmpeg:

```
cd resized/
ffmpeg -framerate 18 -pattern_type glob -i '*.JPG' -c:v libx264 -pix_fmt yuv420p video.mp4
```

`framerate` is in fps.