<script setup>
import { getMap, initMap } from '@/utils/mainMap2.js'
import { fetchMockData } from '@/utils/mock.js'
import GLlayer from '#/gl-layers/lib/index.mjs'
import * as THREE from 'three'
import CropLayer from '@/components/map/cropLayer.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import * as dat from 'dat.gui'

const {
  PolygonLayer,
  FlowlineLayer,
  BorderLayer,
  DrivingLayer,
  TilesLayer,
  LayerManager
} = GLlayer

// 高德可视化类
let loca

// 容器
const container = ref(null)

// 图层管理
const layerManger = new LayerManager()

// 是否检测到有人试图越界
let isInvadeMode = false
let invadeClock = null
// 入侵者标记
let invadeMarker


// 是否第一人称
let isFirstView = false

// 无人机动画混合器
let mixer

let directionalLight
let gui
var guiCtrl = {
  lightPositionX: 1000,
  lightPositionY: 1000,
  lightPositionZ: 1600,
  cameraNear: 0,
  caremaFar: 20000,
  cameraLeft: -5000,
  cameraRight: 5000,
  cameraTop: 5000,
  cameraBottom: -5000,
  mapSize: 1024, // 1024
  planeMaterialOpacity: 0.5,
  intetity: 2.5,
  mixerPlaySpeed: 5
};

const allLayers = [
  { id: 'borderLayer', name: '区域边界', visible: true },
  { id: 'farmLayer', name: '农田', visible: true },
  { id: 'poolLayer', name: '鱼塘', visible: true },
  { id: 'waterLayer', name: '水域', visible: true },
  { id: 'buildingLayer', name: '村居', visible: true },
  { id: 'dronLayer', name: '无人机', visible: true },
  { id: 'dronPathLayer', name: '无人机轨迹', visible: true },
  { id: 'cropLayer', name: '农产品识别', visible: true },
  { id: 'riskLayer', name: '灾害预测', visible: false },
  { id: 'producationLayer', name: '年产量预测', visible: false },
  { id: 'serverLayer', name: '服务点', visible: false },
]

const SETTING = {
  center: [113.532936, 22.738711],
  alone: false,
  topicMap: {
    product: {
      label: '生产', //作物识别、农田灾害预测、水质检测
      layers: ['farmLayer', 'poolLayer', 'waterLayer', 'buildingLayer', 'riskLayer', 'cropLayer']
    },
    security: {
      label: '安全', //入侵检测、无人机巡航
      layers: ['borderLayer', 'waterLayer', 'buildingLayer', 'dronLayer', 'dronPathLayer', 'serverLayer']
    },
    value: {
      label: '经济', //产值预测
      layers: ['farmLayer', 'poolLayer', 'cropLayer', 'producationLayer']
    }
  }
}

// 地图数据提示浮窗
var normalMarker

onMounted(async () => {
  await init()
  // await getPointsData()
  await initLayers()
  animateFn()
  // initGUI()
})

var wxLayer

// 初始化地图
async function init() {
  // 区域掩模
  // const borderData = await getBoundaryData()
  const map = await initMap({
    viewMode: '3D',
    dom: container.value,
    showBuildingBlock: false,
    center: SETTING.center,
    zoom: 15.5,
    pitch: 42.0,
    rotation: 4.9,
    mapStyle: 'amap://styles/light',
    skyColor: '#c8edff'
  })

  // 添加卫星地图
  const satelliteLayer = new AMap.TileLayer.Satellite();

  map.add([satelliteLayer]);
  // 给底图添加css滤镜
  // document.querySelector('.amap-layer').style = 'filter:brightness(0.5) saturate(0.3) grayscale(0.5);'
  map.on('zoomend', (e) => {
    console.log(map.getZoom())
  })
  map.on('click', (e) => {
    const { lng, lat } = e.lnglat
    console.log([lng, lat])

    // 设置图层中心
    // const layer = layerManger.findLayerById('buildingLayer')
    // layer.setCenter([lng, lat])

    // 显示点击坐标
    // const marker = new AMap.Marker({
    //   map: getMap(),
    //   position: [lng, lat],
    //   label: {
    //     content: `[${lng},${lat}]`,
    //     direction: 'top'
    //   }
    // })
    if (normalMarker) {
      map.remove(normalMarker)
    }
  })

  loca = new Loca.Container({
    map,
  });

  normalMarker = new AMap.Marker({
    offset: [70, -15],
    zooms: [1, 22]
  });

  document.addEventListener('keydown', function (event) {
    var keyCode = event.keyCode;
    // 判断是否按下了数字键1、2、3
    if (keyCode === 49) { 
      switchTopic('product')
    } else if (keyCode === 50) {
      switchTopic('security')
    } else if (keyCode === 51) {
      switchTopic('value')
    }
  });
}

