<template>
  <Renderer ref="renderer" resize :orbit-ctrl="{ enableDamping: true }" pointer>
    <Camera :position="{ z: 200 }" />
    <Scene>
      <AmbientLight :color="ambientLightColor" />
      <PointLight color="#ff6000" />
      <PointLight ref="light" color="#0060ff" :intensity="0.5" />
      <PointLight color="#ff6000" :intensity="0.5" :position="{ x: 100 }" />
      <PointLight color="#0000ff" :intensity="0.5" :position="{ x: -100 }" />
      <PointLight color="#ff00ff" :intensity="0.01" :position="{ z: 100 }" />
      <InstancedMesh ref="imesh" :count="NUM_INSTANCES">
        <BoxGeometry :width="2" :height="2" :depth="starLength" />
        <StandardMaterial :props="{ transparent: true, opacity: 0.9, metalness: 0.8, roughness: 0.5 }" />
      </InstancedMesh>
      <Text :text="text" font-src="./assets/helvetiker_regular.typeface.json" align="center" :size="20" :height="1"
        :position="{ x: 0, y: 0, z: 10 }">
        <PhongMaterial />
      </Text>
    </Scene>
    <EffectComposer>
      <RenderPass />
      <UnrealBloomPass :strength="1" />
      <HalftonePass :radius="1" :scatter="0" />
    </EffectComposer>
  </Renderer>
</template>

