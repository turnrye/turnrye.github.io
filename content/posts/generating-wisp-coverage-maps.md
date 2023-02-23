---
title: "Generating WISP Coverage Maps"
date: 2023-02-20T00:00:00-06:00
draft: true
---

Based on prior works: https://fasma.org/modeling-repeaters-in-north-florida/ and https://github.com/kd7lxl/signalserver-scripts.

For this we'll be using a dockerized version of the signal-server. This is handy because the project doesn't distribute binaries and is a bit sporatically maintained, so this lets us skip any sort of build env setup steps. And docker is fun to play in, afterall. You can find the dockerfile in my fork of the [Signal-Server](https://github.com/turnrye/Signal-Server) project.

To explore the image, drop into a shell:

```
docker run -it --entrypoint /bin/bash signal-server
```

## Get your SRTM data setup

We'll use the rough 3 arc second data available widely online. Far more accurate data is available at this point, but to simplify the tutorial, let's go with what's easily available. Download your data from http://viewfinderpanoramas.org/Coverage%20map%20viewfinderpanoramas_org3.htm. I scripted this using a command `wget http://viewfinderpanoramas.org/dem3/{I,J}{15..16}.zip` but this turned out to be way more data than I needed.

Place the zips in a convenient directory; in my case I used `/Users/turnrye/signal-server/srtm`. Unzip them, and then flatten the directories `find ./*/* -type f -print0 | xargs -0 -J % mv % .`. The result should be one flat directory with all of the `.hgt` files present. 

Unfortunately, signal-server uses a format called `sdf` instead, so now we need to convert them. Let's use the built-in conversion script for that. This command maps two volumes into a docker container and then runs the `srtm2sdf` script on a loop within the docker container to perform the conversions.

```
docker run -v /Users/turnrye/signal-server/srtm:/srtm -v /Users/turnrye/signal-server/sdf:/sdf --entrypoint /bin/bash signal-server -c 'cd /sdf && (for i in /srtm/*; do /signal-server/utils/sdf/usgs2sdf/srtm2sdf $i; done)'
```

## Get your antenna data

Next, get the antenna data that you'll use for your map. If you're using a Ubiquiti antenna, you can find it [on this page](https://help.ui.com/hc/en-us/articles/204952114-airMAX-Antenna-Data). For this tutorial, I'll be using the AM-5G19-120. Download the `.ant` files.

Unzip the files to a convenient location, in my case `/Users/turnrye/signal-server/ant`. Then, just like before we need to convert the files. Let's use the `ant2azel` conversion script for this:

```
docker run -v /Users/turnrye/signal-server/ant:/ant --entrypoint python2 signal-server /signal-server/utils/antenna/ant2azel.py -i /ant/AM-5G19-120-Vpol.ant
docker run -v /Users/turnrye/signal-server/ant:/ant --entrypoint python2 signal-server /signal-server/utils/antenna/ant2azel.py -i /ant/AM-5G19-120-Hpol.ant
```

Our application expects the antenna for a given output file to be present in the same directory and named the same as the output file name (less the extension). So this means we now need to copy these files to our `/out` directory where we'll expect results to be.

Also, we actually are only going to be generating the coverage report for one polarity despite having dual polarity. This is because in all honesty, we don't expect this map to want to show coverages that could only be achieved with both polarities. This means that in some scenarios the map will be pessimistic, but that's for the better: it's almost always too generous anyway.

So, let's copy just the horizontal polarity data into our `out` directory with a simple `ant` name to shorten the next few commands.

```
cp /Users/turnrye/signal-server/ant/AM-5G19-120-Hpol.az /Users/turnrye/signal-server/out/ant.az
cp /Users/turnrye/signal-server/ant/AM-5G19-120-Hpol.el /Users/turnrye/signal-server/out/ant.el
```

## Create the coverage models

Now it's time to actually generate the coverage map. This assumes you want to run the coverage for one site with three sector antennas at (0,120,240) degrees true bearing. We'll call this "sco" and number the sectors 1, 2, and 3 accordingly.

First, copy the antenna files into place

```
cp /Users/turnrye/signal-server/out/ant.az /Users/turnrye/signal-server/out/sco1.az
cp /Users/turnrye/signal-server/out/ant.el /Users/turnrye/signal-server/out/sco1.el
cp /Users/turnrye/signal-server/out/ant.az /Users/turnrye/signal-server/out/sco2.az
cp /Users/turnrye/signal-server/out/ant.el /Users/turnrye/signal-server/out/sco2.el
cp /Users/turnrye/signal-server/out/ant.az /Users/turnrye/signal-server/out/sco3.az
cp /Users/turnrye/signal-server/out/ant.el /Users/turnrye/signal-server/out/sco3.el
```

Then, generate the coverages. Note these commands are where you specify key things like lat/lon, tx height, frequency, erp, rx height, receive threshold, and antenna bearing. To see what these options are, run `docker run -it --rm signal-server`

```
docker run -it -v /Users/turnrye/signal-server/color:/color -v /Users/turnrye/signal-server/ant:/ant -v /Users/turnrye/signal-server/sdf:/sdf -v /Users/turnrye/signal-server/out:/out signal-server -sdf /sdf -m -dbm -pm 1 -dbg -lat 35.138667 -lon -90.020167 -txh 62.7888 -f 5900 -erp 39811 -rxh 3 -rt -70 -R 100 -rot 0 -o /out/sco1
docker run -it -v /Users/turnrye/signal-server/color:/color -v /Users/turnrye/signal-server/ant:/ant -v /Users/turnrye/signal-server/sdf:/sdf -v /Users/turnrye/signal-server/out:/out signal-server -sdf /sdf -m -dbm -pm 1 -dbg -lat 35.138667 -lon -90.020167 -txh 62.7888 -f 5900 -erp 39811 -rxh 3 -rt -70 -R 100 -rot 120 -o /out/sco2
docker run -it -v /Users/turnrye/signal-server/color:/color -v /Users/turnrye/signal-server/ant:/ant -v /Users/turnrye/signal-server/sdf:/sdf -v /Users/turnrye/signal-server/out:/out signal-server -sdf /sdf -m -dbm -pm 1 -dbg -lat 35.138667 -lon -90.020167 -txh 62.7888 -f 5900 -erp 39811 -rxh 3 -rt -70 -R 100 -rot 240 -o /out/sco3
```

## Convert the output to a file you can use

The previous commands output an antiquated ppm file format which we can't really use. So, let's convert it to PNG.

```
convert /Users/turnrye/signal-server/out/sco1.ppm -transparent white /Users/turnrye/signal-server/out/sco1.png
convert /Users/turnrye/signal-server/out/sco2.ppm -transparent white /Users/turnrye/signal-server/out/sco2.png
convert /Users/turnrye/signal-server/out/sco3.ppm -transparent white /Users/turnrye/signal-server/out/sco3.png
```

The trouble is, PNGs dont have any geodata. We dont want to lose this, so let's add it to the file using the geotiff format. First, we have to refer back to our signalserver output from above. It tells us the bounds of the images. For instance, given this output from signalserver: `|36.004994|-88.768667|34.204996|-90.968667|`, we can juggle the numbers around and then use gdal_translate like so:
```
docker run -it -v /Users/turnrye/signal-server/out:/out --entrypoint gdal_translate osgeo/gdal -of GTiff -a_srs EPSG:4326 -a_ullr -91.120167 36.038666 -88.920167 34.238668 /out/sco1.png /out/sco1.geotiff 
docker run -it -v /Users/turnrye/signal-server/out:/out --entrypoint gdal_translate osgeo/gdal -of GTiff -a_srs EPSG:4326 -a_ullr -91.120167 36.038666 -88.920167 34.238668 /out/sco2.png /out/sco2.geotiff 
docker run -it -v /Users/turnrye/signal-server/out:/out --entrypoint gdal_translate osgeo/gdal -of GTiff -a_srs EPSG:4326 -a_ullr -91.120167 36.038666 -88.920167 34.238668 /out/sco3.png /out/sco3.geotiff 
```

Out plops a geotiff that you can use in a GIS application. From here you could make tiles to publish you map to the web or do further GIS analysis.