window.getMapView = function () {
  const center = mainMap.getCenter()
  const zoom = mainMap.getZoom()
  const pitch = mainMap.getPitch()
  const rotation = mainMap.getRotation()

  const res = {
    center,
    zoom,
    pitch,
    rotation
  }
  console.log(res)
  return res
}
window.setMapView = function ({ center, zoom, pitch, rotation }) {
  mainMap.setCenter(center)
  mainMap.setZoom(zoom)
  mainMap.setPitch(pitch)
  mainMap.setRotation(rotation)
}

async function initLayers() {
  // initWxLayer()
  await initBorderLayer()
  await initFarmLayer()
  await initPoolLayer()
  await initWaterLayer()
  await initBuildingLayer()
  await initDroneLayer()
  await initCropLayer()
  await initRiskLayer()
  await initProducationLayer()
  await initServePOILayer()
}
function initWxLayer() {
  const map = getMap()
  var wms = new AMap.TileLayer({
    // tileUrl: 'http://www.google.cn/maps/vt?lyrs=s@189&gl=cn&x=[x]&y=[y]&z=[z]',//google X
    tileUrl: 'https://gwxc.shipxy.com/tile.g?z=[z]&x=[x]&y=[y]', //shipInfo
    // tileUrl: 'https://mt1.google.com/vt/lyrs=h&x=[x]&y=[y]&z=[z]', //google 84 
    // tileUrl: 'http://gac-geo.googlecnapps.cn/maps/vt?lyrs=s&gl=CN&x={x}&y={y}&z={z}', //google cgj02 X
    zooms: [4, 22]
  });

  wms.setMap(map);

  wxLayer = wms
}

async function initServePOILayer() {
  const map = getMap()
  const data = await fetchMockData('serverpoi.geojson')

  const geo = new Loca.GeoJSONSource({ data });

  var labelsLayer = new Loca.LabelsLayer({
    zooms: [10, 20],
  })
  labelsLayer.setSource(geo);
  labelsLayer.setStyle({
    icon: {
      type: 'image',
      image: `./static/icons/server_poi.png`,
      size: [48, 75],
      anchor: 'center',
    },
    text: {
      // 每项配置都可使用回调函数来动态配置
      content: (index, feat) => {
        return feat.properties.name;
      },
      style: {
        fontSize: 12,
        fontWeight: 'normal',
        fillColor: '#5CDE8E',
        strokeColor: '#000',
        strokeWidth: 2,
      },
      direction: 'bottom',
    },
    extData: (index, feat) => {
      return feat.properties;
    }
  });
  loca.add(labelsLayer)

  labelsLayer.id = 'serverLayer'
  layerManger.add(labelsLayer)

  labelsLayer.on('complete', () => {
    var labelMarkers = labelsLayer.getLabelsLayer().getAllOverlays();
    for (let marker of labelMarkers) {
      marker.on('mouseover', (e) => {
        var position = e.data.data && e.data.data.position;
        const { name, area, department, manager, contact } = marker.getExtData()
        if (position) {
          normalMarker.setContent(
            `<div class="amap-info-window">
              <p>名称：${name} </p>
              <p>部门${department} </p>
              <p>面积：${area} 平方米</p>
              <p>负责人：${manager}</p>
              <p>电话：${contact}</p>
            </div>`,
          );
          normalMarker.setOffset(new AMap.Pixel(90, -40))
          normalMarker.setPosition(position);
          map.add(normalMarker);
        }
      });
      marker.on('mouseout', () => {
        map.remove(normalMarker);
      });
      marker.on('click', () => {
        const { view } = marker.getExtData()
        if (view) {
          setMapView(view)
        }
      })
    }
  });

}

