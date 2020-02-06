## Satpy支持的文件类型

[Satpy](https://satpy.readthedocs.io/en/latest/)目前[支持的卫星数据](https://satpy.readthedocs.io/en/latest/index.html#reader-table)有50种(MSG, Himawari 8, GOES-R, MODIS, Sentinel- 1/2/3/5, SNPP等)。

本文以最近[**朝曦dawn**](https://dreambooker.site/)添加的**风云4A(FY4A) AGRI L1**数据为例。

[**Notebook**](https://github.com/zxdawn/FY-4/tree/master/satpy/examples)已放在GitHub上，供大家学习。
<br>

---

FY4A AGRI L1数据有两种类别：

1.全圆盘
	
> FY4A-_AGRI--\_N_DISK_1047E_L1-_FDI-_MULT_NOM_20190807060000_20190807061459_4000M_V0001.HDF
    
2.中国区

> FY4A-_AGRI--\_N_REGC_1047E_L1-_FDI-_MULT_NOM_20190807045334_20190807045750_1000M_V0001.HDF

全圆盘标识：DISK , 中国区标识：REGC.

<br>


**风云数据链接**

1.[实时数据](https://fy4.nsmc.org.cn/data/en/data/realtime.html)(30天内)及文档

2.[历史数据](http://satellite.nsmc.org.cn/PortalSite/Data/Satellite.aspx) (2018-03-12 -- )

3.[FY4A官方应用平台](http://rsapp.nsmc.org.cn/geofy/)

## 安装

推荐通过conda安装，简单快捷。
```
$ conda install -c conda-forge satpy
```

## 定标

FY4A AGRI L1有以下三种定标方式:

1.原始数字量化值 (所有通道)

2.反射率 (C01 - C06)

3.辐射率和亮温 (C07 - C14)


## 读取数据

只需几行即可读取数据：
```python
import os, glob
from satpy.scene import Scene

# 加载FY4A文件
filenames = glob.glob('/xin/data/FY4A/20190807/FY4A-_AGRI*4000M_V0001.HDF')

# 创建scene对象
scn = Scene(filenames, reader='agri_l1')

# 查看可用的通道
scn.available_dataset_names()
```

输出结果:

    ['C01',
     'C02',
     'C03',
     'C04',
     'C05',
     'C06',
     'C07',
     'C08',
     'C09',
     'C10',
     'C11',
     'C12',
     'C13',
     'C14',
     'satellite_azimuth_angle',
     'satellite_zenith_angle',
     'solar_azimuth_angle',
     'solar_glint_angle',
     'solar_zenith_angle']

<br>

读取指定通道数据：


```python
# 以红外通道为例
ir_channel = 'C12'
scn.load([ir_channel])
```

## 绘图

### 全圆盘图

*一行命令*即可保存或显示图片：
```python
# 保存到文件
# scn.save_dataset(ir_channel, \
	filename='{sensor}_{name}.png')

# 在notebook中显示
scn.show(ir_channel)
```

<img src='./figures/agri_C12.png'>

### 全圆盘真彩色图

首先查看可用的合成选项：
```python
scn.available_composite_names()
```

输出结果显示，支持真彩图：

    ['ash',
     'dust',
     'fog',
     'green',
     'green_snow',
     'ir108_3d',
     'ir_cloud_day',
     'natural_color',
     'natural_color_sun',
     'night_background',
     'night_background_hires',
     'overview',
     'overview_sun',
     'true_color']

<br>

于是我们可以*直接加载*真彩色数据*并绘图*：


```python
# 注：这步需要大内存 (取决于cpu核数)
# 可查看FAQ关于内存的讨论：
#    https://satpy.readthedocs.io/en/latest/faq.html

composite = 'true_color'
scn.load([composite])
scn.show(composite)

# scn.save_dataset(composite, \
		filename='{sensor}_{name}.png')
```


<img src='./figures/agri_true_color.png'>

### 特定区域图

以下以台风利奇马为例。

首先，我们需要定义地图投影和区域，然后将数据投影到该区域上。

#### 方式一

用[Pyresample](http://pyresample.readthedocs.org/)来定义区域。


```python
from pyresample import get_area_def

area_id = 'lekima'

x_size = 549
y_size = 499
area_extent = (-1098006.560556, -967317.140452, \
		1098006.560556, 1026777.426728)
projection = '+proj=laea +lat_0=19.0 +lon_0=128.0 +ellps=WGS84'
description = "Typhoon Lekima"
proj_id = 'laea_128.0_19.0'

areadef = get_area_def(area_id, description, proj_id, projection,x_size, y_size, area_extent)
```

#### 方式二

用[coord2area_def.py](https://github.com/pytroll/satpy/blob/master/utils/coord2area_def.py)程序来直接生成区域定义。

比如用`python coord2area_def.py lekima_4km laea 10 28 118 138 4`，即可得到之前定义的利奇马区域：

```
lekima_4km:
  description: lekima_4km
  projection:
    proj: laea
    ellps: WGS84
    lat_0: 19.0
    lon_0: 128.0
  shape:
    height: 499
    width: 549
  area_extent:
    lower_left_xy: [-1098006.560556, -967317.140452]
    upper_right_xy: [1098006.560556, 1026777.426728]
```

<br>

然后将该定义拷贝到`$PPP_CONFIG_DIR/areas.yaml`中，即可*直接调用*。


```python
# 如果你已经添加区域到areas.yaml中,可直接调用:
os.environ['PPP_CONFIG_DIR'] = '/yin_raid/xin/satpy_config/'
lekima_scene = scn.resample('lekima_4km')

# 否则需要使用之前定义的区域:
# lekima_scene = scn.resample(areadef)
```


```python
lekima_scene.show(composite)
# lekima_scene.save_dataset(composite, \
		filename='{sensor}_{name}_resampled.png')
```

<img src='./figures/agri_true_color_resampled.png'>

如果想利用自定义的colormap来生成图像（如下图），请参阅关于`enhancement`的notebook（正在忍饿赶稿中）。

<img src='./figures/agri_C12_resampled_colorize.png'>
