<template>
  <div>
    <div ref="mapContainer" class="map"></div>
    <div class="controls">
      <el-cascader
        v-model="selected"
        :options="options"
        :props="{
          checkStrictly: true,
          value: 'adcode',
          label: 'name',
          lazy: true,
          lazyLoad: lazyLoad,
        }"
        placeholder="行政区域预GeoJson预览,请选择省/市"
        @change="areaChange"
        class="area-select"
        clearable
      />
      <div>
        <el-button type="primary" :icon="Crop" @click="cropMap">范围</el-button>
        <el-button type="primary" :icon="Crop" @click="openBox"
          >输入经纬度</el-button
        >
        <el-button
          type="primary"
          :icon="Download"
          @click="showDialog"
          :disabled="!available"
          >下载</el-button
        >
      </div>
    </div>
    <div class="zoom">{{ zoom }}</div>
    <el-dialog top="60px" v-model="isShowDialog" width="60%" append-to-body>
      <div v-if="available">
        <el-descriptions :column="2" border title="矩形范围坐标信息">
          <el-descriptions-item>
            <template #label> 左上 </template>
            <el-tag size="small">{{ rect.coordinatesLonLat[3] }}</el-tag>
          </el-descriptions-item>
          <el-descriptions-item>
            <template #label> 右上 </template>
            <el-tag size="small">{{ rect.coordinatesLonLat[2] }}</el-tag>
          </el-descriptions-item>
          <el-descriptions-item>
            <template #label> 左下 </template>
            <el-tag size="small">{{ rect.coordinatesLonLat[0] }}</el-tag>
          </el-descriptions-item>
          <el-descriptions-item>
            <template #label> 右下 </template>
            <el-tag size="small">{{ rect.coordinatesLonLat[1] }}</el-tag>
          </el-descriptions-item>
          <el-descriptions-item>
            <template #label> 中心点 </template>
            <el-tag size="small">{{ rect.centerLonLat }}</el-tag>
          </el-descriptions-item>
        </el-descriptions>
        <el-table
          :data="tableData"
          height="810"
          stripe
          border
          @selection-change="handleSelectionChange"
        >
          <el-table-column type="selection" width="55" />
          <el-table-column prop="zoom" label="缩放级别"></el-table-column>
          <el-table-column prop="totalTiles" label="瓦片数量"></el-table-column>
          <el-table-column prop="times" label="下载所需时间"></el-table-column>
        </el-table>
      </div>
      <template #footer>
        <div style="text-align: right">
          <el-button
            type="primary"
            @click="download()"
            :disabled="!multipleSelection.length"
            >下载</el-button
          >
        </div>
      </template>
    </el-dialog>
    <div class="loading" v-show="isShowLoading">
      <el-text type="primary" size="large" tag="b"
        >{{ progressDownloaded }} / {{ progressTotal }}</el-text
      >
      <el-progress
        :percentage="progressPercentage"
        :stroke-width="15"
        :color="'#409EFF'"
        striped
        striped-flow
        :duration="10"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from "vue";
import { Crop, Download } from "@element-plus/icons-vue";
import "ol/ol.css";
import Map from "ol/Map";
import View from "ol/View";
import TileLayer from "ol/layer/Tile";
import XYZ from "ol/source/XYZ";
import Draw from "ol/interaction/Draw";
import { Vector as VectorLayer } from "ol/layer";
import { Vector as VectorSource } from "ol/source";
import { createBox } from "ol/interaction/Draw";
import { fromLonLat, toLonLat } from "ol/proj";
import Feature from "ol/Feature";
import Point from "ol/geom/Point";
import { Style, Text, Fill, Stroke } from "ol/style";
import { getCenter } from "ol/extent";
import { ElMessage, ElMessageBox } from "element-plus";
import JSZip from "jszip";
import { saveAs } from "file-saver-fixed";
import GeoJSON from "ol/format/GeoJSON";
import { Polygon } from "ol/geom";

const url = ref(
  "http://wprd04.is.autonavi.com/appmaptile?lang=zh_cn&size=1&style=7&x={x}&y={y}&z={z}",
);

// 测试url，将下载的tiles.zip解压到public目录下进行测试，注意默认zoom是13
// const url = ref("http://localhost:5173/tiles/{z}/{x}/{y}.png");
const mapContainer = ref(null);
let mapInstance = null;
let source = null;
let geoJsonSource = null;
let drawInteraction = null;
const isShowDialog = ref(false);
const isShowLoading = ref(false);
const rect = ref({});
const progressDownloaded = ref(0);
const progressTotal = ref(0);
const multipleSelection = ref([]);
const zoom = ref(13);