<script>
import { Object3D, MathUtils, Vector3 } from 'three';
const { randFloat: rnd, randFloatSpread: rndFS } = MathUtils;
import {
  AmbientLight,
  BoxGeometry,
  Camera,
  EffectComposer,
  HalftonePass,
  InstancedMesh,
  PhongMaterial,
  PointLight,
  Renderer,
  RenderPass,
  StandardMaterial,
  Scene,
  Text,
  UnrealBloomPass,
} from 'troisjs';
import { Pane } from 'tweakpane'
export default {
  components: {
    AmbientLight,
    BoxGeometry,
    Camera,
    EffectComposer,
    HalftonePass,
    InstancedMesh,
    PhongMaterial,
    PointLight,
    Renderer,
    RenderPass,
    StandardMaterial,
    Scene,
    Text,
    UnrealBloomPass,
  },
  data() {
    return {
      x: 0,
      y: 0,
      z: 0,
      dirUp: true,
      dirRight: true,
      pane: {},
      attraction: 0,
      ambientLightColor: '#808080',
      vlimit: 1.2,
      text: 'Peleg.tech',
      starLength: 10,
      cursorAttraction: false
    }
  },
  setup() {
    const NUM_INSTANCES = 9000;
    const instances = [];
    const target = new Vector3();
    const dummyO = new Object3D();
    const dummyV = new Vector3();

    for (let i = 0; i < NUM_INSTANCES; i++) {
      instances.push({
        position: new Vector3(rndFS(12), rndFS(12), rndFS(12)),
        scale: rnd(0.2, 1),
        scaleZ: rnd(0.1, 1),
        velocity: new Vector3(rndFS(6), rndFS(3), rndFS(10)),
        attraction: 0,
        vlimit: 1.2 + rnd(-0.1, 0.1),
      });
    }
    return {
      NUM_INSTANCES,
      instances,
      target,
      dummyO,
      dummyV,
    };
  },
  watch: {
    attraction(val) {
      this.instances.map(instance => {
        instance.attraction = val;
        return instance;
      });
    }
  },
  methods: {
    init() {
      // init instanced mesh matrix
      for (let i = 0; i < this.NUM_INSTANCES; i++) {
        const { position, scale, scaleZ } = this.instances[i];
        this.dummyO.position.copy(position);
        this.dummyO.scale.set(scale, scale, scaleZ);
        this.dummyO.updateMatrix();
        this.imesh.setMatrixAt(i, this.dummyO.matrix);
      }
      this.imesh.instanceMatrix.needsUpdate = true;
      // animate
      this.renderer.onBeforeRender(this.animate);
    },
    animate() {
      if (this.cursorAttraction) {
        const { pointer } = this.renderer.three;
        this.target.copy(pointer.positionV3);
      }
      else
        this.target.copy({ x: this.x, y: this.y, z: this.z });
      this.light.position.copy(this.target);
      for (let i = 0; i < this.NUM_INSTANCES; i++) {
        const { position, velocity } = this.instances[i];
        let { scale, scaleZ, attraction, vlimit } = this.instances[i];
        this.dummyV.copy(this.target).sub(position).normalize().multiplyScalar(attraction);
        velocity.add(this.dummyV).clampScalar(-vlimit, vlimit);
        position.add(velocity);
        if (position.z > 300 || position.z < -1000) {
          this.instances[i] = {
            position: new Vector3(rndFS(23), rndFS(23), -999),
            scale: rnd(0.2, 1),
            scaleZ: rnd(0.1, 1),
            velocity: new Vector3(rndFS(2), rndFS(2), rndFS(100)),
            attraction: this.attraction,
            vlimit: this.vlimit + rnd(-0.1, 0.1),
          };
        }
        this.dummyO.position.copy(position);
        this.dummyO.scale.set(scale, scale, scaleZ);
        this.dummyO.lookAt(this.dummyV.copy(position).add(velocity));
        this.dummyO.updateMatrix();
        this.imesh.setMatrixAt(i, this.dummyO.matrix);
      }
      this.imesh.instanceMatrix.needsUpdate = true;
      this.movmentX(200);
      this.movmentY(50);
    },
    movmentX(range) {
      let value = this.x;
      if (this.dirRight) {
        value += 1;
      }
      else {
        value -= 1;
      }
      if (value > range)
        this.dirRight = false;
      else if (value < - range)
        this.dirRight = true;
      this.x = value;
    },
    movmentY(range) {
      let value = this.y;
      if (this.dirUp)
        value += 1;
      else
        value -= 1;
      if (value > range)
        this.dirUp = false;
      else if (value < - range)
        this.dirUp = true;
      this.y = value;
    }
  },
  computed: {
    computeCameraPosition() {
      let position = { x: 0, y: 0, z: 200 };
      position.x = this.x / 6;
      position.y = this.y / 6;
      return position;
    }
  },
  unmounted() {
    this.pane.dispose();
  },
  mounted() {
    this.renderer = this.$refs.renderer;
    this.imesh = this.$refs.imesh.mesh;
    this.light = this.$refs.light.light;
    this.pane = new Pane();
    this.pane.addInput(this, 'attraction', { min: 0, max: 1 });
    this.pane.addInput(this, 'vlimit', { min: 0, max: 12 });
    this.pane.addInput(this, 'ambientLightColor');
    this.pane.addInput(this, 'text');
    this.pane.addInput(this, 'starLength', { min: 0, max: 100 });
    this.pane.addInput(this, 'cursorAttraction');
    document.addEventListener('keydown', (e) => {
      debugger;
      if (e.code == 'KeyW' || e.code == 'ArrowUp') {
        this.keyPressed = true;
        this.vlimit = this.vlimit > 99 ? 99 : this.vlimit + 1;
        this.starLength = this.starLength > 99 ? 99 : this.starLength + 1.9;
      }
      if (e.code == 'KeyS' || e.code == 'ArrowDown') {
        this.keyPressed = true;
        this.vlimit = this.vlimit < 1 ? 1 : this.vlimit - 1;
        this.starLength = this.starLength < 1 ? 1 : this.starLength - 1;
      }
      if (e.code == 'Space') {
        this.attraction = 0;
      }
      if (e.ctrlKey) {
        this.attraction = this.attraction >= 1 ? 1 : this.attraction + 0.01;
      }
      if (e.code == 'ShiftLeft') {
        this.attraction = this.attraction >= 0.01 ? this.attraction - 0.01 : 0.01;
      }
      this.pane.refresh();
    });
    this.init();
  },
};
</script>