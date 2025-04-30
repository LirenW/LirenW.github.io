---
# 必填
title: Problems and solutions when interpolating NCEP-NCAR reanalysis data  
published: 2025-02-21 20:00:00

# 可选
# description: 
updated: 2025-02-21 20:00:00
tags:
  - tools
  - Python
  - Climate

# 进阶，可选
draft: false
# pin: 0
toc: false
lang: en
# abbrlink: theme-guide
---

Recently, the daily data of NCEP-NCAR Reanalysis 1 is needed to calculate the diffuse skylight ratio. The calculation formula requires the following four variables:
1. Visible Beam Downward Solar Flux
2. Visible Diffuse Downward Solar Flux
3. Near IR Beam Downward Solar Flux
4. Near IR Diffuse Downward Solar Flux

And the diffuse skylight ratio (`diffuse_skylight_ration`) can be calculated by the following formula: 
$$
diffuse\_skylight\_ration = \frac{vddsf+nddsf}{vbdsf+vddsf+nbdsf+nddsf}
$$
This step is calculated using Python.  

## Interpolation

A diffuse skylight ratio can be obtained on each grid. The resolution of the NCEP-NCAR Reanalysis 1 data is $192 \times 94$. At this time, I need to use the bilinear interpolation algorithm to interpolate the source data to the grid consistent with MODIS (0.05-degree grid). Generally speaking, there are the following three methods: 
- CDO，[Climate Data Operators](https://code.mpimet.mpg.de/projects/cdo)
- NCO，[The netCDF Operators](https://nco.sourceforge.net/nco.html)
- [xESMF: Universal Regridder for Geospatial Data](https://xesmf.readthedocs.io/en/latest/)

However, none of these three methods can obtain "correct" results:
1. CDO method: Can output results, but all are missing values (`NaN`) 
```Shell
> cdo remapbil,MODIS_005deg.grid nbdsf.sfc.gauss.2001.nc nbdsf.sfc.MODIS.2001.nc
```
After interpolation for some time, cdo successfully output the result which is all `NaN`. 

2. NCO method: Cannot output the result and the following error will occur  
```
Grid(src): /tmp/ncremap_tmp_grd_src.nc.pid445452
Grid(dst): /tmp/ncremap_tmp_grd_dst.nc.pid445452
nco_err_exit(): ERROR Short NCO-generated message (usually name of function that triggered error): nco__enddef()
nco_err_exit(): ERROR Error code is -62. Translation into English with nc_strerror(-62) is "NetCDF: One or more variable sizes violate format constraints"
ncremap: ERROR Failed to generate map-file. Debug this:
ncks -O --dmm_in_mk --thr_nbr=2 --no_tmp_fl --hdr_pad=10000 --gaa remap_script=ncremap --gaa remap_hostname=login04 --gaa remap_version=5.2.1 --grd_src="/tmp/ncremap_tmp_grd_src.nc.pid445452" --grd_dst="/tmp/ncremap_tmp_grd_dst.nc.pid445452" --map_fl="/tmp/ncremap_tmp_map_nco_ncoaave.nc.pid445452"  --rgr lat_nm_out=lat --rgr lon_nm_out=lon "/tmp/ncremap_tmp_dmm.nc.pid445452" "/tmp/ncremap_tmp_out.nc.pid445452" > /dev/null
```

3. xESMF: Outputs incorrect interpolation results  
```Python
import xarray as xr
import xesmf as xe

grid_file = "NCAR2MODIS.nc"
with xr.open_dataset(
    "MODIS.nc"
) as ds:
    ds_tgt = ds
ds_out = xr.Dataset(
    {
        "lat": (["lat"], ds_tgt.lat.values, {"units": ds_tgt.lat.units}),
        "lon": (["lon"], ds_tgt.lon.values, {"units": ds_tgt.lon.units}),
    }
)

if not os.path.exists(grid_file):
    print("Generate grid file")

    regridder = xe.Regridder(diffuse_ratio, ds_out, "bilinear", parallel=True)
    regridder.to_netcdf(grid_file)

else:
    regridder = xe.Regridder(diffuse_ratio, ds_out, "bilinear", weights=grid_file)
```

The interpolation result is shown in the following figure (the time is 2001-01-01). There is an obviously unreasonable straight line with a value of 0 near 0°, and it extends to the entire polar region. 

Since this is interpolation for `diffuse_skylight_ration`, if interpolation is performed on each four variables, due to the issue of data volume (the interpolated result is 32GB), interpolating the radiative fluxes is not realistic. 

## Solution
This problem seems to stem from the missing values in the data, which not only include `NaN`, but also seem to include `Inf`. So we can process the data first and then perform interpolation. My operations are as follows:
1. First, limit the result to 0-1 during the calculation
2. Assign the missing value `NaN` as $1.0e20$, and specify this value as `_FillValue` when writing 

The statements in Python are as follows:  
```Python
 diffuse_ratio_arr = np.where(diffuse_ratio_arr < 0, np.nan, diffuse_ratio_arr)
 diffuse_ratio_arr = np.where(diffuse_ratio_arr > 1, np.nan, diffuse_ratio_arr)
 diffuse_ratio[:] = diffuse_ratio_arr
 diffuse_ratio = diffuse_ratio.fillna(1.e20)

 diffuse_ratio = diffuse_ratio.to_dataset(name="diffuse_ratio")

  encoding = {
     "lat": {"zlib": False, "_FillValue": None},
     "lon": {"zlib": False, "_FillValue": None},
     "diffuse_ratio": {"zlib": False, "_FillValue": 1.0e20},
 }
 diffuse_ratio.to_netcdf(f"{in_path}/test.nc", encoding=encoding)
```
After processing in this way and using cdo for interpolation, the correct result can be obtained. 