const selected = ref([]);
const options = ref([{ name: "中国", adcode: 100000 }]);

const available = computed(() => {
  return Object.keys(rect.value).length > 0;
});

const progressPercentage = computed(() => {
  return Math.round((progressDownloaded.value / progressTotal.value) * 100);
});

// 获取行政区域
const lazyLoad = async (node, resolve) => {
  try {
    const res = await fetch(
      `https://geo.datav.aliyun.com/areas_v3/bound/${node.value}_full.json`,
    );
    const geojson = await res.json();
    const children = geojson.features.map((item) => {
      return { adcode: item.properties.adcode, name: item.properties.name };
    });
    resolve(children);
  } catch {
    resolve([]);
  }
};

onMounted(() => {
  // crop
  source = new VectorSource();
  const vector = new VectorLayer({
    source,
    style: new Style({
      stroke: new Stroke({
        width: 5,
        color: "rgb(64, 158, 255)",
      }),
    }),
  });

  // geoJson
  geoJsonSource = new VectorSource();
  const geoJsonVector = new VectorLayer({
    source: geoJsonSource,
    style: (feature) => {
      return new Style({
        fill: new Fill({
          color: "rgba(0, 128, 255, 0.3)", // 填充
        }),
        stroke: new Stroke({
          color: "#31326F", // 边界
          width: 2,
        }),
        // 内陆地区可开启，沿海有岛屿的地区会展示过多
        /*        text: new Text({
          text: feature.get("name") || "",
          font: "14px sans-serif",
          fill: new Fill({ color: "rgba(0,0,0,0.65)" }), // 字体颜色
          backgroundFill: new Fill({ color: "#fff" }), // 白色背景
          padding: [1, 1, 1, 1],
          overflow: true, // 避免被裁剪
        }),*/
      });
    },
  });

  mapInstance = new Map({
    target: mapContainer.value,
    layers: [
      new TileLayer({
        source: new XYZ({ url: url.value }),
      }),
      vector,
      geoJsonVector,
    ],
    view: new View({
      center: fromLonLat([120.15, 30.25]),
      zoom: 13,
    }),
  });

  // 监听 zoom 变化
  mapInstance.getView().on("change:resolution", () => {
    zoom.value = Math.round(mapInstance.getView().getZoom());
  });
});

// 选择地区
async function areaChange(val) {
  if (!val) {
    geoJsonSource.clear();
    return;
  }
  const adcode = val[val.length - 1]; // 最后一级就是市

  try {
    const res = await fetch(
      `https://geo.datav.aliyun.com/areas_v3/bound/${adcode}_full.json`,
    );
    const geojson = await res.json();
    geoJsonSource.clear();

    const features = new GeoJSON().readFeatures(geojson, {
      featureProjection: "EPSG:3857",
    });
    geoJsonSource.addFeatures(features);

    // 自动缩放到选中区域
    mapInstance
      .getView()
      .fit(geoJsonSource.getExtent(), { padding: [20, 20, 20, 20] });
  } catch (e) {
    console.error("加载 GeoJSON 出错:", e);
  }
}

onBeforeUnmount(() => {
  if (mapInstance) {
    mapInstance.setTarget(null);
    mapInstance = null;
  }
});

const cropMap = () => {
  if (source) {
    source.clear();
    rect.value = {};
  }

  if (drawInteraction) {
    mapInstance.removeInteraction(drawInteraction);
  }

  drawInteraction = new Draw({
    source: source,
    type: "Circle",
    geometryFunction: createBox(),
  });
  mapInstance.addInteraction(drawInteraction);

  drawInteraction.on("drawend", (evt) => {
    rect.value.geometry = evt.feature.getGeometry();
    markMap();
  });
};

