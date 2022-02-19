+++
title = "Map matching using OSRM"
date = 2022-01-08
+++
Hoping to be more active during the pandemic, I bought a Withings smart watch, and at some point got the idea of playing with the GPS tracks recorded when using it.
The manufacturer allows you to download your data by logging in to their web dashboard, going to settings, and choosing "Download my data".
In 10 minutes or so, you will get a ZIP archive in your inbox.

## The Withings data

I like to take long walks and I'm currently living in a small suburb near a larger city, so most of my walks were around the town.
Let's look first at the format of the data.
There are 38 CSVs and a small `README.txt`:

```
 303790 Jan  8 12:01 activities.csv
   8253 Jan  8 11:53 aggregates_calories_earned.csv
   8980 Jan  8 11:53 aggregates_calories_passive.csv
   8591 Jan  8 11:53 aggregates_distance.csv
   7029 Jan  8 11:53 aggregates_elevation.csv
    447 Jan  8 11:53 aggregates_manual_spo2.csv
   7274 Jan  8 11:53 aggregates_steps.csv
   2091 Jan  8 11:53 bp.csv
     55 Jan  8 11:53 height.csv
   1576 Jan  8 11:53 manual_spo2.csv
     30 Jan  8 12:01 note.csv
 821784 Jan  8 11:59 raw_bed_calories-earned.csv
2414611 Jan  8 11:59 raw_bed_hr.csv
     21 Jan  8 11:59 raw_bed_maximum_movement.csv
     21 Jan  8 11:59 raw_bed_movement_pim.csv
     21 Jan  8 11:59 raw_bed_respiratory-rate.csv
  59615 Jan  8 11:59 raw_bed_sleep-state.csv
     21 Jan  8 11:59 raw_bed_snoring.csv
7775984 Jan  8 11:53 raw_hr_hr.csv
2258790 Jan  8 11:56 raw_location_altitude.csv
     21 Jan  8 11:56 raw_location_direction.csv
2151043 Jan  8 11:56 raw_location_gps-speed.csv
2196946 Jan  8 11:56 raw_location_horizontal-radius.csv
2376827 Jan  8 11:56 raw_location_latitude.csv
2377039 Jan  8 11:56 raw_location_longitude.csv
1992700 Jan  8 11:56 raw_location_vertical-radius.csv
 293536 Jan  8 11:58 raw_spo2_auto_spo2.csv
2615639 Jan  8 11:58 raw_spo2_quality_score.csv
  13277 Jan  8 11:58 raw_swim_lap-pool.csv
1562522 Jan  8 11:54 raw_tracker_calories-earned.csv
1483782 Jan  8 11:54 raw_tracker_distance.csv
1303766 Jan  8 11:54 raw_tracker_elevation.csv
   5553 Jan  8 11:54 raw_tracker_lap-pool.csv
 116658 Jan  8 11:54 raw_tracker_sleep-state.csv
1337423 Jan  8 11:54 raw_tracker_steps.csv
   3323 Jan  8 12:01 README.txt
1650810 Jan  8 11:53 signal.csv
  39917 Jan  8 12:01 sleep.csv
   1048 Jan  8 11:53 weight.csv
```

In the basic-but-better-than-last-time docs, we see:

```
raw_location_latitude.csv: Latitude data
· Duration: (seconds)

raw_location_longitude.csv: Longitude data
· Duration: (seconds)
```

```
# raw_location_latitude.csv
start,duration,value
2022-01-06T12:50:26+01:00,[60],[44.XXX]
2022-01-06T13:28:49+01:00,[60],[44.XXX]
2022-01-06T13:28:59+01:00,[60],[44.XXX]

# raw_location_longitude.csv
start,duration,value
2022-01-06T12:50:26+01:00,[60],[26.XXX]
2022-01-06T13:28:49+01:00,[60],[26.XXX]
2022-01-06T13:28:59+01:00,[60],[26.XXX]
```

The `XXX`s are not in the original files, I've (quite pointlessly) masked the digits after the decimal point.
I don't think we care about the durations, but the timestamps are not monotonic, which might matter later.
Fortunately, the timestamps appear to be correlated across the two files.
Otherwise it would be quite annoying to match the two values.
On the other hand, we have some weird-looking entries:

```
# raw_location_latitude.csv
[snip]
2022-01-06T13:45:36+01:00,"[60,60]","[44.XXX,657.XXX]"
[snip]
2021-11-10T13:14:42+01:00,"[60,60]","[44.XXX,2021.XXX]"
[snip]
2021-11-09T15:05:49+01:00,"[60,60]","[44.XXX,-217.XXX]"

# raw_location_longitude.csv
[snip]
2022-01-06T13:45:36+01:00,"[60,60]","[26.XXX,277.XXX]"
[snip]
2021-11-10T13:14:42+01:00,"[60,60]","[26.XXX,1448.XXX]"
[snip]
2021-11-09T15:05:49+01:00,"[60,60]","[26.XXX,-534.XXX]"
```

At first sight, those might appear to be two measurements merged into one entry, as the duration seems legit.
But the second coordinate values are clearly bogus.
There are no entries with three values, and those with a single one don't have any outrageous outliers.
So whatever these are, we will make sure to ignore them.

