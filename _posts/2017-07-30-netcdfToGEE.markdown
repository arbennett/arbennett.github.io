---
layout: post
title:  "A Pipeline for Analysis of NetCDF data in Google Earth Engine"
date:   2017-07-30
categories: Software, Hydrology
---

_Note: An example of the code used in this pipeline is available in [this
gist](https://gist.github.com/arbennett/6185add8773d312a2802614fca5e9527)_


[Google Earth Engine](https://earthengine.google.com/) (GEE) is a platform that
combines an catalogue of satellite remote sensing data with a data analysis
API and environment for combining them.  Having direct and instantaneous
access to these huge amounts of data is a boon to the field in itself,
but it would be useful to be able to upload your own data to combine with
this data catalogue.

The way GEE delineates data types is similar to most GIS programs, with data
being either raster or vector.  In GEE terminology, raster data are _Images_ and
vector data are _Features_.  Timeseries of each of these data types can be stored
in _ImageCollections_ and _FeatureCollections_, respectively.

For users looking to upload small amounts of files, it is not too cumbersome
to simply upload the data within the code editor interface.  However, once the
dataset becomes more than about 25 images or features this becomes a very tedious
task; more than 100 and it would be downright mind-numbing. It is this case that
I became concerned with.

To further complicate things, the Earth science community has mostly standardized
around the NetCDF data format, which is not natively accepted by GEE. So, now we
have two problems.  First, we need to figure out how to process the NetCDF datasets
we want to analyze in GEE into a format that GEE will be able to handle. Then, we
need to automate the upload process so that we don't have to individually choose
file to upload.

# Translating NetCDF to GTIFF

GEE can interpret raster data in the GeoTIFF format, so it is what we will target
for the NetCDF data. My initial implementation was based on
[this approach.](https://www.linkedin.com/pulse/convert-netcdf4-file-geotiff-using-python-chonghua-yin)
At the top level there is some basic scaffolding where we open the dataset, and
delegate tasks to helper functions.  The main call is:

{% highlight python %}
import os
import sys
from osgeo import gdal, osr, gdal_array
import xarray as xr
import numpy as np
import pandas as pd

BAND_VARS = ['mean', 'min', 'max', 'std', 'median']

def main(args):
	varname, infile, outdir, outname = args
	if os.path.exists(outdir) and os.path.isfile(outdir):
		exit("Output path exists and is a regular file! \n"
	    	 "Please provide a new output directory and try again")
	if not os.path.exists(outdir):
		os.mkdir(outdir)

	ds = xr.open_dataset(infile)
	ndv, xs, ys, geot, proj = get_netcdf_info(infile, varname)
	data = calculate_band_stats(infile, varname)
	dates = data.time.values
	n_bands = len(BAND_VARS)
	n_iter = len(dates)
	for i in range(n_iter):
		date = pd.to_datetime(str(dates[i])).strftime('%Y_%m_%d')
		print(date)
		create_geotiff('{}{}out_{}_{}'.format(outdir, os.path.sep, outname, date),
					   data.isel(time=i), ndv, xs, ys, geot, proj)
{% endhighlight %}

The core of this code is that we set up the metadata for the GTIFF,
do some calculations that will go into the bands, and finally, break
the NetCDF data up into time slices (each of which is its own GTIFF)
and write them out.  Each of these functions recieve their own helper
function.

The function for setting the GTIFF metadata is relatively straightforward:

{% highlight python %}
def get_netcdf_info(fname, var):
    print('Creating GDAL datastructures')
    subset = 'NETCDF:"' + fname + '":' + var
    sub_ds = gdal.Open(subset)
    nodata = sub_ds.GetRasterBand(1).GetNoDataValue()
    xsize = sub_ds.RasterXSize
    ysize = sub_ds.RasterYSize
    geot = sub_ds.GetGeoTransform()
    proj = osr.SpatialReference()
    proj.SetWellKnownGeogCS('NAD27')
    return nodata, xsize, ysize, geot, proj
{% endhighlight %}

For my particular application I wanted to get weekly statistics from
daily data, so I implemented a function to pull out that data and
put it into a new `xarray` dataset:

{% highlight python %}
def calculate_band_stats(fname, var):
    print('Calculating band stats')
    ds = xr.open_dataset(fname)
    results = []
    if var == 'Soil_liquid':
        ds = ds[var].sum(dim='soil_layers')
    else:
        ds = ds[var]
    print('  Calculating mean')
    ds_weekly = ds.resample('7d', dim='time', how='mean').to_dataset().rename({var: 'mean'})
    print('  Calculating min')
    ds_weekly['min'] = ds.resample('7d', dim='time', how='min')
    print('  Calculating max')
    ds_weekly['max'] = ds.resample('7d', dim='time', how='max')
    print('  Calculating std')
    ds_weekly['std'] = ds.resample('7d', dim='time', how='std')
    print('  Calculating median')
    ds_weekly['median'] = ds.resample('7d', dim='time', how='median')
    return ds_weekly
{% endhighlight %}

Finally, we can write out the data by creating the GTIFF data structure
with the number of bands that we require, then transfer the data from
the `xarray` dataset that was preprocessed into the GTIFF band. It may
be important to note that this method assumes all variables have the
same data type (ie float, int, etc).  The function is as follows:

{% highlight python %}
def create_geotiff(suffix, data, ndv, xsize, ysize, geot, proj):
    dt = gdal_array.NumericTypeCodeToGDALTypeCode(data[BAND_VARS[0]].values.dtype)
    if type(dt) != np.int:
        if dt.startswith('gdal.GDT_') is False:
            dt = eval('gdal.GDT_'+dt)
    new_fname = suffix + '.tif'
    zsize = len(BAND_VARS)
    driver = gdal.GetDriverByName('GTiff')
    ds = driver.Create(new_fname, xsize, ysize, zsize, dt)
    ds.SetProjection(proj.ExportToWkt())
    ds.SetGeoTransform(geot)
    for i, var in enumerate(BAND_VARS):
        d = np.flip(data[var].values, 0)
        d[np.isnan(d)] = ndv
        ds.GetRasterBand(i+1).WriteArray(d)
        ds.GetRasterBand(i+1).SetNoDataValue(ndv)
    ds.FlushCache()
    return new_fname
{% endhighlight %}


# Uploading GTIFFs to Google Cloud Storage

Now that the data exists in a format that GEE can understand we must
find a way to efficiently (and preferably in an automated fashion)
upload the data.  The documentation provided is not very clear about how
this should be done, but there is a nugget of information found within
the [command line tool's upload section](https://developers.google.com/earth-engine/command_line#upload).
Specifically:

`earthengine upload image --asset_id=users/username/asset_id gs://bucket/image.tif `

Where the last argument uses an address to a
[Google Cloud Storage (GCS)](https://cloud.google.com/storage/) Bucket.
So, before reading any further, you should go create an account there.
It's worth noting that this service does cost money based on the amount of
data you store, how often it's accessed, and where it's accessed.  I have
uploaded over 10,000 files (about 15 GB) and been charged $0.51.  Additionally,
there is a $300.00 credit to your account for the first year.

Once you have an account you should create a bucket that you will store your
data in.  I named mine `uw_hydro_raster`, so the address I use is `gs://uw_hydro_raster`.

Then, once you have an account on GCS, you will need to install
[`gsutil`](https://cloud.google.com/storage/docs/gsutil) in order to script the upload
process.  Once installed it's relatively easy to script the upload process.  I use:

{% highlight shell %}
#!/usr/bin/env bash
#
# Uploads some files to google cloud storage
#
# -----------------------------------------
# WARNING: This will make everything in the
#           output bucket public!!!
# -----------------------------------------
#
# Usage:
#   ./transfer_to_gcs.sh src_dir dest_bucket

gsutil cp $1/* gs://$2
gsutil -m acl set -R -a public-read gs://$2
{% endhighlight %}

Usage for this script is simple enough.  For example, if I want
to upload all of my data that's in the `~/arbennett/gtiff` directory
I would use:

`./transfer_to_gcs.sh ~/arbennett/gtiff uw_hydro_raster`


# Transferring data from Google Cloud Storage to Google Earth Engine

Okay, now you've got a process to get all of the data into a bucket, but
how can you actually _use_ that data in GEE?  Unfortunately, there is still
one more step.  In order to transfer the data you will have to install the
[`earthengine` command line tool](https://developers.google.com/earth-engine/command_line).

Then, you can use it in a script, like the one I've prepared below:

{% highlight shell %}
#!/usr/bin/env bash
# Uploads some files from google cloud storage to gee
# Usage:
#   ./transfer_to_gee.sh src_bucket dest_asset

result=`earthengine create collection users/$2`

if `test -z "$result"`; then
    echo $result
	exit 1
fi

for geotiff in `gsutil ls gs://$1/*.tif`; do
	filename=`basename $geotiff`
	asset_id="${filename%.*}"
	earthengine upload image --asset_id=users/$2/$asset_id $geotiff
done
{% endhighlight %}

Which would have usage:

`./transfer_to_gee.sh uw_hydro_raster arbennett/my_data`

And you've done it!  You should see all of the uploaded data in your _Assets_
tab in the GEE code editor.

# Caveats, pitfalls, etc

There are still some rather large limitations on uploading and using the
data.  First, it's important to note that there is a hard 10000 file limit
for users.   At a daily time step, this is roughly 27 years, so you may have
to bin your data differently for climatological studies. Second, user uploaded
data seems much slower to access and manipulate than the provided datasets,
so be aware of if GEE is the right tool for your study.