const markMap = () => {
  const addFourCorners = () => {
    rect.value.coordinates.forEach((coord, index) => {
      const lonLat = toLonLat(coord);
      const pointFeature = new Feature({
        geometry: new Point(coord),
      });

      pointFeature.setStyle(
        new Style({
          text: new Text({
            text: `[${lonLat[0]}, ${lonLat[1]}]`,
            offsetY: index > 1 ? -15 : 15,
            font: "16px Arial",
            fill: new Fill({ color: "#000", fontWeight: "bold" }),
            stroke: new Stroke({ color: "#fff", width: 2 }),
          }),
        }),
      );
      source.addFeature(pointFeature);
    });
  };
  const addCenterPoint = () => {
    const extent = rect.value.geometry.getExtent();
    const center = getCenter(extent);
    const centerLonLat = toLonLat(center);
    rect.value.center = center;
    rect.value.centerLonLat = centerLonLat;

    const centerFeature = new Feature({
      geometry: new Point(center),
    });

    centerFeature.setStyle(
      new Style({
        text: new Text({
          text: `Center: [${centerLonLat[0]}, ${centerLonLat[1]}]`,
          font: "16px Arial",
          fill: new Fill({ color: "#000" }),
          stroke: new Stroke({ color: "#fff", width: 2 }),
        }),
      }),
    );

    source.addFeature(centerFeature);
  };

  const coordinates = rect.value.geometry.getCoordinates()[0].slice(0, -1);
  rect.value.coordinates = coordinates;
  rect.value.coordinatesLonLat = coordinates.map((coord) => toLonLat(coord));
  addFourCorners();
  addCenterPoint();
  mapInstance.removeInteraction(drawInteraction);
  drawInteraction = null;
};

const showDialog = () => {
  isShowDialog.value = true;
};

const handleSelectionChange = (val) => {
  multipleSelection.value = val;
};

const tableData = computed(() => {
  if (available.value) {
    return calculateTileCount(rect.value.geometry.getExtent(), 18);
  }
  return [];
});

const calculateTileCount = (extent, maxZoom) => {
  const tileCounts = [];
  const earthCircumference = 40075016.686;

  for (let zoom = 0; zoom <= maxZoom; zoom++) {
    const tilesPerSide = Math.pow(2, zoom);
    const tileWidth = earthCircumference / tilesPerSide;
    const [minX, minY, maxX, maxY] = extent;

    const minTileX = Math.floor((minX + earthCircumference / 2) / tileWidth);
    const maxTileX = Math.floor((maxX + earthCircumference / 2) / tileWidth);
    const minTileY = Math.floor((earthCircumference / 2 - maxY) / tileWidth);
    const maxTileY = Math.floor((earthCircumference / 2 - minY) / tileWidth);

    const tileCountX = maxTileX - minTileX + 1;
    const tileCountY = maxTileY - minTileY + 1;
    const totalTiles = tileCountX * tileCountY;
    const times = formatTime(((totalTiles * 0.1) / 6).toFixed(0));

    tileCounts.push({ zoom, totalTiles, times });
  }

  return tileCounts;
};

const formatTime = (seconds) => {
  if (isNaN(seconds) || seconds < 0) return "0秒";

  const h = Math.floor(seconds / 3600);
  const m = Math.floor((seconds % 3600) / 60);
  const s = seconds % 60;

  let result = "";
  if (h > 0) result += h + "小时";
  if (m > 0) result += m + "分钟";
  if (s > 0 || result === "") result += s + "秒";

  return result;
};

const download = async () => {
  const totalTiles = multipleSelection.value.reduce(
    (pre, cur) => pre + cur.totalTiles,
    0,
  );
  await ElMessageBox.confirm(
    `大概需要${formatTime(((totalTiles * 0.1) / 6).toFixed(0))}`,
    "是否下载?",
    {
      confirmButtonText: "确认",
      cancelButtonText: "取消",
      type: "warning",
    },
  );

  const extent = rect.value.geometry.getExtent();
  const zooms = multipleSelection.value.map((item) => item.zoom);
  const tileUrls = getTileUrls(extent, zooms);
  await downloadTilesAsZip(tileUrls);
};

const getTileUrls = (extent, zooms) => {
  const tileUrls = [];
  const earthCircumference = 40075016.686;
  const baseUrl = url.value;

  zooms.forEach((zoom) => {
    const tilesPerSide = Math.pow(2, zoom);
    const tileWidth = earthCircumference / tilesPerSide;
    const [minX, minY, maxX, maxY] = extent;

    const minTileX = Math.floor((minX + earthCircumference / 2) / tileWidth);
    const maxTileX = Math.floor((maxX + earthCircumference / 2) / tileWidth);
    const minTileY = Math.floor((earthCircumference / 2 - maxY) / tileWidth);
    const maxTileY = Math.floor((earthCircumference / 2 - minY) / tileWidth);

    for (let x = minTileX; x <= maxTileX; x++) {
      for (let y = minTileY; y <= maxTileY; y++) {
        const url = baseUrl
          .replace("{z}", zoom)
          .replace("{x}", x)
          .replace("{y}", y);
        tileUrls.push({
          x,
          y,
          zoom,
          url,
          path: `tiles/${zoom}/${x}/${y}.png`,
        });
      }
    }
  });

  return tileUrls;
};
const fetchTileBlob = async (url) => {
  const response = await fetch(url, { mode: "cors" });
  return await response.blob();
};

