# Bridge/Mapbox/Mapbox.php 分析报告

## 概述

`Mapbox` 集成 Mapbox Geocoding API，提供地理编码（地址→坐标）和反向地理编码（坐标→地址）两个工具。

## 工具列表

| 工具名 | 方法 | 说明 |
|---|---|---|
| `geocode` | `geocode()` | 将地址字符串转换为经纬度坐标 |
| `reverse_geocode` | `reverseGeocode()` | 将经纬度坐标转换为地址 |

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Toolbox/Attribute/AsTool` | 工具注册注解 |
| `Platform/Contract/JsonSchema/Attribute/With` | 参数约束（limit 范围限制） |
