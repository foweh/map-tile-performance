# Leaflet大图叠加缩放卡顿优化方案

## 问题
1024px NDVI图用L.imageOverlay贴上去，缩放的时候整个图重新采样，卡成PPT。

## 方案一：关闭缩放动画（最简单，立竿见影）
```javascript
const map = L.map('map', {
  zoomAnimation: false,      // 关缩放动画，直接跳档
  fadeAnimation: false,
  markerZoomAnimation: false,
  zoomSnap: 1,
  zoomDelta: 1
});
```
效果：缩放的时候没有平滑过渡，直接切到下一级，虽然糙但是不卡。用户接受的话这是成本最低的方案。

## 方案二：自定义GridLayer动态生成256瓦片（丝滑）
```javascript
const NDVIGrid = L.GridLayer.extend({
  createTile: function(coords, done) {
    const tile = document.createElement('canvas');
    tile.width = tile.height = 256;
    const ctx = tile.getContext('2d');
    
    // 从1024源图裁剪对应区域画到256瓦片上
    // ...坐标计算...
    
    done(null, tile);
    return tile;
  }
});

const ndviLayer = new NDVIGrid({
  minZoom: 12,
  maxZoom: 18,
  tileSize: 256,
  opacity: 0.75,
  updateWhenZooming: false,  // 关键：缩放动画期间不生成瓦片，结束后再画
  keepBuffer: 4              // 视口外多缓存4圈，回拉秒出
});
```

## 方案三：预切片（最丝滑）
用gdal2tiles.py把GeoTIFF切成标准XYZ瓦片，用L.tileLayer加载：
```bash
gdal2tiles.py -z 12-18 -p raster ndvi.tif tiles/
```
```javascript
L.tileLayer('/tiles/{z}/{x}/{y}.png', { maxZoom: 18 }).addTo(map);
```
缺点：要提前切片，体积大，用户动态画地块没法重切。

## 为什么会卡
| 方案 | 缩放时浏览器干了啥 | 卡顿程度 |
|---|---|---|
| imageOverlay 1024图 | 整个1024px位图重新采样 | 严重 |
| imageOverlay + 关动画 | 直接跳，不重采样 | 轻微 |
| GridLayer动态瓦片 | 每个256瓦片单独画，主线程压力小 | 丝滑 |
| 预切片tileLayer | 直接加载现成png，浏览器只做CSS transform | 最丝滑 |

## 许可证
MIT