I won't show any code now, but let's extract the points from there, open [QGIS](https://qgis.org/) and check whether the values (for one arbitrary day) appear to be correct:

<a href="/assets/gps-points.webp" target="_blank"><img src="/assets/gps-points.webp"></a>

Yes, there are a couple of outliers but overall it's fine.

## Map matching

My hope was to match these points to road segments obtained from OpenStreetMap data.
It took me a while to stumble across a way to do this, but I managed to find the [OSRM](http://project-osrm.org/) project.
OSRM is open-source and they have a demo web service, but the strict query length limitation made trying it a was of time.

Looking at the docs, we find the [`match`](http://project-osrm.org/docs/v5.24.0/api/#match-service) endpoint.
It takes a set of coordinates and returns a list of route legs corresponding to them.
You can see an example if you follow the previous link.

I found a couple of sites that offered OSM exports, but couldn't figure out quickly how to register for an API key.
In the end, I downloaded a `pbf` file for my whole country from <https://data.osm-hr.org/>.
It was quite small, at 247 MB.

In order to use OSRM, we need to do some pre-processing steps, and also pick a profile.
`foot` is a good choice here because I don't care about one-way streets and passable barriers.

```
docker run --rm -t -v $PWD:/data osrm/osrm-backend osrm-extract -p /opt/foot.lua /data/romania.osm.pbf
docker run --rm -t -v $PWD:/data osrm/osrm-backend osrm-partition /data/romania.osrm
docker run --rm -t -v $PWD:/data osrm/osrm-backend osrm-customize /data/romania.osrm
```

On my computer this took 33 seconds and used about 3.1 GB RAM at its peak.
Now we can start the OSRM routing service:

```
docker run --rm -p 5000:5000 -v $PWD:/data osrm/osrm-backend osrm-routed --algorithm mld --max-matching-size 5000 /data/romania.osrm
```

It's a bit weird, but `match` also expects a profile.
It doesn't seem to matter (as you would expect, since we picked a profile during the pre-processing), and this was slightly confusing.
Does it work?

<a href="/assets/gps-tracks-bad.webp" target="_blank"><img src="/assets/gps-tracks-bad.webp"></a>

No, that's not great.
I tried a couple of things: displaying all the matched legs, changing the profile (I originally used the wrong one), sorting the points by timestamp, and even including the timestamps in the request:

<a href="/assets/gps-tracks-better.webp" target="_blank"><img src="/assets/gps-tracks-better.webp"></a>

This is much better.
What if we also use the location accuracy, available in the `raw_location_horizontal-radius.csv` file from the archive?

<a href="/assets/gps-tracks-not-better.webp" target="_blank"><img src="/assets/gps-tracks-not-better.webp"></a>

This slows down the matching from "apparently instant" to almost 90 seconds, and introduces some errors.
Looking at the accuracy radius values, most of them are around 10 m, but some have suspicious values like 600 and 800 m.
Let's keep the points with accuracy better than 50 m:

<a href="/assets/gps-tracks-better-again.webp" target="_blank"><img src="/assets/gps-tracks-better-again.webp"></a>

With the old output in green, you can see that most of the extra segments are now gone.
But some are still missing and this confused me quite a bit until I looked at the OSM data.

Hint: don't try open a `pbf` file directly; you can use [GDAL](https://gdal.org/) to convert it to a format with spatial index support:

```
ogr2ogr romania.osm.gpkg romania.osm.pbf
```

<a href="/assets/gps-tracks-osm-gaps.webp" target="_blank"><img src="/assets/gps-tracks-osm-gaps.webp"></a>

It's hard to see in the image, but there's a visible gap near the top-left corner.
Could my OSM export have been too old?
Looking at that site again, the other files are from 2018.
My download probably triggered an update, since now the `pbf` shows a timestamp from today.

I searched for other ways to get the OSM data and [Geofabrik](https://download.geofabrik.de/) seems to be more popular.
Let's see how it looks:

<a href="/assets/gps-tracks-new-osm.webp" target="_blank"><img src="/assets/gps-tracks-new-osm.webp"></a>

Unfortunately, some segments are still missing or wrong, even though this time they are present in the OSM data.
Checking one of them, it had an `access => private` tag, which makes sense.
There are some residential buildings with barriers and I might have passed near some of them without noticing it wasn't allowed.

The `foot` profile has a list of blacklisted tags:

```lua
    access_tag_blacklist = Set {
      'no',
      'agricultural',
      'forestry',
      'private',
      'delivery',
    },
```

I removed them, re-ran the OSRM pre-processing, and re-did the matching:

<a href="/assets/gps-tracks-not-bad.webp" target="_blank"><img src="/assets/gps-tracks-not-bad.webp"></a>

The missing segment is a foot path through an uncultivated area between two recently-built apartment buildings:

<a href="/assets/gps-photo.jpg" target="_blank"><img src="/assets/gps-photo-thumb.webp"></a>

But overall it's quite decent, so I guess this is it for today.
Thanks for reading if you got to the end.

In the next post, I will try to show these tracks in a more interesting form.

*Map data from [OpenStreetMap](https://openstreetmap.org/copyright)*