const downloadTilesAsZip = async (tileUrls) => {
  progressDownloaded.value = 0;
  isShowLoading.value = true;

  const zip = new JSZip();
  progressTotal.value = tileUrls.length;

  const batchSize = 6; // 提高并发量以加速
  for (let i = 0; i < progressTotal.value; i += batchSize) {
    const batch = tileUrls.slice(i, i + batchSize);
    const promises = batch.map(async ({ url, path }) => {
      const blob = await fetchTileBlob(url);
      zip.file(path, blob);
    });
    await Promise.all(promises);
    progressDownloaded.value += batchSize;
  }

  const content = await zip.generateAsync({ type: "blob" });
  saveAs(content, "tiles.zip");

  ElMessage({
    message: `下载完成！共成功下载 ${progressDownloaded.value.toString()} 张瓦片`,
    type: "success",
    duration: 5000,
  });
  isShowLoading.value = false;
  progressDownloaded.value = 0;
};
const openBox = async () => {
  const res = await ElMessageBox.prompt(
    "请输入左上和右下坐标（格式：lon,lat;lon,lat）",
    "框选范围",
    {
      confirmButtonText: "确定",
      cancelButtonText: "取消",
      inputPlaceholder: "120.0229,30.3295;120.2782,30.1623",
      inputPattern: /^-?\d+(\.\d+)?,-?\d+(\.\d+)?;-?\d+(\.\d+)?,-?\d+(\.\d+)?$/,
      inputErrorMessage: "格式错误，应为 lon,lat;lon,lat",
    },
  );
  // 解析输入的坐标
  const [topLeft, bottomRight] = res.value.split(";");
  const [topLeftLon, topLeftLat] = topLeft.split(",");
  const [bottomRightLon, bottomRightLat] = bottomRight.split(",");
  // 校验经纬度范围
  if (
    topLeftLon < -180 ||
    topLeftLon > 180 ||
    bottomRightLon < -180 ||
    bottomRightLon > 180 ||
    topLeftLat < -90 ||
    topLeftLat > 90 ||
    bottomRightLat < -90 ||
    bottomRightLat > 90
  ) {
    ElMessage.error("经纬度超出范围");
    return;
  }

  if (source) {
    source.clear();
    rect.value = {};
  }

  // WGS84 经纬度坐标 (EPSG:4326) - 矩形四个顶点（逆时针）
  const coordinatesLonLat = [
    [topLeftLon, bottomRightLat], // 左下
    [bottomRightLon, bottomRightLat], // 右下
    [bottomRightLon, topLeftLat], // 右上
    [topLeftLon, topLeftLat], // 左上
    [topLeftLon, bottomRightLat], // 闭合
  ];
  // 转换为 Web Mercator 坐标 (EPSG:3857)
  const coordinates = coordinatesLonLat.map((coord) => fromLonLat(coord));
  // 创建 OpenLayers Polygon 几何对象
  const geometry = new Polygon([coordinates]);

  rect.value.geometry = geometry;

  // 创建特征并设置样式（与 Draw 一致）
  const feature = new Feature({ geometry });

  // 添加到矢量图层
  source.addFeature(feature);

  markMap();
};
</script>

<style lang="less" scoped>
.controls {
  position: absolute;
  width: 100%;
  top: 30px;
  display: flex;
  flex-flow: row wrap;
  justify-content: center;
  column-gap: 12px;
  :deep(.area-select) {
    width: 300px;
  }
}
.zoom {
  position: absolute;
  padding: 2px 5px;
  top: 30px;
  right: 30px;
  background: #ffffff;
  color: rgba(0, 0, 0, 0.65);
  font-size: 24px;
  border-radius: 8px;
  box-shadow: 10px 10px 20px rgba(0, 0, 0, 0.3);
}
.map {
  width: 100%;
  height: 100vh;
}
.el-table {
  margin-top: 16px;
}
.loading {
  top: 0;
  position: absolute;
  height: 100%;
  width: 100%;
  background: rgba(255, 255, 255, 0.8);
  z-index: 9999;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  row-gap: 10px;
  .el-progress {
    width: 700px;
  }
}
</style>
