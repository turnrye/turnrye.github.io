---
title: "Generating WISP Coverage Maps"
date: 2023-02-20T00:00:00-06:00
draft: true
---

Based on W9CR https://fasma.org/modeling-repeaters-in-north-florida/


https://github.com/kd7lxl/signalserver-scripts

https://github.com/turnrye/Signal-Server

For this we'll be using a dockerized version of the signal-server. This is handy because the project doesn't distribute binaries and is a bit sporatically maintained, so this lets us skip any sort of build env setup steps. And docker is fun to play in, afterall. 

To explore the image, drop into a shell:

```
docker run -it --entrypoint /bin/bash signal-server
```

## Get your SRTM data setup

Download your data from:
http://viewfinderpanoramas.org/Coverage%20map%20viewfinderpanoramas_org3.htm

Place the zips in a convenient directory; in my case `/Users/turnrye/srtm`.

Script this via:
wget http://viewfinderpanoramas.org/dem3/{I,J}{15..16}.zip


Unzip them
Flatten the directories `find ./*/* -type f -print0 | xargs -0 -J % mv % .`

Unfortunately these are in hgt format and we need sdf. So let's convert them.

Convert srtm data:
```
docker run -v /Users/turnrye/srtm:/srtm -v /Users/turnrye/sdf:/sdf --entrypoint /bin/bash signal-server -c 'cd /sdf && (for i in /srtm/*; do /signal-server/utils/sdf/usgs2sdf/srtm2sdf $i; done)'
```

## Create your color file

```
cat <<- EOF > /Users/turnrye/signal-server/color/red.dcf
-70: 255, 0, 0
EOF
```

## Get your az els

https://help.ui.com/hc/en-us/articles/204952114-airMAX-Antenna-Data

AM-5G19-120

➜  signal-Server git:(master) ✗ md5 ~/Downloads/AM-5G19-120/AM-5G19-120-Hpol.ant
MD5 (/Users/turnrye/Downloads/AM-5G19-120/AM-5G19-120-Hpol.ant) = 0dc3f7fe40859e467ac26086c5498cba
➜  signal-Server git:(master) ✗ md5 ~/Downloads/AM-5G19-120/AM-5G19-120-Vpol.ant
MD5 (/Users/turnrye/Downloads/AM-5G19-120/AM-5G19-120-Vpol.ant) = 4e2da576824f7027e8bdba31392a4fb3

Put the ant files in a convenient directory (in my case it's ~/signal-server/ant)

docker run -v /Users/turnrye/signal-server/ant:/ant --entrypoint python2 signal-server /signal-server/utils/antenna/ant2azel.py -i /ant/AM-5G19-120-Vpol.ant
docker run -v /Users/turnrye/signal-server/ant:/ant --entrypoint python2 signal-server /signal-server/utils/antenna/ant2azel.py -i /ant/AM-5G19-120-Hpol.ant

## Prep your run

Execution:
```
-f 5900 -erp 39811 -rxh 30 -rt -70 -R 62 -lat 48.7080722 -lon -122.3930283 -txh 60 -o GalbraithAllSectors
-lat 46.4879 -lon -123.2144 -txh 60 -f 145 -erp 30 -rxh 5 -rt -100 -R 80 -o outfile 
signalserver -sdf $SRTMDIR/SRTM3 -dbm -pm 1 -dbg 


```
docker run -v /Users/turnrye/signal-server/color:/color -v /Users/turnrye/signal-server/ant:/ant -v /Users/turnrye/sdf:/sdf -v /Users/turnrye/signal-server/out:/out signal-server -m -sdf /sdf -dbm -pm 1 -dbg -lat 35.1002 -lon -89.8640 -txh 100 -f 5900 -erp 30 -rxh 5 -rt -60 -R 80 -ant ../../ant/AM-5G19-120-Hpol -hp -color /color/red.dcf -o /out/hil

```


Ugh, the above doesn't work, but if I run this inside the container is does. Forces me to put the antenna file in the /out directory named hil.{az,el}:
```
cp /Users/turnrye/signal-server/out/ant.az /Users/turnrye/signal-server/out/sco1.az
cp /Users/turnrye/signal-server/out/ant.el /Users/turnrye/signal-server/out/sco1.el
cp /Users/turnrye/signal-server/out/ant.az /Users/turnrye/signal-server/out/sco2.az
cp /Users/turnrye/signal-server/out/ant.el /Users/turnrye/signal-server/out/sco2.el
cp /Users/turnrye/signal-server/out/ant.az /Users/turnrye/signal-server/out/sco3.az
cp /Users/turnrye/signal-server/out/ant.el /Users/turnrye/signal-server/out/sco3.el
```

And then run the conversion:

```
docker run -it -v /Users/turnrye/signal-server/color:/color -v /Users/turnrye/signal-server/ant:/ant -v /Users/turnrye/signal-server/sdf:/sdf -v /Users/turnrye/signal-server/out:/out signal-server -sdf /sdf -m -dbm -pm 1 -dbg -lat 35.138667 -lon -90.020167 -txh 62.7888 -f 5900 -erp 39811 -rxh 3 -rt -70 -R 100 -rot 0 -o /out/sco1
docker run -it -v /Users/turnrye/signal-server/color:/color -v /Users/turnrye/signal-server/ant:/ant -v /Users/turnrye/signal-server/sdf:/sdf -v /Users/turnrye/signal-server/out:/out signal-server -sdf /sdf -m -dbm -pm 1 -dbg -lat 35.138667 -lon -90.020167 -txh 62.7888 -f 5900 -erp 39811 -rxh 3 -rt -70 -R 100 -rot 120 -o /out/sco2
docker run -it -v /Users/turnrye/signal-server/color:/color -v /Users/turnrye/signal-server/ant:/ant -v /Users/turnrye/signal-server/sdf:/sdf -v /Users/turnrye/signal-server/out:/out signal-server -sdf /sdf -m -dbm -pm 1 -dbg -lat 35.138667 -lon -90.020167 -txh 62.7888 -f 5900 -erp 39811 -rxh 3 -rt -70 -R 100 -rot 240 -o /out/sco3
```

## Convert the output to a file you can use

```
convert /Users/turnrye/signal-server/out/sco1.ppm -transparent white /Users/turnrye/signal-server/out/sco1.png
convert /Users/turnrye/signal-server/out/sco2.ppm -transparent white /Users/turnrye/signal-server/out/sco2.png
convert /Users/turnrye/signal-server/out/sco3.ppm -transparent white /Users/turnrye/signal-server/out/sco3.png
```

## Make it into a geotiff

Given this output from signalserver: `|36.004994|-88.768667|34.204996|-90.968667|`
|36.038666|-88.920167|34.238668|-91.120167|
Use gdal_translate like so:
```
docker run -it -v /Users/turnrye/signal-server/out:/out --entrypoint gdal_translate osgeo/gdal -of GTiff -a_srs EPSG:4326 -a_ullr -91.120167 36.038666 -88.920167 34.238668 /out/sco1.png /out/sco1.geotiff 
docker run -it -v /Users/turnrye/signal-server/out:/out --entrypoint gdal_translate osgeo/gdal -of GTiff -a_srs EPSG:4326 -a_ullr -91.120167 36.038666 -88.920167 34.238668 /out/sco2.png /out/sco2.geotiff 
docker run -it -v /Users/turnrye/signal-server/out:/out --entrypoint gdal_translate osgeo/gdal -of GTiff -a_srs EPSG:4326 -a_ullr -91.120167 36.038666 -88.920167 34.238668 /out/sco3.png /out/sco3.geotiff 
```
Out plops a geotiff that you can use! Le voila!