/**
 * 初始化区域边界图层
 */
async function initBorderLayer() {
  const map = getMap()
  const data = await fetchMockData('border.geojson')

  const layer = new BorderLayer({
    id: 'borderLayer',
    alone: SETTING.alone,
    map,
    wallColor: '#3dfcfc', // 33ffe0
    wallHeight: 100,
    data,
    speed: 0.3,
    animate: true,
    zooms: [11, 22],
    altitude: 0
  })

  layerManger.add(layer)
}

/**
 * 初始化农田图层
 */
async function initFarmLayer() {
  const map = getMap()
  const data = await fetchMockData('farm.geojson')
  console.log(data)

  data.features.forEach(item => {
    const { used } = item.properties
    item.properties.color = used == 1 ? "#33a02c" : (used == 0 ? "#b2df8a" : "#ceb89e")
  })

  const layer = new PolygonLayer({
    id: 'farmLayer',
    alone: SETTING.alone,
    map,
    data,
    lineWidth: 0,
    opacity: 0.4,
    interact: true,
    zIndex: 100,
    altitude: 2
  })
  layerManger.add(layer)

}
/**
 * 初始化池塘图层
 */
async function initPoolLayer() {
  const map = getMap()
  const data = await fetchMockData('pool.geojson')
  console.log(data)

  const layer = new PolygonLayer({
    id: 'poolLayer',
    alone: SETTING.alone,
    map,
    data,
    lineWidth: 0,
    opacity: 0.4,
    interact: true,
    zIndex: 100,
    altitude: 2
  })
  layerManger.add(layer)

  // data.features.forEach((item) => {
  //   const { geometry, properties } = item
  //   const { used } = properties

  //   // debugger
  //   let polygon = new AMap.Polygon({
  //     path: geometry.coordinates[0][0],
  //     fillColor: used == 1 ? "#19bbf1" : (used == 0 ? "#c4e9fb" : "#988c48"),
  //     strokeOpacity: 1,
  //     fillOpacity: 0.7,
  //     strokeColor: "#fff",
  //     strokeWeight: 2,
  //     strokeOpacity: 0.2,
  //     strokeStyle: 'solid'
  //   });
  //   map.add(polygon)

  // });

}

/**
 * 添加水域图层
 */
async function initWaterLayer() {
  const map = getMap()
  const data = await fetchMockData('water.geojson')
  const layer = new GLlayers.WaterLayer({
    id: 'waterLayer',
    map,
    data,
    alone: SETTING.alone,
    zooms: [16, 22],
    animate: true,
    waterColor: '#CFEACD',
    altitude: -5
  })
  layerManger.add(layer)
}

/**
 * 初始化无人机巡航图层
 */
async function initDroneLayer() {
  const map = getMap()
  const data = await fetchMockData('dronWander2.geojson')
  const NPC = await getDroneModel()

  const layer = new DrivingLayer({
    id: 'dronLayer',
    map,
    zooms: [4, 30],
    path: data,
    altitude: 50,
    speed: 50.0,
    NPC,
    interact: true
  })
  layer.on('complete', ({ scene }) => {

    // 调整模型的亮度
    const aLight = new THREE.AmbientLight(0xffffff, 3.5)
    scene.add(aLight)

    layer.resume()
  })
  layerManger.add(layer)

  // 移动路径图层
  const dronPathLayer = new FlowlineLayer({
    id: 'dronPathLayer',
    map,
    zooms: [16, 22],
    data,
    speed: 0.5,
    lineWidth: 10,
    altitude: 50
  })
  // dronPathLayer.on('complete', ({ scene }) => {
  //   dronPathLayer.hide()
  // })
  layerManger.add(dronPathLayer)
}

