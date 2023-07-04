<template>
  <div>
    <div id="container" @mousemove="mouseMove">
      <div
        id='plane'
        :style="{left: state.planePos.left,top:state.planePos.top,display: state.planeDisplay}"
      >
        <p>机柜名称：{{ state.curCabinet.name }}</p>
        <p>机柜温度：{{ state.curCabinet.temperature }}°</p>
        <p>使用情况：{{ state.curCabinet.count}} / {{ state.curCabinet.capacity}}</p>
      </div>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

export default {
  data() {
    return {
      mesh: null,
      camera: null,
      scene: null,
      renderer: null,
      controls: null,
      maps: null,
      //机柜集合
      cabinets: [],
      //鼠标划入的机柜
      curCabinet: '',
      state: {
        planePos: {
          //信息面板的位置
          left: 0,
          top:0
        },
        //信息面板的可见性
        planeDisplay: 'none',
        //机柜信息
        curCabinet: {
          //名称
          name:'Loading……',
          //温度
          temperature: 0,
          //容量
          capacity: 0,
          //服务器数量
          count:0
        }
      }
    }
  },
  mounted() {
    this.maps = new Map()
    this.crtTexture("cabinet-hover.jpg")
    this.init()
  },
  methods: {
    // 初始化
    init() {
      this.createScene() // 创建场景
      this.loadGLTF() // 加载GLTF模型
      this.createLight() // 创建光源
      this.createCamera() // 创建相机
      this.createRender() // 创建渲染器
      this.createControls() // 创建控件对象
      this.render() // 渲染
      
    },
    // 创建场景
    createScene() {
      this.scene = new THREE.Scene()
    },
    // 加载GLTF模型
    loadGLTF() {
      const loader = new GLTFLoader()
      loader.load(`./models/machineRoom.gltf`, ({ scene: { children } }) => {
        console.log(...children);
        
        children.forEach((obj) => {
          const { map, color} = obj.material 
          this.changeMat(obj, map, color)
          if (obj.name.includes('cabinet')) {
            this.cabinets.push(obj)
          }
        })
        
        this.scene.add(...children)
      })
    },

    // 创建光源
    createLight() {
      // 环境光
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.1) // 创建环境光
      this.scene.add(ambientLight) // 将环境光添加到场景

      const spotLight = new THREE.SpotLight(0xffffff) // 创建聚光灯
      spotLight.position.set(150, 150, 150)
      spotLight.castShadow = true
      this.scene.add(spotLight)
    },
    // 创建相机
    createCamera() {
      const element = document.getElementById('container')
      const width = element.clientWidth // 窗口宽度
      const height = element.clientHeight // 窗口高度
      const k = width / height // 窗口宽高比
      // PerspectiveCamera( fov, aspect, near, far )
      this.camera = new THREE.PerspectiveCamera(45, k, 0.1, 1000)
      this.camera.position.set(0, 10, 15) // 设置相机位置

      this.camera.lookAt(0, 0, 0) // 设置相机方向
      this.scene.add(this.camera)
    },
    // 创建渲染器
    createRender() {
      const element = document.getElementById('container')
      this.renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
      this.renderer.setSize(element.clientWidth, element.clientHeight) // 设置渲染区域尺寸
      // this.renderer.shadowMap.enabled = true // 显示阴影
      // this.renderer.shadowMap.type = THREE.PCFSoftShadowMap
      this.renderer.setClearColor(0x3f3f3f, 1) // 设置背景颜色
      element.appendChild(this.renderer.domElement)
    },

    render() {
      // if (this.mesh) {
      //   this.mesh.rotation.z += 0.006
      // }
      this.renderer.render(this.scene, this.camera)
      requestAnimationFrame(this.render)
    },
    // 创建控件对象
    createControls() {
      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
    },
    //改变材质颜色
    changeMat(obj, map, color) {
      if (map) {
        obj.material = new THREE.MeshBasicMaterial({
          map: this.crtTexture(map.name)
        })
      } else {
        obj.material = new THREE.MeshBasicMaterial({color})
      }
    },
    crtTexture(imgName) {
      let curTexture = this.maps.get(imgName)
      if (!curTexture) {
        curTexture=new THREE.TextureLoader().load('./models/'+imgName)
        curTexture.flipY = false
        curTexture.wrapS = 1000
        curTexture.wrapT = 1000
        this.maps.set(
          imgName,
          curTexture
        )
      }
      return curTexture
    },
    //选中机柜
    selectCabinet(x, y) {
      const {cabinets,renderer,camera,maps,curCabinet}=this
      const { width, height } = renderer.domElement
      //射线投射器，可基于鼠标点和相机，在世界坐标系内建立一条射线，用于选中模型
      const raycaster = new THREE.Raycaster()
      //鼠标在裁剪空间中的点位
      const pointer = new THREE.Vector2()

      // 鼠标的canvas坐标转裁剪坐标
      pointer.set(
        (x / width) * 2 - 1,
        -(y / height) * 2 + 1,
      )
      // 基于鼠标点的裁剪坐标位和相机设置射线投射器
      raycaster.setFromCamera(
        pointer, camera
      )
      // 选择机柜
      const intersect = raycaster.intersectObjects(cabinets)[0]
      let intersectObj=intersect ? intersect.object : null
      // 若之前已有机柜被选择，且不等于当前所选择的机柜，取消之前选择的机柜的高亮
      if (curCabinet && curCabinet!== intersectObj) {
        const material =curCabinet.material
        material.setValues({
          map: maps.get('cabinet.jpg')
        })
      }
      /* 
        若当前所选对象不为空：
          触发鼠标在机柜上移动的事件。
          若当前所选对象不等于上一次所选对象：
            更新curCabinet。
            将模型高亮。
            触发鼠标划入机柜事件。
        否则若上一次所选对象存在：
          置空curCabinet。
          触发鼠标划出机柜事件。
      */
      if (intersectObj) {
        this.onMouseMoveCabinet(x,y)
        if (intersectObj !== curCabinet) {
          this.curCabinet= intersectObj
          const material = intersectObj.material
          material.setValues({
            map: maps.get('cabinet-hover.jpg')
          })
          this.onMouseOverCabinet(intersectObj)
        }
      } else if(curCabinet) {
        this.curCabinet = null
        this.onMouseOutCabinet()
      }
    },
    // 鼠标移动事件
    mouseMove({clientX,clientY}) {
      this.selectCabinet(clientX, clientY)
    },
    //鼠标划入机柜事件，参数为机柜对象
    onMouseOverCabinet (cabinet) {
      console.log(cabinet.name);
      this.state.planeDisplay = 'block'
      //基于cabinet.name 获取机柜数据
      // this.getCabinateByName(cabinet.name).then(({ data }) => {
      //   this.state.curCabinet = { ...data, name: cabinet.name }

      // });
    },
    //鼠标在机柜上移动的事件，参数为鼠标在canvas画布上的坐标位
    onMouseMoveCabinet(x,y) {
      // console.log(x,y);
      this.state.planePos.left = x + 'px'
      this.state.planePos.top = y + 'px'
    },
    //鼠标划出机柜的事件
    onMouseOutCabinet () { 
      this.state.planeDisplay = 'none'
    },
    getCabinateByName() {
       
    }
  }
}
</script>
<style>
#container {
  position: absolute;
  width: 100%;
  height: 100%;
}
#plane{
  position: absolute;
  top: 0;
  left: 0;
  background-color: rgba(0,0,0,0.5);
  color: #fff;
  padding: 0 18px;
  transform: translate(12px,-100%);
  display: none;
  text-align: left;
}
</style>
