# Mapbox: Living Weather Data on Web Maps 
##### 
By Damon Burgett, Geographer, Mapbox and Charlie Lloyd, Mapbox

##### *Part 2: Using Mapbox and Open source tools to animate map atmospheric water* 
By [Damon Burgett](https://www.mapbox.com/about/team/#damon-burgett), Geographer, Mapbox and Charlie Lloyd, Mapbox

*As part of the [Commerce Data Usability Project](https://www.commerce.gov/datausability/),  Mapbox in collaboration with the [Commerce Data Service](https://www.commerce.gov/dataservice/) has created a two part tutorial that will guide you though processing and visualizating precipitable water data from NOAA.  If you have question, feel free to reach out to the Commerce Data Service at data@doc.gov or Mapbox at help@mapbox.com.*


#### Atmospheric data is a digital representation of the living environment.

In this second tutorial in a two part series working with National Oceanic + Atmospheric Administration (NOAA) Precipitable Water (Pwat) data, we extend the data processing and visualization demonstrated in Part One and an animated interactive map of global Pwat over 12 days. In this segment, we’ll be repeating the steps from [Part One](http://commercedataservice.github.io/tutorial_mapbox_part1/), but in bulk and in parallel on over 100 datasets. At the end of this tutorial, we’ll have an animated, interactive map of global pwat over 12 days.


![full](https://cloud.githubusercontent.com/assets/5084513/13926440/9296d324-ef49-11e5-9819-425eb184098f.gif)

## Getting Started

To get started quickly, the code for this tutorial can be found at the following [Github repo]([https://github.com/CommerceDataService/tutorial_mapbox_part2]). 

### Step 1: Preliminaries

#### Libraries and Utilities
We'll be using the following tools to wrangle the weather model output files:

   - **[GDAL](http://www.gdal.org/)**: Translator library for raster and vector geospatial data formats;
   - **[parallel](http://www.gnu.org/software/parallel/)**: Shell tool for executing jobs in parallel;
   - **[rasterio](https://github.com/mapbox/rasterio)**: Clean and fast and geospatial raster I/O for Python programmers who use Numpy;
   - **[gribdoctor](https://github.com/mapbox/grib-doctor)**: Utilities for handling quirks of weather data General Regularly-distributed Information in Binary form (grib) files. grib data is a concise data format commonly used in meteorology to store historical and forecast weather data which can looked at using software applications; and
   - **[ffmpeg](https://www.ffmpeg.org/)**: Library to record, convert and stream audio and video

#### Folder Setup

Let's first setup a few folders to store intermediate files. In your working directory, make these 6 files - keeping it organized as such will keep things simpler down the line:

```
working directory
 +-data
 +-global
 +-clip
 +-mercator
 +-pngs
 +-output
```

As a command (from within your working directory):

```
mkdir data global clip mercator color png output 
```

### Step 2: Get the data

From part one of our tutorial, we have the following url:

```
http://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl?file=gfs.t12z.pgrb2.0p25.f000&all_lev=on&var_PWAT=on&leftlon=0&rightlon=360&toplat=90&bottomlat=-90&dir=%2Fgfs.2016011212
```

This links to the Global Forecast System (GFS) model of Pwat on 01/12/2016 at 12 GST, for a forecast of 0 hours. The GFS forcast dataset is created every six hours, with predictions produced from each model every 3 hours. By combining every six-hour model run's hour 0 and hour 3 prediction data, we'll be able to download data at a 3-hour interval.

[GNU Parallel](http://www.gnu.org/software/parallel/) offers an efficient approach to abstract away a lot of the complication of iterating over numerous files. In particular, `parallel`'s abstraction of number expansion makes this manageable. Taking the above URL and replacing all the variables we want to iterate over, we have the following:
 
```
http://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl?file=gfs.t{time}z.pgrb2.0p25.f{time}&all_lev=on&var_PWAT=on&leftlon=0&rightlon=360&toplat=90&bottomlat=-90&dir=%2Fgfs.201603{date}{time}
```
Iterations are determined by the following three variables:

- `{date}` = Prediction date. We want 12 days, from the to 5th to the 17th, as two-digit numbers `{05..17}`

- `{hour}` = Prediction hour. We want 0, 6, 12, and 18 hours (every 6 hours), as two digit numbers `{00..18..6}`

- `{time}` = Prediction time. We want the 0 and 3 hour predictions, as three-digit numbers `{000..003..3}`

Let's put it all together to see how it works:
```
parallel --header : echo {date}-{hour}-{off} ::: off {000..003..3} ::: date {05..17} ::: hour {00..18..6}
```
This should create 104 combinations of our input ranges in no particular order. With the URL, we can simply put these variables in place of the static numbers we see in the download link as in the example above.

Here, we use `wget`, a command to download data from a URL. Plugging it into `parallel` means that for every combination of numbers we input, we'll perform `wget`. Let's first "dryrun" (`--dryrun`) to see the URLs we are creating:
```
parallel --dryrun --header : wget '"http://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl?file=gfs.t{time}z.pgrb2.0p25.f{time}&all_lev=on&var_PWAT=on&leftlon=0&rightlon=360&toplat=90&bottomlat=-90&dir=%2Fgfs.201603{date}{time}"' ::: date {05..17} ::: time {00..18..06} ::: hour {000..003..003}
```
This will print the URLs we'll want - grab one and paste it into a browser to see if it works. It should start downloading a *grib2* file. If not, double check your syntax. Keep an eye out for the URL string formatting; various shells handle these characters differently.

To keep the gribs in sorted order for the rest of the process, use the date-hour prediction number as the filename. This will ensure that we can simple re-use the iteration numbers as the output, saving in our `raw` folder as `data/{date}{time}{hour}.grib2`. Add this, remove the `--dryrun` flag, and re-run.
```
 parallel --header : wget '"http://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl?file=gfs.t{time}z.pgrb2.0p25.f{time}&all_lev=on&var_PWAT=on&leftlon=0&rightlon=360&toplat=90&bottomlat=-90&dir=%2Fgfs.201603{date}{time}"' -O data/{date}{time}{hour}.grib2 ::: date {05..17} ::: time {00..18..06} ::: hour {000..003..003}
```
If this worked, you'll have a directory of numerically sorted grib files! Open any of these freshly downloaded raster files in a desktop GIS system such as [QGIS](http://www.qgis.org/) and you should see the following:

![image](https://cloud.githubusercontent.com/assets/5084513/12276750/5fc8b3ca-b92c-11e5-8332-9460e5655074.png)

### Part 3: Geoprocessing

We are going to repeat each step that we referenced in Part 1, this time in parallel on our 104 input files. Luckily, `parallel` again handles much of the complication for us. A quick introduction to working with files in parallel. To list each file in our data directory with the `.grib2` extension, we can run the following command:

```
parallel echo {} ::: data/*.grib2
```

This should list out all the `grib2`s in your data directory. To list just the base filename:

```
parallel echo {/} ::: data/*.grib2
```

And to list just the filename without the file extension:
```
parallel echo {/.} ::: data/*.grib2
```

We'll use these three abstractions to process our data.

#### Converting to global grib using `gribdoctor`

A quick refresh on usage of this tool (see Part 1 for details):

```
gribdoctor smoosh -dev -uw <input>.grib2 <output>.tif
```
We want to take each `.grib2` in our `/data` directory, perform this operation, and output a `.tif` file of the same name in our `/global` directory. Here is the command:

```
parallel gribdoctor smoosh -dev -uw {} global/{/.}.tif ::: data/*.grib2
```

To deconstruct the above:

- `gribdoctor smoosh` is the command used to undertake geoprocessing; 
- `-dev -uw` are input options for `gribdoctor`
- `data/*.grib2` is the input directory. Each file in this directory will run in `parallel` as `{}`; and
- `global/{/.}.tif` is the output filename (which is each input base filename without the extension - after which we add on the `/global` directory and the `.tif` extension.

After running `gribdoctor`, you should have a set files that look like the following (open them in a GIS or other `GeoTIFF` viewer):

![image](https://cloud.githubusercontent.com/assets/5084513/12286506/6bae416c-b979-11e5-8eeb-31fa137ca8b1.png)

#### Clip to reasonable latitudes

We'll be adding this data to a web map in web mercator, which will overly stretch the higher latitudes to take up the bulk of our video size. To make the presentation neater, let's trim off the data at extreme latitudes using `rio clip`, a [rasterio](https://github.com/mapbox/rasterio) command.

We want to clip to West: -180.0, South: -70.0, East: 180.0, North: 70.0. Here again, is the command in `parallel`:

```
parallel rio clip {} clip/{/} --bounds -180.0 -70.0 180.0 70.0  ::: global/*.tif
```

Note that this time, we can simply use `{/}` as the input and outputs are both `.tif`s. Your `clip/` directory should have images like this:

![image](https://cloud.githubusercontent.com/assets/5084513/12313441/482599ea-ba1c-11e5-818f-7fa095fde032.png)

#### Warp to Web Mercator

Mapbox GL JS utilizes the web mercator projection. In order to integrate the PWAT imagery with street and terrain data, we will need to warp our input data into this projection. Let's also take this opportunity to make sure our image dimensions are divisible by two, which is neccesary for properly encoding into video.

The dimensions of your input data _after_ being warped to web mercator should be ~ 2705 x 1494. This gives us an aspect ratio of:

```
2705 / 1495 = ~1.8:1
```

Let's say we want our eventual output video to be 2048 pixels wide:

```
2048 * (1 / 1.8) = 1138px
```
Giving us desired output dimensions of 2048 x 1138. We then specify this in the `gdalwarp` command as `-ts`. Here it is in `parallel`:

```
parallel gdalwarp {} mercator/{/} -t_srs EPSG:3857 -co COMPRESS=DEFLATE -r bilinear -ts 2048 1138 ::: clip/*.tif
```
Here's an example output file:

![image](https://cloud.githubusercontent.com/assets/5084513/12313499/cd5c8de4-ba1c-11e5-8ddc-dc884dd94291.png)


#### Colorize PWAT

Let's colorize our data. Take the color ramp that you created in the first tutorial, and apply it on all of the warped files:

```
parallel gdaldem color-relief {} color-ramp.txt color/{/} ::: mercator/*.tif
```
Now, you should essentially have a set of `tifs` similar to what we'll have in our final product. Take the time now to tweak your color ramp as you see fit.

![image](https://cloud.githubusercontent.com/assets/5084513/12313587/ac86d808-ba1d-11e5-9405-e1597f9db8a6.png)


#### Convert to png

In order to use this in an animation, we need to convert them into a format that `ffmpeg` can recognize. In this case, we'll use `png` - here is the command:

```
parallel rio convert {} pngs/{/.}.png --format PNG ::: color/*.tif
```

### Part 3: Animate

To animate this set of `png`s we'll use the `ffmpeg` library. To encode this dataset in a way that maintains a high quality file while keeping the size down requires customization of the command, and a "double-pass" encoding technique. To keep this simpler, we've put this command into a bash script, and made the parameters that you may want to tweak into variables - the input `dir` and output `movie`, `bitrate`, and `framerate`:

```bash
#!/usr/bin/env bash

dir=png
movie=output/pwat.mp4
bitrate=4000k
framerate=12
ffmpeg -r $framerate -y -f image2 -pattern_type glob -i "$dir/*.png" -c:v libx264 -preset slow -b:v $bitrate -pass 1 -c:a libfdk_aac -b:a 0k -f mp4 -r $framerate -profile:v high -level 4.2 -pix_fmt yuv420p -movflags +faststart /dev/null && \
ffmpeg -r $framerate -f image2 -pattern_type glob -i "$dir/*.png" -c:v libx264 -preset slow -b:v $bitrate -pass 2 -c:a libfdk_aac -b:a 0k -r $framerate -profile:v high -level 4.2 -pix_fmt yuv420p -movflags +faststart $movie
```
View the created animation to see how it looks. You can adjust the speed by modifying the framerate: lower framerate = slower. If the animation is too large (I try to stay under 5mb), you can lower the bitrate.

#### Style your reference data

Let's first modify the reference data in the style that we used for our visualization in [Part 1](http://commercedataservice.github.io/tutorial_mapbox_part1/). Navigate to https://www.mapbox.com/studio/styles/, find your style, and duplicate it:

![image](https://cloud.githubusercontent.com/assets/5084513/13894313/9b06ea62-ed23-11e5-93f0-5a410edd3ffb.png)

Then, edit this copied style. At the very least, we need to:
1. Delete the static pwat layer we used in part 1; and
2. Change the water + background colors to match our video.

BEFORE deleting the layer, move our previous pwat layer to the back to "stand in" for our video while we still - drag it to the bottom of the layers panel, _right_ above "Background."

Take note of the _first_ color category you used in the colorization process. In my case, I had:
```
10.0 23 31 39
```
Values 10.0 and under are the RGB value 23, 31, 39. Let's make this our water (fill) and background color:
![image](https://cloud.githubusercontent.com/assets/5084513/13894412/b3028544-ed24-11e5-8ecf-0fd344ccf8d0.png)

Since our animation will be _under_ the map, let's also make the water transparent - I am starting with 0.25.

You may also want to remove certain features, and change the style of others. I like to keep terrain visible, as I find the interplay between mountain ranges and pwat fascinating:

![inter](https://cloud.githubusercontent.com/assets/5084513/13924322/18f7cb0c-ef41-11e5-9d02-1ba23b44f6fa.gif)

Once you are done styling, click publish, and save your style url. Don't worry - you can edit your style and access its url at any time.

#### Add your data to the map

We'll be using [Mapbox GL JS](https://www.mapbox.com/mapbox-gl-js/api/) to create an interactive animated map with our data. First, create an html document, and copy in the skeleton html for a Mapbox GL map:

```code
<!DOCTYPE html>
<html>
<head>
    <meta charset='utf-8' />
    <title></title>
    <meta name='viewport' content='initial-scale=1,maximum-scale=1,user-scalable=no' />
    <script src='https://api.tiles.mapbox.com/mapbox-gl-js/v0.11.1/mapbox-gl.js'></script>
    <link href='https://api.tiles.mapbox.com/mapbox-gl-js/v0.11.1/mapbox-gl.css' rel='stylesheet' />
    <style>
        body { margin:0; padding:0; }
        #map { position:absolute; top:0; bottom:0; width:100%; }
    </style>
</head>
<body>

<div id='map'></div>
<script>
// code goes here!
</script>
</body>
</html>
```

Next, we'll create a GL map in the `#map` div, and add our style to it. Within the `<script></script>` tags, add:

```Javascript

// mapboxgl.accessToken = '<your access token - acquire from https://www.mapbox.com/studio/account>';

var map = new mapboxgl.Map({
    container: 'map', // container id
    style: 'mapbox://styles/dnomadb/cilybj8yj00a99km8o74phrfp', //stylesheet url from above
    center: [
              0,
              0
            ], // starting position
    zoom: 1.6 // starting zoom
});
```

Lets also add a contol for our map:
```
map.addControl(new mapboxgl.Navigation());
```

If you view your map, you should see something pretty bare bones:
![image](https://cloud.githubusercontent.com/assets/5084513/13925430/80464276-ef45-11e5-9d0d-adc21966e19e.png)

Let's add our video! For convenience, move the video to the same directory as your `html` file from above. To insert this into our map, we'll need to specify 3 main things:

1. When the video should be added to the map. In our case, we'll want to do it _after_ our stylesheed loads so that it is accessible.
2. Where to add the video spatially. We'll be using our clip bounds of `-180.0 -70.0 180.0 70.0` as the bounds for the video.
3. Where to add the video in terms of z-index. We want to add our video directly under the style's `hillshade` layer.

Here's how to do these three things:

```Javascript
map.once('style.load', function() {
    // once the style loads, do this
    map.batch(function() {
        // add our video source
        map.addSource('storm-source', {
            type: 'video',
            urls: [
                'pwat.mp4' // path to our video
            ],
            coordinates: [
                [ -180, 70 ],
                [ 180, 70 ],
                [ 180, -70 ],
                [ -180, -70 ]
            ] // the coordinates to which your data is clipped:
        });
        // add this layer to the map
        map.addLayer({
            type: 'raster',
            id: 'storm-layer',
            source: 'storm-source',
            paint: {
              'raster-opacity': 0.75
            }
        },
        'hillshade_shadow_faint'); // "after" our lowest hillshade layer
    });
});
```

Add this to your script, and your map should be ready to go!