function animateFn() {
  requestAnimationFrame(animateFn);
  if (mixer) {
    // 无人机旋转动画
    mixer.update(0.01); //必须加上参数才有动画    
  }
}

/**
 * 作物类型图层
 */
async function initCropLayer() {
  const map = getMap()
  const data1 = await fetchMockData('crop.geojson')
  const data2 = await fetchMockData('poolCenter.geojson')
  data1.features = data1.features.concat(data2.features)

  const layer = new CropLayer({
    id: 'cropLayer',
    data: data1,
    zooms: [16, 22],
    zIndex: 200,
    map
  })

  layer.on('mouseover', (e) => {
    const { crop, style } = e.data
    normalMarker.setPosition(e.data.lnglat);
    normalMarker.setOffset(new AMap.Pixel(90, -10))

    let content = ''
    if (style <= 4) {
      //农作物
      content = `<div class="amap-info-window">
        <p>作物: ${crop}</p>
        <p>识别匹配度: ${parseInt(Math.random() * 20) + 80}%</p>
        <p>产量预计: ${parseInt(Math.random() * 30) + 20}吨</p>
      </div>`
    } else {
      //水产品
      content = `<div class="amap-info-window">
        <p>作物: ${crop}</p>
        <p>产量预计: ${parseInt(Math.random() * 20) + 10}吨</p>
      </div>`
    }

    normalMarker.setContent(content)
    normalMarker.setMap(map)
  })
  layer.on('mouseout', (e) => {
    map.remove(normalMarker);
  })

  layerManger.add(layer)
}

/**
 * 灾害风险检测图层
 */
async function initRiskLayer() {
  const map = getMap()
  const data = await fetchMockData('fertility.geojson')
  const geo = new Loca.GeoJSONSource({ data })

  const heatmap = new Loca.HeatMapLayer({
    zIndex: 10,
    opacity: 1,
    visible: false,
    zooms: [2, 22],
  });

  heatmap.setSource(geo, {
    id: 'riskLayer',
    radius: 150,
    unit: 'meter',
    height: 300,
    gradient: {
      1: '#FF4C2F',
      0.8: '#FAA53F',
      0.6: '#FFF100',
      0.5: '#7DF675',
      0.4: '#5CE182',
      0.2: '#29CF6F',
    },
    value: function (index, feature) {
      return feature.properties.weight ?? 0;
    },
    min: 0,
    max: 100,
    visible: true
  });
  loca.add(heatmap);

  map.on('click', function (e) {
    const feat = heatmap.queryFeature(e.pixel.toArray());
    if (feat) {
      normalMarker.setAnchor('bottom-center')
      normalMarker.setOffset(new AMap.Pixel(0, -40))
      normalMarker.setPosition(feat.lnglat)
      normalMarker.setContent(`<div style="margin-bottom: 15px; border:1px solid #fff; 
         border-radius: 4px;color: #fff; width: 150px;
         text-align: center;">风险值: ${feat.value ?? 0}</div>`)
    }
  });

  heatmap.id = 'riskLayer'
  layerManger.add(heatmap)
}

/**
 * 产量预测图层
 */
