+++
title = "react+mapbox初实践"
date = "2018-11-11" 
tags = ["react","mapbox"]
categories=["前端"]
+++

> 本篇博文主要搭建React+Mapbox的测试环境，简单测试Mapbox的调用。

#### 前置环境/条件
- 已经安装在node 、npm或cnpm服务；
- 会React基本操；
- 需要在去mapbox申请注册。

### 1. 创建工程
- 简便起见，使用手脚架创建项目工程，起名react-mapbox。
```
create-react-app react-mapbox
```
创建好基本工程后，安装mapbox相关插件
```
> cnpm i -D mapbox-gl

> cnpm i -D @mapbox/mapbox-gl-language

```

创建完工程后，写一份简单的使用案例：

```
import React, {Component} from 'react';
import mapboxgl from 'mapbox-gl';
import '../assets/style/map.css';

class Map extends Component {
    constructor(props) {
        super(props);
    }

    componentDidMount() {
        mapboxgl.accessToken = <Your accessToken>;

        //设置地图区域
        let bounds = [
            [118.21, 28.11], // Southwest coordinates，西南坐标
            [122.40, 31.33]  // Northeast coordinates，东北坐标
        ];

        new mapboxgl.Map({
            style: 'mapbox://styles/mapbox/streets-v10',
            center: [120.15, 30.3], //地图中心经纬度
            zoom: 11.5, //缩放级别
            minZoom: 9,
            maxZoom: 19,
            pitch: 45,
            bearing: -17.6,
            container: 'map',
            maxBounds: bounds
        });
    }

    render() {
        return (
            <div id="map" className="map">
            </div>
        );
    }
}

export default Map;
```

### mapbox 官网注册
mapbox的访问需要通过一个accessToken，这必须通过去其官网注册，官网地址：https://www.mapbox.com 

注册后获取产生一个accessToken，将其赋值给上面的mapboxgl.accessToken即可。