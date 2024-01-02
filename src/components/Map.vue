<script setup>
import 'mapbox-gl/dist/mapbox-gl.css';
import mapbox from 'mapbox-gl';
import * as turf from '@turf/turf'
import vueDanmaku from 'vue3-danmaku'
import { ref, onMounted } from 'vue'

const danmus = ref(['我是弹幕', '好好好', '嘿嘿嘿', '哈哈哈', 'akjdflfj', 'akdflakjf', 'lkdajdfoa', 'aldfjakf', '我是测试数据']);
onMounted(() => {
    initMapbox()
})
async function initMapbox() {
    mapbox.accessToken = 'pk.eyJ1IjoiemhvbmdkaXNodW1hIiwiYSI6ImNsNXJoYXR5eTI2bGgzZW53d2didWF1c3AifQ.6vOplM2NQc_xnJW3aA5ZBA';
    const map = new mapbox.Map({
        container: 'map-container', // container ID
        style: 'mapbox://styles/mapbox/streets-v12', // style URL
        center: [114.35400105, 30.6512685], // starting position [lng, lat]
        zoom: 10, // starting zoom
        minZoom: 10, // 最小缩放层级
    });
    const poi = await createPoi()
    console.log(poi);
    map.on('load', function () {
        viewportChanged()
        map.addSource('wuhanSource', {
            type: 'geojson',
            data: poi
        });
        map.addLayer({
            id: 'wuhanLayer',
            type: 'heatmap',
            source: 'wuhanSource',
        });
    });
    map.on('zoomend', function () {
        console.log("zoomend");
        viewportChanged();
    })
    map.on('moveend', function () {
        console.log("moveend");
        viewportChanged();
    })
    //视口改变时调用此方法
    function viewportChanged() {

        if (map.getSource('gridsource')) {
            map.removeLayer('grid-layer')
            map.removeSource('gridsource')
        } else {
            const squareGrid = createGrid();
            const randomPoints = poi;
            squareGrid.features.forEach(gridCell => {

                // 将随机点分配到网格中
                const pointsInCell = randomPoints.features.filter(point => turf.booleanPointInPolygon(point, gridCell.geometry));
                console.log(pointsInCell.length);
                gridCell.properties.pointCount = parseInt(pointsInCell.length);
                console.log(gridCell);
            });
            console.log(squareGrid);

            // 找出最大和最小点数
            const maxPointCount = Math.max(...squareGrid.features.map(cell => cell.properties.pointCount));
            const minPointCount = Math.min(...squareGrid.features.map(cell => cell.properties.pointCount));

            map.addSource('gridsource', {
                type: 'geojson',
                data: squareGrid // 这里假设squareGrid是已经生成的网格数据
            })

            map.addLayer({
                id: 'grid-layer',
                type: 'fill',
                source: 'gridsource',
                paint: {
                    'fill-color': [
                        'interpolate',
                        ['linear'],
                        ['get', 'pointCount'], // 假设每个格网包含点的数量在属性中称为 pointCount
                        minPointCount, 'rgba(0, 255, 0, 0.7)', // 最小点数时的颜色，绿色
                        maxPointCount, 'rgba(255, 0, 0, 0.7)' // 最大点数时的颜色，红色
                    ],
                    'fill-opacity': 0.7 // 填充透明度
                }
            });
        }
    }
    //根据视口生成格网
    function createGrid() {
        const result = getBounds(); //获取视口边界
        const { lngMin, lngMax, latMin, latMax } = result
        const bbox = [lngMin, latMin, lngMax, latMax];  //格网坐标 minX,minY,maxX,maxY. modify here.
        const topLeft = turf.point([lngMin, latMin]);
        const bottomLeft = turf.point([lngMin, latMax]);

        const height = turf.distance(topLeft, bottomLeft, { units: 'kilometers' });

        // const currentZoom = map.getZoom()
        const cellNum = 10
        const cellSize = height / cellNum;  //格网长度 
        const options = {
            units: 'kilometers',
        };
        const squareGrid = turf.squareGrid(bbox, cellSize, options);  //生成格网

        // console.log(squareGrid);

        // const cellSizeX = 0.2; // X 方向上格网的宽度
        // const cellSizeY = 0.1; // Y 方向上格网的高度

        // const grid = [];
        // for (let x = mapViewport[0]; x < mapViewport[2]; x += cellSizeX) {
        //     for (let y = mapViewport[1]; y < mapViewport[3]; y += cellSizeY) {
        //         const cell = turf.bboxPolygon([x, y, x + cellSizeX, y + cellSizeY]);
        //         grid.push(cell);
        //     }
        // }

        // console.log('生成的矩形格网数量：', grid.length);

        // // 将网格数据转换为 GeoJSON 格式
        // const gridGeoJSON = turf.featureCollection(grid.map(cell => turf.bboxPolygon(turf.bbox(cell))));

        // // 输出 GeoJSON 格式的网格数据
        // console.log(JSON.stringify(gridGeoJSON));
        return squareGrid;
    }
    //生成poi点
    function createPoi() {
        return new Promise((resolve, reject) => {
            // 使用 Fetch API 加载 GeoJSON 文件
            fetch('./data/wuhan.json')
                .then(response => response.json()) // 将响应转换为 JSON 格式
                .then(data => {
                    // 使用 Turf.js 处理加载的 GeoJSON 数据
                    const turfData = turf.featureCollection(data.features);
                    resolve(turfData);
                })
                .catch(error => {
                    reject(error);
                });
        })

    }
    // 获取视口边界
    function getBounds() {
        const bounds = map.getBounds()
        const result = {
            lngMax: bounds._ne.wrap().lng, // 最大经度
            lngMin: bounds._sw.wrap().lng, // 最小经度
            latMax: bounds._ne.wrap().lat, // 最大纬度
            latMin: bounds._sw.wrap().lat, // 最小纬度
        }
        return result
    }
}
</script>

<template>
    <vue-danmaku class="danmaku" ref="danmakuRef" v-model:danmus="danmus" useSlot loop :speeds="100" :channels="10">
        <template v-slot:dm="{ danmu }">
            <div class="danmaku-name">{{ danmu }}</div>
        </template>
    </vue-danmaku>
    <div id="map-container">
    </div>
</template>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#map-container {
    padding: 0%;
    margin: 0%;
    width: 100%;
    height: 100%;
    overflow: hidden;
}

.danmaku {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    z-index: 999;
    pointer-events: none;
    /* 禁止弹幕元素被选中 */
    /* background-color: #fff; */
}

.danmaku-name {
    color: red;
}
</style>