async function initProducationLayer() {
  const data = await fetchMockData('productation.geojson')
  const geo = new Loca.GeoJSONSource({ data })
  const map = getMap()

  const hLayer = new Loca.HexagonLayer({
    zIndex: 220,
    opacity: 0.9,
    visible: false,
    zooms: [2, 22]
  });
  hLayer.setSource(geo);

  var colors = [
    '#e0422f',
    '#fdb96a',
    '#d5ed88',
    '#5ab760',
    '#006837',
    '#4c99c6',
  ].reverse()
  hLayer.setStyle({
    unit: 'px',
    radius: 20,
    gap: 0,
    altitude: 0,
    visible: false,
    height: function (index, feature) {
      var weight = feature.coordinates[0].properties.value ?? 0
      return weight * 2
    },
    topColor: function (index, feature) {
      var ranks = feature.coordinates[0].properties.value ?? 0
      return ranks < 10 ?
        colors[0] : ranks < 30 ?
          colors[1] : ranks < 60 ?
            colors[2] : ranks < 80 ?
              colors[3] : ranks < 90 ?
                colors[4] : colors[5];
    },
    sideTopColor: function (index, feature) {
      var ranks = feature.coordinates[0].properties.value ?? 0
      return ranks < 10 ?
        colors[0] : ranks < 30 ?
          colors[1] : ranks < 60 ?
            colors[2] : ranks < 80 ?
              colors[3] : ranks < 90 ?
                colors[4] : colors[5];
    },
    sideBottomColor: function (index, feature) {
      var ranks = feature.coordinates[0].properties.value ?? 0
      return ranks < 10 ?
        colors[0] : ranks < 30 ?
          colors[1] : ranks < 60 ?
            colors[2] : ranks < 80 ?
              colors[3] : ranks < 90 ?
                colors[4] : colors[5];
    }
  })

  loca.add(hLayer);

  hLayer.id = 'producationLayer'
  layerManger.add(hLayer)
}

// 加载无人机
function getDroneModel() {
  return new Promise((resolve) => {

    const loader = new GLTFLoader()
    loader.load('./static/model/drone/drone1.glb', (gltf) => {

      const model = gltf.scene.children[0]
      const size = 10.0
      model.scale.set(size, size, size)

      // 动画
      mixer = new THREE.AnimationMixer(gltf.scene);
      const action = mixer.clipAction(gltf.animations[0])
      action.setEffectiveTimeScale(guiCtrl.mixerPlaySpeed);
      action.play();

      resolve(model)
    })

  })
}

/**
 * 初始化建筑图层
 */
async function initBuildingLayer() {
  const map = getMap()

  const layer = new TilesLayer({
    id: 'buildingLayer',
    title: '村居建筑图层',
    alone: SETTING.alone,
    map,
    center: [113.531905, 22.737473], //重新调校后的中心
    zooms: [4, 30],
    interact: false,
    // tilesURL: 'http://localhost:9003/model/twQ1mVSwQ/tileset.0.json', // 村居 Cesiumlab
    tilesURL: './static/3dtiles/all/tileset.0.json', // 村居 工程内
    // altitude: 5,
    needShadow: true
  })
  layerManger.add(layer)

  layer.on('complete', ({ scene, renderer }) => {
    // 调整模型的亮度
    const aLight = new THREE.AmbientLight(0xffffff, 0.5)
    scene.add(aLight)

    const {
      lightPositionX,
      lightPositionY,
      lightPositionZ,
      cameraNear,
      caremaFar,
      cameraLeft,
      cameraRight,
      cameraTop,
      cameraBottom,
      mapSize,
      planeMaterialOpacity,
      intetity
    } = guiCtrl;

    // 平行光,增加投影
    var dLight = new THREE.DirectionalLight(0xffffff, intetity);
    dLight.position.set(lightPositionX, lightPositionY, lightPositionZ);
    dLight.castShadow = true; // 开启阴影投射
    dLight.shadow.mapSize.width = mapSize; // 增加阴影分辨率
    dLight.shadow.mapSize.height = mapSize;
    dLight.shadow.camera.near = cameraNear;
    dLight.shadow.camera.far = caremaFar;
    dLight.shadow.camera.left = cameraLeft;
    dLight.shadow.camera.right = cameraRight;
    dLight.shadow.camera.top = cameraTop;
    dLight.shadow.camera.bottom = cameraBottom;
    dLight.shadow.bias = -0.0001; // 负值将阴影稍微向外偏移
    scene.add(dLight);
    directionalLight = dLight
    // const helper = new THREE.DirectionalLightHelper(dLight, 500);
    // scene.add(helper);

    // 立方体阴影
    // const geometry = new THREE.BoxGeometry( 100, 100, 100 ); 
    // const material = new THREE.MeshStandardMaterial( {color: 0xffffff} ); 
    // const cube = new THREE.Mesh( geometry, material );  
    // cube.castShadow = true; // 启用阴影投射
    // cube.receiveShadow = true; // 接收阴影 
    // cube.position.z = 50;
    // scene.add( cube );

    // 平面阴影
    const geometry1 = new THREE.PlaneGeometry(5000, 5000);
    const material1 = new THREE.ShadowMaterial({ opacity: 1.0 })
    // const material1 = new THREE.MeshStandardMaterial({
    //   color:0xfcfcfc,
    // });

    const plane = new THREE.Mesh(geometry1, material1);
    plane.position.z = 0;
    plane.receiveShadow = true;
    scene.add(plane);

  })

}


