---
# 必填
title: 插值NCEP-NCAR再分析数据时出错及解决方法
published: 2025-02-21 20:00:00

# 可选
# description: 
updated: 2025-02-21 20:00:00
tags:
  - Python
  - Climate

# 进阶，可选
draft: false
# pin: 0
toc: false
lang: zh
# abbrlink: theme-guide
---

最近需要使用NCEP-NCAR Reanalysis 1的逐日数据计算漫射天窗比（diffuse skylight ration）。其计算公式需要用到以下四个变量：
1. Visible Beam Downward Solar Flux（可见光直射辐射，vbdsf）
2. Visible Diffuse Downward Solar Flux（可见光散射辐射，vddsf）
3. Near IR Beam Downward Solar Flux（近红外直射辐射，nbdsf）
4. Near IR Diffuse Downward Solar Flux（近红外散射辐射，nddsf）

而漫射天窗比（`diffuse_skylight_ration`）可以如下公式计算：
$$
diffuse\_skylight\_ration = \frac{vddsf+nddsf}{vbdsf+vddsf+nbdsf+nddsf}
$$
这一步骤是使用Python计算的。

## 插值问题

在每个网格上都能得到一个漫射天窗比，NCEP-NCAR Reanalysis 1数据的分辨率为$192 \times 94$。此时我需要用双线性插值算法将源数据插值到与MODIS一致的网格（0.05度网格）。一般来说有以下三种方法：
- CDO，[Climate Data Operators](https://code.mpimet.mpg.de/projects/cdo)
- NCO，[The netCDF Operators](https://nco.sourceforge.net/nco.html)
- [xESMF: Universal Regridder for Geospatial Data](https://xesmf.readthedocs.io/en/latest/)

但是用这三种方法都无法获得正常的结果：
1. CDO方法：能输出结果，但全为缺失值（`NaN`）
```Shell
> cdo remapbil,MODIS_005deg.grid nbdsf.sfc.gauss.2001.nc nbdsf.sfc.MODIS.2001.nc
```
经过一段时间的插值，cdo成功输出了全是`NaN`的结果。

2. NCO方法：无法输出结果，会出现如下报错
```
Grid(src): /tmp/ncremap_tmp_grd_src.nc.pid445452
Grid(dst): /tmp/ncremap_tmp_grd_dst.nc.pid445452
nco_err_exit(): ERROR Short NCO-generated message (usually name of function that triggered error): nco__enddef()
nco_err_exit(): ERROR Error code is -62. Translation into English with nc_strerror(-62) is "NetCDF: One or more variable sizes violate format constraints"
ncremap: ERROR Failed to generate map-file. Debug this:
ncks -O --dmm_in_mk --thr_nbr=2 --no_tmp_fl --hdr_pad=10000 --gaa remap_script=ncremap --gaa remap_hostname=login04 --gaa remap_version=5.2.1 --grd_src="/tmp/ncremap_tmp_grd_src.nc.pid445452" --grd_dst="/tmp/ncremap_tmp_grd_dst.nc.pid445452" --map_fl="/tmp/ncremap_tmp_map_nco_ncoaave.nc.pid445452"  --rgr lat_nm_out=lat --rgr lon_nm_out=lon "/tmp/ncremap_tmp_dmm.nc.pid445452" "/tmp/ncremap_tmp_out.nc.pid445452" > /dev/null
```

3. xESMF：输出错误的插值结果
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

插值结果（时间为2001-01-01）在0°附近有一条值为0的明显不合理的直线，并且延伸到了整个极地地区。


由于这里是对`diffuse_skylight_ration`插值，假如对四个变量插值，由于数据量的问题（插值后的结果为32GB），对辐射通量插值并不现实。

## 解决方案
这一问题似乎是源自数据中的缺失值，其中不仅包含`NaN`，似乎还包含`Inf`。这样我们可以先对数据进行处理，之后再进行插值，我的操作如下：
1. 首先在计算时将结果限制在0-1
2. 将缺失值`NaN`指定为$1.0e20$，并在写入时将这个值指定为`_FillValue`

在Python中语句如下：
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

这样处理后使用cdo插值，就能够得到正确的结果了。
