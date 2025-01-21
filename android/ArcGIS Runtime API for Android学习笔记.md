# ArcGIS Runtime API for Android学习笔记

## Display a map

`mapview`是一个用来显示地图的UI组件，同时还负责用户与地图的交互。

显示地图的第一步就是将map view添加到xml文件中。

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <com.esri.arcgisruntime.mapping.view.MapView
        android:id="@+id/mapView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

然后

```java
public class MainActivity extends AppCompatActivity {
    //创建一个MapView对象
    private MapView mMapView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		//设置API_KEY
        ArcGISRuntimeEnvironment.setApiKey("API_KEY");
        //将activity中的mapView和xml文件中的mapView绑定
        mMapView = findViewById(R.id.mapView);
        //使用地形底图来创建一个地图
        ArcGISMap map = new ArcGISMap(BasemapStyle.ARCGIS_TOPOGRAPHIC);
        //使用mapView来展示map
        mMapView.setMap(map);
        //展示在屏幕上的地图是以lat和lon为中心，放缩一万倍的地图
        mMapView.setViewpoint(new Viewpoint(34.056295, -117.195800, 10000));
    }
    @Override
    protected void onPause() {
        mMapView.pause();
        super.onPause();
    }

    @Override
    protected void onResume() {
        super.onResume();
        mMapView.resume();
    }

    @Override
    protected void onDestroy() {
        mMapView.dispose();
        super.onDestroy();
    }
}
```

## Offline Map

### Use mobile packages

To use a **mobile package** in an offline application, you first create the mobile package using ArcGIS Pro. You then sideload or download the package to the device.

An offline application can access a mobile package to display fully interactive maps or scenes. Mobile packages support offline applications that never have a network connection.

Mobile packages do not support editing of data content，You can use mobile packages even if a network connection *is* available. 

### Use data files

A data file is a local file stored on a device that contains data that can be displayed by a layer in a map or scene. Some data files also support editing, querying, and analyzing data

To use a **data file** in an offline application, you need to sideload or download the data file to the device. You can get data files from many sources, including desktop GIS tools such as ArcGIS Pro, data conversion scripts, and the internet. Data files are typically used in offline applications that are unable to use a network connection.

You can use the following types of data files:

| Data file type                   | Supports editing |
| -------------------------------- | ---------------- |
| Shapefile                        | Yes              |
| Vector tile package              | No               |
| Image tile package               | No               |
| Scene Layer Package              | No               |
| Local raster file                | No               |
| OGC GeoPackage (feature data)    | Yes              |
| OGC GeoPackage (raster data)     | No               |
| OGC KML file                     | Yes              |
| Electronic Nautical Chart (S-57) | No               |
| Other (e.g. GeoJSON)             | Yes              |

在安卓端加载Shapefile文件的关键是ShapefileFeatureTable（com.esri.arcgisruntime.data.ShapefileFeatureTable）。
相比于.geodatabase文件，Shapefile文件的缺点在于只是单图层，且没有符号化，当然可以通过移动端的可视化API进行处理。

## 将shp文件拷贝到模拟器的SD卡中

1. 找到自己安装SDK的目录
2. 进入SDK目录下的tools文件
3. 双击monitor.bat文件（monitor.bat打不开的问题：[AndroidStudio3.x 打开Android Device Monitor正确做法](https://blog.csdn.net/CodeFarmer__/article/details/96836764)）
4. 出现如下界面![image-20210510001501647](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210510001501647.png)
5. 点击File Explorer
6. 找到sdcard文件所在位置（可能在mnt下，也有可能在外面）
7. 点击右上角的导入即可拷贝文件