/**
 * 切换图层的显示隐藏状态
 * @param layerId {String} 图层名称
 */
function toggleLayer(layerId) {
  switch (layerId) {
    case 'wx':
      const fn = wxLayer.getVisible() ? 'hide' : 'show'
      wxLayer[fn]()
      break;
    default:
      const layer = layerManger.findLayerById(layerId)
      if (layer) {
        const fn = (layer.visible ?? layer.getVisible()) ? 'hide' : 'show'
        layer[fn]()
      }
      break;
  }
}

/**
 * 切换边缘图层的告警状态
 */
async function toggleInvade() {

  const map = getMap()
  const borderLayer = layerManger.findLayerById('borderLayer')  

  isInvadeMode = !isInvadeMode

  // 入侵检测范围
  let ring = []
  // 入侵者路径
  let invadePath

  // 当前步数
  let invadeStep = 0
  
  if (isInvadeMode) { 
    const borderPath =  await fetchMockData('border.geojson')
    ring = borderPath.features[0].geometry.coordinates[0]
    initInvade()

    invadeClock = setInterval(() => {
      // 更新目标位置
      const pos = invadePath[invadeStep]
      invadeStep = (invadeStep + 1) % invadePath.length
      invadeMarker.setPosition(pos)
      
      // 判断为入侵，边界墙修改颜色
      const color = isInRing(pos, ring) ? '#ff0000' : '#3dfcfc'
      if(borderLayer._color !== color){
        borderLayer.setColor(color)
      }
      
    }, 1000)

  } else {
    clearInvade()
    borderLayer.setColor('#3dfcfc')
  }

  // 创建
  async function initInvade(){
    // 路径
    const {features} = await fetchMockData('invade-path.geojson')
    invadePath = features[0].geometry.coordinates[0]
    // 目标
    invadeMarker = new AMap.Marker({
      content: `<img style="width:30px;" src="./static/icons/ico-invade.png">`,
      anchor: 'bottom-center',
      offset: new AMap.Pixel(-15, -20)
    })
    map.add(invadeMarker)   
  }

  // 销毁
  function clearInvade(){
    clearInterval(invadeClock)
    invadeClock = null
    
    map.remove(invadeMarker)
    invadeMarker = null  
  }

  // 检测是否在范围内
  function isInRing (pos, ring){
    const res = AMap.GeometryUtil.isPointInRing(pos, ring)
    console.log('is in ring ', res)
    return res
  }
}

/**
 * 切换无人机第一人称状态
 */
function toggleDronView() {
  const map = getMap()
  const dronLayer = layerManger.findLayerById('dronLayer')
  const dronPathLayer = layerManger.findLayerById('dronPathLayer')
  const targetValue = isFirstView ? false : true

  if (dronLayer) {
    // 镜头跟随
    dronLayer.setCameraFollow(targetValue)
  }
  if (dronPathLayer) {
    // 显示路径
    dronPathLayer[targetValue ? 'show' : 'hide']()
  }
  map.setZoom(targetValue ? 17.5 : 15, false)

  isFirstView = targetValue
}

// 调节参数
function initGUI() {
  gui = new dat.GUI();
  var lightFolder = gui.addFolder("Directional Light");
  var shadowFolder = gui.addFolder("Shadow Camera");
  var otherFolder = gui.addFolder("Other Setting");

  // 控制光源位置
  lightFolder
    .add(directionalLight.position, "x", 0, 10000)
    .name("Position X");
  lightFolder
    .add(directionalLight.position, "y", 0, 10000)
    .name("Position Y");
  lightFolder
    .add(directionalLight.position, "z", 0, 10000)
    .name("Position Z");
  lightFolder.add(directionalLight, "intensity", 0, 3).name("Intensity");

  // 控制阴影相机参数
  shadowFolder
    .add(directionalLight.shadow.camera, "near", 0, 1)
    .step(0.01)
    .onChange(function (value) {
      directionalLight.shadow.camera.near = value;
      directionalLight.shadow.camera.updateProjectionMatrix();
    });
  shadowFolder
    .add(directionalLight.shadow.camera, "far", 10000, 1000000)
    .step(10)
    .onChange(function (value) {
      directionalLight.shadow.camera.far = value;
      directionalLight.shadow.camera.updateProjectionMatrix();
    });
  shadowFolder
    .add(directionalLight.shadow.camera, "left", -5000, 5000)
    .step(1)
    .onChange(function (value) {
      directionalLight.shadow.camera.left = value;
      directionalLight.shadow.camera.updateProjectionMatrix();
    });
  shadowFolder
    .add(directionalLight.shadow.camera, "right", -5000, 5000)
    .step(1)
    .onChange(function (value) {
      directionalLight.shadow.camera.right = value;
      directionalLight.shadow.camera.updateProjectionMatrix();
    });
  shadowFolder
    .add(directionalLight.shadow.camera, "top", -5000, 5000)
    .step(1)
    .onChange(function (value) {
      directionalLight.shadow.camera.top = value;
      directionalLight.shadow.camera.updateProjectionMatrix();
    });
  shadowFolder
    .add(directionalLight.shadow.camera, "bottom", -5000, 5000)
    .step(1)
    .onChange(function (value) {
      directionalLight.shadow.camera.bottom = value;
      directionalLight.shadow.camera.updateProjectionMatrix();
    });

  otherFolder
    .add(guiCtrl, "mapSize", 2, 2048)
    .step(1)
    .onChange(function (value) {
      directionalLight.shadow.mapSize.width = value;
      directionalLight.shadow.mapSize.height = value;
    });
  otherFolder
    .add(guiCtrl, "mixerPlaySpeed", 0, 10)
    .step(0.1)
    .onChange(function (value) {
      const action = mixer._actions[0]
      action.setEffectiveTimeScale(value);
    });

  lightFolder.open();
  shadowFolder.open();
  otherFolder.open();
}

// 回到中心
function gotoCenter() {
  getMap().setCenter(SETTING.center)
}

// 切换专题
function switchTopic(name) {
  // 专题包含哪些图层id
  const { layers, label } = SETTING.topicMap[name]

  console.log(layers)

  allLayers.forEach(({ name, id }) => {
    const layer = layerManger.findLayerById(id)
    if (!layer) {
      console.log(`${id} 不存在`)
      return
    }
    if (layers.includes(layer.id)) {
      layer.show()
      animateLayer(layer)
    } else {
      layer.hide()
    }
  })
}

/**
 * 给图层的显示增加动画效果
 */
function animateLayer(layer) {
  switch (layer.id) {
    case 'riskLayer':
      layer.addAnimate({
        key: 'height',
        value: [0, 1],
        duration: 2000,
        easing: 'BackOut',
      });
      layer.addAnimate({
        key: 'radius',
        value: [0, 1],
        duration: 2000,
        easing: 'BackOut',
        transform: 1000,
        random: true,
        delay: 5000,
      });
      break;
    case 'producationLayer':
      layer.addAnimate({
        key: 'height',
        value: [0, 1],
        duration: 500,
        easing: 'Linear',
        transform: 500,
        random: true,
        delay: 5 * 1000,
      });
      break;
    default:
      break;
  }
}

</script>

<template>
  <div ref="container" class="container"></div>
  <div class="tool">
    <!-- <button @click="toggleLayer('wx')">卫星影像</button>
    <button @click="toggleLayer('farmLayer')">农田图层</button>
    <button @click="toggleLayer('dronPathLayer')">无人机路径图层</button> -->
    <div class="btn" @click="gotoCenter()">回到中心</div>
    <div class="btn" @click="toggleInvade()">越界告警</div>
    <div class="btn" @click="toggleDronView()">无人机巡航</div>
  </div>
  <div class="layers-list">
    <div class="layers-list-h">
      图层管理
    </div>
    <div class="layers-list-b">
      <ul>
        <li v-for="(item, index) in allLayers" :key="index" @click="toggleLayer(item.id)">{{ item.name }}</li>
      </ul>
    </div>

  </div>
  <div class="topic-list">
    <div v-for="(item, key) in SETTING.topicMap" :key="key" class="item" @click="switchTopic(key)">{{ item.label }}
    </div>
    <!-- <div class="item" @click="switchTopic('security')">安全</div> -->
  </div>
  <div class="mask"></div>
</template>

<style lang="scss" type="text/scss" scoped>
.container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: #000;

}

.mask{
  position: absolute;
  width: 100%;
  height: 100%;
  pointer-events: none;
  background: radial-gradient(transparent 40%, rgb(1, 18, 38));
}

.tool {
  position: absolute;
  bottom: 2em;
  left: 50%;
  width: 50%;
  text-align: center;
  transform: translate(-50%, 0);

  .btn {
    display: inline-block;
    margin: 0 .2em;
    ;
    padding: 0.3em .6em;
    background: rgba(6, 28, 151, 0.8);
    border: 1px solid #05c5f5;
    cursor: pointer;
    color: #eefcff;
    transition: all .4s ease-in-out;

    &:hover {
      background: rgba(8, 36, 192, 0.9);
      transform: translateY(4px);
    }
  }
}

.layers-list {
  // display: none;
  position: absolute;
  top: 100px;
  right: 20px;
  z-index: 999;
  background-color: rgba(4, 16, 85, 0.5);

  &-h {
    padding: .6em 1em;
    color: #fff;
    font-weight: bold;
    border-bottom: 1px solid rgb(51, 79, 236);
  }

  &-b {
    backdrop-filter: blur(10px);

    li {
      padding: .4em 1em;
      color: #fff;
      font-size: 14px;
      cursor: pointer;

      &:hover {
        background-color: rgb(59, 59, 245);
      }

      // &.active {
      //   border-left-color: #16e242;
      // }
    }
  }
}

.topic-list {
  display: flex;
  flex-direction: row;
  position: absolute;
  bottom: 5em;
  left: 50%;
  transform: translateX(-50%);
  background-color: blue;
  color: #fff;
  text-align: center;
  width: 200px;

  .item {
    padding: .4em;
    flex: 1;
    cursor: pointer;
    font-weight: bold;
  }

  .item:hover {
    background-color: #16e242;
  }
}
</style>
<style lang="scss" type="text/scss">
.amap-info-window {
  width: 150px;
  background: #fff;
  border-radius: 3px;
  padding: 3px 7px;
  box-shadow: 0 2px 6px 0 rgba(114, 124, 245, .5);
  position: relative;
  font-size: 12px;

  p {
    line-height: 1.6em;
  }
}
</style>