<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue';
import * as MathFlow from '@mathflowjs/mathflow';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

type Axis = 'x' | 'y' | 'z';
type Face = 'U' | 'D' | 'F' | 'B' | 'R' | 'L';
type MoveName = 'U' | "U'" | 'R' | "R'" | 'F' | "F'" | 'L' | "L'" | 'B' | "B'" | 'D' | "D'" | 'M' | "M'" | 'E' | "E'" | 'S' | "S'";

type Vec3 = {
  x: number;
  y: number;
  z: number;
};

type Sticker = {
  id: number;
  color: string;
  solvedFace: Face;
  pos: Vec3;
  normal: Vec3;
};

type Move = {
  name: MoveName;
  axis: Axis;
  layer: number;
  turns: number;
};

type TurnAnimation = {
  pivot: THREE.Group;
  cubies: THREE.Group[];
  move: Move;
  startedAt: number;
  duration: number;
  fromAngle: number;
  targetAngle: number;
  commit: boolean;
  record: boolean;
  resolve: (completed: boolean) => void;
};

type RingDrag = {
  pointerId: number;
  axis: Axis;
  layer: number;
  pivot: THREE.Group;
  cubies: THREE.Group[];
  lastAngle: number;
  angle: number;
};

type GraphSlot = {
  id: string;
  face: Face;
  row: number;
  col: number;
  x: number;
  y: number;
};

type CircleGroup = {
  id: 'top' | 'left' | 'right';
  cx: number;
  cy: number;
  radii: number[];
};

type Point2 = {
  x: number;
  y: number;
};

const palette: Record<Face, string> = {
  U: '#f8b932',
  D: '#f7f4ec',
  F: '#f07f22',
  B: '#cf3a26',
  R: '#18559f',
  L: '#245f43',
};

const faceVectors: Record<Face, Vec3> = {
  U: { x: 0, y: 1, z: 0 },
  D: { x: 0, y: -1, z: 0 },
  F: { x: 0, y: 0, z: 1 },
  B: { x: 0, y: 0, z: -1 },
  R: { x: 1, y: 0, z: 0 },
  L: { x: -1, y: 0, z: 0 },
};

const moves: Move[] = [
  { name: 'U', axis: 'y', layer: 1, turns: -1 },
  { name: "U'", axis: 'y', layer: 1, turns: 1 },
  { name: 'R', axis: 'x', layer: 1, turns: -1 },
  { name: "R'", axis: 'x', layer: 1, turns: 1 },
  { name: 'F', axis: 'z', layer: 1, turns: -1 },
  { name: "F'", axis: 'z', layer: 1, turns: 1 },
  { name: 'L', axis: 'x', layer: -1, turns: 1 },
  { name: "L'", axis: 'x', layer: -1, turns: -1 },
  { name: 'B', axis: 'z', layer: -1, turns: 1 },
  { name: "B'", axis: 'z', layer: -1, turns: -1 },
  { name: 'D', axis: 'y', layer: -1, turns: 1 },
  { name: "D'", axis: 'y', layer: -1, turns: -1 },
];

const sliceMoves: Move[] = [
  { name: 'M', axis: 'x', layer: 0, turns: 1 },
  { name: "M'", axis: 'x', layer: 0, turns: -1 },
  { name: 'E', axis: 'y', layer: 0, turns: 1 },
  { name: "E'", axis: 'y', layer: 0, turns: -1 },
  { name: 'S', axis: 'z', layer: 0, turns: 1 },
  { name: "S'", axis: 'z', layer: 0, turns: -1 },
];
const allMoves = [...moves, ...sliceMoves];

const initialMoves: MoveName[] = ['R', 'U', "F'", 'L', 'D', 'B', 'R', "U'", 'F', "D'", 'L', 'B'];

const circleGroups: CircleGroup[] = [
  { id: 'top', cx: 280, cy: 160, radii: [85, 110, 135] },
  { id: 'left', cx: 215, cy: 275, radii: [85, 110, 135] },
  { id: 'right', cx: 345, cy: 275, radii: [85, 110, 135] },
];

// Each circle family represents a turn axis; its three radii represent layers -1, 0, and 1.
const axisCircleGroup: Record<Axis, CircleGroup['id']> = {
  x: 'left',
  y: 'top',
  z: 'right',
};

const graphFormula = createMathFlowFormula();
const graphSlots = createGraphSlots();
const cubeCanvas = ref<HTMLCanvasElement | null>(null);
const stickers = ref(createInitialCube());
const history = ref<MoveName[]>([]);
const activeMove = ref<MoveName | null>(null);
const turnProgress = ref(0);
const isBusy = ref(false);

const activeCircle = computed(() => {
  const move = allMoves.find((item) => item.name === activeMove.value);
  if (!move) return null;
  const group = groupById(axisCircleGroup[move.axis]);
  return { group, radius: group.radii[move.layer + 1] };
});
const turnDirection = computed(() => allMoves.find((move) => move.name === activeMove.value)?.turns ?? 1);
const turnAccent = ref(palette.B);

const graphNodes = computed(() =>
  graphSlots.map((slot) => {
    const sticker = stickerAt(slot.face, slot.row, slot.col);
    return {
      ...slot,
      color: sticker?.color ?? '#d7d1c8',
      stickerId: sticker?.id ?? -1,
    };
  }),
);

const rotatingGraphNodes = computed(() => {
  const circle = activeCircle.value;
  if (!circle) return [];
  return graphNodes.value.filter((node) =>
    Math.abs(Math.hypot(node.x - circle.group.cx, node.y - circle.group.cy) - circle.radius) < 0.01,
  );
});
const rotatingNodeIds = computed(() => new Set(rotatingGraphNodes.value.map((node) => node.id)));
const stationaryGraphNodes = computed(() =>
  graphNodes.value.filter((node) => !rotatingNodeIds.value.has(node.id)),
);
const rotatingNodesTransform = computed(() => {
  const circle = activeCircle.value;
  if (!circle) return undefined;
  return `rotate(${turnProgress.value * turnDirection.value * 360} ${circle.group.cx} ${circle.group.cy})`;
});

const inverseHistory = computed(() => history.value.slice(-18).reverse());
const isSolved = computed(() =>
  stickers.value.every((sticker) => sticker.color === palette[vectorToFace(sticker.normal)]),
);

let renderer: THREE.WebGLRenderer | null = null;
let camera: THREE.PerspectiveCamera | null = null;
let orbitControls: OrbitControls | null = null;
let scene: THREE.Scene | null = null;
let cubeGroup: THREE.Group | null = null;
let frameId = 0;
let resizeObserver: ResizeObserver | null = null;
let activeTurn: TurnAnimation | null = null;
let ringDrag: RingDrag | null = null;
const stickerMeshes: Array<{ mesh: THREE.Mesh; face: Face; row: number; col: number }> = [];

function createSolvedCube() {
  const result: Sticker[] = [];
  let id = 0;

  (Object.keys(faceVectors) as Face[]).forEach((face) => {
    for (let row = 0; row < 3; row += 1) {
      for (let col = 0; col < 3; col += 1) {
        result.push({
          id,
          color: palette[face],
          solvedFace: face,
          normal: { ...faceVectors[face] },
          pos: facePosition(face, row, col),
        });
        id += 1;
      }
    }
  });

  return result;
}

function createInitialCube() {
  return initialMoves.reduce((cube, name) => {
    const move = moves.find((item) => item.name === name);
    return move ? applyMove(cube, move) : cube;
  }, createSolvedCube());
}

function createMathFlowFormula() {
  const runtime = MathFlow as unknown as {
    createExpression?: (expression: string) => (scope: Record<string, number>) => number;
    compile?: (expression: string) => (scope: Record<string, number>) => number;
  };

  const compile = runtime.createExpression ?? runtime.compile;
  const toX = compile?.('cx + radius * cos(angle * pi / 180)');
  const toY = compile?.('cy + radius * sin(angle * pi / 180)');

  return (scope: { cx: number; cy: number; radius: number; angle: number }) => {
    if (toX && toY) {
      return { x: toX(scope), y: toY(scope) };
    }

    const angle = (scope.angle * Math.PI) / 180;
    return {
      x: scope.cx + scope.radius * Math.cos(angle),
      y: scope.cy + scope.radius * Math.sin(angle),
    };
  };
}

function turnArcPath(group: CircleGroup, radius: number, segment: 'shadow' | 'accent') {
  const direction = turnDirection.value;
  const start = -130 + turnProgress.value * direction * 360;
  const fromAngle = start - direction * (segment === 'shadow' ? 0 : 88);
  const toAngle = start - direction * (segment === 'shadow' ? 88 : 154);
  const from = graphFormula({ cx: group.cx, cy: group.cy, radius, angle: fromAngle });
  const to = graphFormula({ cx: group.cx, cy: group.cy, radius, angle: toAngle });
  return `M ${from.x} ${from.y} A ${radius} ${radius} 0 0 ${direction > 0 ? 0 : 1} ${to.x} ${to.y}`;
}

function createGraphSlots() {
  const slots: GraphSlot[] = [];
  const topLeft = splitIntersectionBranches(groupById('top'), groupById('left'));
  const topRight = splitIntersectionBranches(groupById('top'), groupById('right'));
  const leftRight = splitIntersectionBranches(groupById('left'), groupById('right'));

  addFaceSlots(slots, 'B', topLeft.positive);
  addFaceSlots(slots, 'F', topLeft.negative);
  addFaceSlots(slots, 'L', topRight.positive);
  addFaceSlots(slots, 'R', topRight.negative);
  addFaceSlots(slots, 'U', leftRight.negative);
  addFaceSlots(slots, 'D', leftRight.positive);

  return slots;
}

function groupById(id: CircleGroup['id']) {
  const group = circleGroups.find((item) => item.id === id);
  if (!group) throw new Error(`Missing circle group: ${id}`);
  return group;
}

function splitIntersectionBranches(first: CircleGroup, second: CircleGroup) {
  const positive: Point2[] = [];
  const negative: Point2[] = [];
  const axis = { x: second.cx - first.cx, y: second.cy - first.cy };

  first.radii.forEach((firstRadius) => {
    second.radii.forEach((secondRadius) => {
      circleIntersections(first, firstRadius, second, secondRadius).forEach((point) => {
        const side = axis.x * (point.y - first.cy) - axis.y * (point.x - first.cx);
        if (side >= 0) {
          positive.push(point);
        } else {
          negative.push(point);
        }
      });
    });
  });

  return {
    positive: sortIntersectionPoints(positive),
    negative: sortIntersectionPoints(negative),
  };
}

function circleIntersections(
  first: Pick<CircleGroup, 'cx' | 'cy'>,
  firstRadius: number,
  second: Pick<CircleGroup, 'cx' | 'cy'>,
  secondRadius: number,
) {
  const dx = second.cx - first.cx;
  const dy = second.cy - first.cy;
  const distance = Math.hypot(dx, dy);

  if (
    distance === 0 ||
    distance > firstRadius + secondRadius ||
    distance < Math.abs(firstRadius - secondRadius)
  ) {
    return [];
  }

  const along = (firstRadius ** 2 - secondRadius ** 2 + distance ** 2) / (2 * distance);
  const height = Math.sqrt(Math.max(firstRadius ** 2 - along ** 2, 0));
  const midX = first.cx + (along * dx) / distance;
  const midY = first.cy + (along * dy) / distance;
  const offsetX = (-dy * height) / distance;
  const offsetY = (dx * height) / distance;

  return [
    { x: midX + offsetX, y: midY + offsetY },
    { x: midX - offsetX, y: midY - offsetY },
  ];
}

function sortIntersectionPoints(points: Point2[]) {
  return [...points].sort((a, b) => a.y - b.y || a.x - b.x);
}

function addFaceSlots(slots: GraphSlot[], face: Face, points: Point2[]) {
  points.slice(0, 9).forEach((point, index) => {
    slots.push({
      id: slotId(face, Math.floor(index / 3), index % 3),
      face,
      row: Math.floor(index / 3),
      col: index % 3,
      x: point.x,
      y: point.y,
    });
  });
}

function slotId(face: Face, row: number, col: number) {
  return `${face}-${row}-${col}`;
}

function facePosition(face: Face, row: number, col: number): Vec3 {
  const a = col - 1;
  const b = 1 - row;

  if (face === 'U') return { x: a, y: 1, z: row - 1 };
  if (face === 'D') return { x: a, y: -1, z: 1 - row };
  if (face === 'F') return { x: a, y: b, z: 1 };
  if (face === 'B') return { x: 1 - col, y: b, z: -1 };
  if (face === 'R') return { x: 1, y: b, z: 1 - col };
  return { x: -1, y: b, z: col - 1 };
}

function stickerLocation(sticker: Sticker) {
  const face = vectorToFace(sticker.normal);
  const { x, y, z } = sticker.pos;

  if (face === 'U') return { row: z + 1, col: x + 1 };
  if (face === 'D') return { row: 1 - z, col: x + 1 };
  if (face === 'F') return { row: 1 - y, col: x + 1 };
  if (face === 'B') return { row: 1 - y, col: 1 - x };
  if (face === 'R') return { row: 1 - y, col: 1 - z };
  return { row: 1 - y, col: z + 1 };
}

function faceGrid(face: Face) {
  return [...stickers.value]
    .filter((sticker) => vectorToFace(sticker.normal) === face)
    .sort((a, b) => {
      const locationA = stickerLocation(a);
      const locationB = stickerLocation(b);
      return locationA.row - locationB.row || locationA.col - locationB.col;
    });
}

function stickerAt(face: Face, row: number, col: number) {
  return stickers.value.find((sticker) => {
    if (vectorToFace(sticker.normal) !== face) return false;
    const location = stickerLocation(sticker);
    return location.row === row && location.col === col;
  });
}

function vectorToFace(vector: Vec3): Face {
  if (vector.y === 1) return 'U';
  if (vector.y === -1) return 'D';
  if (vector.z === 1) return 'F';
  if (vector.z === -1) return 'B';
  if (vector.x === 1) return 'R';
  return 'L';
}

function faceIndex(face: Face) {
  return ['U', 'R', 'F', 'D', 'L', 'B'].indexOf(face);
}

function initThreeScene() {
  if (!cubeCanvas.value) return;

  renderer = new THREE.WebGLRenderer({
    canvas: cubeCanvas.value,
    antialias: true,
    alpha: true,
  });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setClearColor(0xfbf7ef, 0);

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(35, 1, 0.1, 100);
  camera.position.set(6.6, 3.9, 7.1);
  camera.lookAt(0, 0, 0);

  orbitControls = new OrbitControls(camera, renderer.domElement);
  orbitControls.enableDamping = true;
  orbitControls.dampingFactor = 0.09;
  orbitControls.enablePan = false;
  orbitControls.minDistance = 7;
  orbitControls.maxDistance = 16;
  orbitControls.minPolarAngle = 0.12;
  orbitControls.maxPolarAngle = Math.PI - 0.12;

  cubeGroup = new THREE.Group();
  scene.add(cubeGroup);

  const ambient = new THREE.HemisphereLight(0xffffff, 0xc6c3b9, 1.9);
  scene.add(ambient);

  const keyLight = new THREE.DirectionalLight(0xffffff, 1.8);
  keyLight.position.set(5, 8, 7);
  scene.add(keyLight);

  const fillLight = new THREE.DirectionalLight(0xffffff, 0.8);
  fillLight.position.set(-6, 1, 4);
  scene.add(fillLight);

  buildThreeCube();
  updateThreeCube();
  resizeThreeScene();

  resizeObserver = new ResizeObserver(resizeThreeScene);
  resizeObserver.observe(cubeCanvas.value);

  const animate = (time: number) => {
    frameId = window.requestAnimationFrame(animate);
    advanceTurn(time);
    orbitControls?.update();
    if (renderer && scene && camera) {
      renderer.render(scene, camera);
    }
  };

  animate(performance.now());
}

function resizeThreeScene() {
  if (!cubeCanvas.value || !renderer || !camera) return;

  const { width, height } = cubeCanvas.value.getBoundingClientRect();
  renderer.setSize(width, height, false);
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
}

function roundedSquareGeometry(size: number, radius: number) {
  const half = size / 2;
  const shape = new THREE.Shape();
  shape.moveTo(-half + radius, -half);
  shape.lineTo(half - radius, -half);
  shape.quadraticCurveTo(half, -half, half, -half + radius);
  shape.lineTo(half, half - radius);
  shape.quadraticCurveTo(half, half, half - radius, half);
  shape.lineTo(-half + radius, half);
  shape.quadraticCurveTo(-half, half, -half, half - radius);
  shape.lineTo(-half, -half + radius);
  shape.quadraticCurveTo(-half, -half, -half + radius, -half);
  return new THREE.ShapeGeometry(shape, 5);
}

function positionFacePatch(mesh: THREE.Mesh, face: Face, distance: number) {
  const normal = faceVectors[face];
  mesh.position.set(normal.x * distance, normal.y * distance, normal.z * distance);
  if (face === 'U') mesh.rotation.x = -Math.PI / 2;
  if (face === 'D') mesh.rotation.x = Math.PI / 2;
  if (face === 'R') mesh.rotation.y = Math.PI / 2;
  if (face === 'L') mesh.rotation.y = -Math.PI / 2;
  if (face === 'B') mesh.rotation.y = Math.PI;
}

function buildThreeCube() {
  if (!cubeGroup) return;

  const bodyGeometry = new THREE.BoxGeometry(0.994, 0.994, 0.994);
  const edgeGeometry = new THREE.EdgesGeometry(bodyGeometry);
  const trimGeometry = roundedSquareGeometry(0.86, 0.08);
  const stickerGeometry = roundedSquareGeometry(0.825, 0.07);
  const bodyMaterial = new THREE.MeshStandardMaterial({ color: 0xf2f4f1, roughness: 0.78 });
  const edgeMaterial = new THREE.LineBasicMaterial({ color: 0x899492, transparent: true, opacity: 0.9 });
  const trimMaterial = new THREE.MeshStandardMaterial({ color: 0xb8c0bd, roughness: 0.85 });
  const cubies = new Map<string, THREE.Group>();
  const key = (pos: Vec3) => `${pos.x},${pos.y},${pos.z}`;

  for (let x = -1; x <= 1; x += 1) {
    for (let y = -1; y <= 1; y += 1) {
      for (let z = -1; z <= 1; z += 1) {
        const cubie = new THREE.Group();
        cubie.userData.home = { x, y, z } satisfies Vec3;
        cubie.add(new THREE.Mesh(bodyGeometry, bodyMaterial));
        cubie.add(new THREE.LineSegments(edgeGeometry, edgeMaterial));
        cubie.position.set(x, y, z);
        cubeGroup.add(cubie);
        cubies.set(key({ x, y, z }), cubie);
      }
    }
  }

  (Object.keys(faceVectors) as Face[]).forEach((face) => {
    for (let row = 0; row < 3; row += 1) {
      for (let col = 0; col < 3; col += 1) {
        const cubie = cubies.get(key(facePosition(face, row, col)));
        if (!cubie) continue;

        const trim = new THREE.Mesh(trimGeometry, trimMaterial);
        positionFacePatch(trim, face, 0.499);
        cubie.add(trim);

        const mesh = new THREE.Mesh(
          stickerGeometry,
          new THREE.MeshStandardMaterial({ color: palette[face], roughness: 0.82 }),
        );
        positionFacePatch(mesh, face, 0.502);
        cubie.add(mesh);
        stickerMeshes.push({ mesh, face, row, col });
      }
    }
  });
}

function updateThreeCube() {
  stickerMeshes.forEach(({ mesh, face, row, col }) => {
    const color = stickerAt(face, row, col)?.color;
    if (color) (mesh.material as THREE.MeshStandardMaterial).color.set(color);
  });
}

function rotateVec(vector: Vec3, axis: Axis, turns: number): Vec3 {
  const t = ((turns % 4) + 4) % 4;
  let next = { ...vector };

  for (let index = 0; index < t; index += 1) {
    if (axis === 'x') next = { x: next.x, y: -next.z, z: next.y };
    if (axis === 'y') next = { x: next.z, y: next.y, z: -next.x };
    if (axis === 'z') next = { x: -next.y, y: next.x, z: next.z };
  }

  return next;
}

function applyMove(cube: Sticker[], move: Move) {
  return cube.map((sticker) => {
    if (sticker.pos[move.axis] !== move.layer) return sticker;
    return {
      ...sticker,
      pos: rotateVec(sticker.pos, move.axis, move.turns),
      normal: rotateVec(sticker.normal, move.axis, move.turns),
    };
  });
}

function moveForLayer(axis: Axis, layer: number, direction: number) {
  const move = allMoves.find((item) =>
    item.axis === axis && item.layer === layer && item.turns === direction,
  );
  if (!move) throw new Error(`Missing move for ${axis} layer ${layer}`);
  return move;
}

function axisForCircleGroup(id: CircleGroup['id']) {
  const axis = (Object.keys(axisCircleGroup) as Axis[]).find((item) => axisCircleGroup[item] === id);
  if (!axis) throw new Error(`Missing axis for circle group ${id}`);
  return axis;
}

function setTurnAccent(move: Move) {
  const group = groupById(axisCircleGroup[move.axis]);
  const radius = group.radii[move.layer + 1];
  const head = graphFormula({ cx: group.cx, cy: group.cy, radius, angle: -130 - move.turns * 154 });
  const nearest = graphNodes.value
    .filter((node) => Math.abs(Math.hypot(node.x - group.cx, node.y - group.cy) - radius) < 0.01)
    .reduce<{ color: string; distance: number } | null>((best, node) => {
      const distance = Math.hypot(node.x - head.x, node.y - head.y);
      return !best || distance < best.distance ? { color: node.color, distance } : best;
    }, null);
  turnAccent.value = nearest?.color === palette.D ? '#9aa29e' : nearest?.color ?? palette.B;
}

function createTurnPivot(axis: Axis, layer: number) {
  const root = cubeGroup;
  if (!root) return null;

  const cubies = root.children.filter(
    (child): child is THREE.Group =>
      child instanceof THREE.Group && (child.userData.home as Vec3 | undefined)?.[axis] === layer,
  );
  const pivot = new THREE.Group();
  root.add(pivot);
  cubies.forEach((cubie) => pivot.add(cubie));
  return { pivot, cubies };
}

function restoreTurnCubies(pivot: THREE.Group, cubies: THREE.Group[]) {
  cubies.forEach((cubie) => {
    pivot.remove(cubie);
    cubeGroup?.add(cubie);
    const home = cubie.userData.home as Vec3;
    cubie.position.set(home.x, home.y, home.z);
    cubie.rotation.set(0, 0, 0);
  });
  cubeGroup?.remove(pivot);
}

function animateLayerTurn(move: Move, record: boolean): Promise<boolean> {
  turnProgress.value = 0;
  setTurnAccent(move);
  const turn = createTurnPivot(move.axis, move.layer);
  if (!turn) {
    stickers.value = applyMove(stickers.value, move);
    if (record) history.value = [...history.value, move.name];
    return Promise.resolve(true);
  }

  return new Promise((resolve) => {
    activeTurn = {
      ...turn, move, startedAt: performance.now(), duration: 460,
      fromAngle: 0, targetAngle: move.turns * Math.PI / 2,
      commit: true, record, resolve,
    };
  });
}

function advanceTurn(time: number) {
  const turn = activeTurn;
  if (!turn) return;

  const progress = Math.min((time - turn.startedAt) / turn.duration, 1);
  const eased = progress < 0.5
    ? 4 * progress ** 3
    : 1 - (-2 * progress + 2) ** 3 / 2;
  const angle = turn.fromAngle + (turn.targetAngle - turn.fromAngle) * eased;
  turn.pivot.rotation[turn.move.axis] = angle;
  turnProgress.value = Math.abs(angle) / (Math.PI / 2);
  if (progress < 1) return;

  restoreTurnCubies(turn.pivot, turn.cubies);
  activeTurn = null;

  if (turn.commit) {
    stickers.value = applyMove(stickers.value, turn.move);
    updateThreeCube();
    if (turn.record) history.value = [...history.value, turn.move.name];
  }
  turn.resolve(turn.commit);
}

function ringPointerAngle(event: PointerEvent, group: CircleGroup) {
  const svg = (event.currentTarget as SVGCircleElement).ownerSVGElement;
  const matrix = svg?.getScreenCTM();
  if (!matrix) return null;
  const point = new DOMPoint(event.clientX, event.clientY).matrixTransform(matrix.inverse());
  return Math.atan2(point.y - group.cy, point.x - group.cx);
}

function startRingDrag(event: PointerEvent, group: CircleGroup, ringIndex: number) {
  if (isBusy.value || (event.pointerType === 'mouse' && event.button !== 0)) return;
  const angle = ringPointerAngle(event, group);
  if (angle === null) return;
  const axis = axisForCircleGroup(group.id);
  const layer = ringIndex - 1;
  const turn = createTurnPivot(axis, layer);
  if (!turn) return;

  event.preventDefault();
  (event.currentTarget as SVGCircleElement).setPointerCapture(event.pointerId);
  isBusy.value = true;
  activeMove.value = moveForLayer(axis, layer, 1).name;
  turnProgress.value = 0;
  setTurnAccent(moveForLayer(axis, layer, 1));
  ringDrag = { ...turn, pointerId: event.pointerId, axis, layer, lastAngle: angle, angle: 0 };
}

function moveRingDrag(event: PointerEvent, group: CircleGroup) {
  const drag = ringDrag;
  if (!drag || drag.pointerId !== event.pointerId) return;
  const angle = ringPointerAngle(event, group);
  if (angle === null) return;
  const delta = Math.atan2(Math.sin(angle - drag.lastAngle), Math.cos(angle - drag.lastAngle));
  drag.lastAngle = angle;
  drag.angle = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, drag.angle + delta));
  drag.pivot.rotation[drag.axis] = drag.angle;
  turnProgress.value = Math.abs(drag.angle) / (Math.PI / 2);

  const move = moveForLayer(drag.axis, drag.layer, drag.angle < 0 ? -1 : 1);
  if (activeMove.value !== move.name) {
    activeMove.value = move.name;
    setTurnAccent(move);
  }
}

function finishRingDrag(event: PointerEvent, cancelled = false) {
  const drag = ringDrag;
  if (!drag || drag.pointerId !== event.pointerId) return;
  ringDrag = null;
  const target = event.currentTarget as SVGCircleElement;
  if (target.hasPointerCapture(event.pointerId)) target.releasePointerCapture(event.pointerId);

  const commit = !cancelled && Math.abs(drag.angle) >= Math.PI / 9;
  const move = moveForLayer(drag.axis, drag.layer, drag.angle < 0 ? -1 : 1);
  const targetAngle = commit ? move.turns * Math.PI / 2 : 0;
  if (Math.abs(drag.angle - targetAngle) < 0.001) {
    restoreTurnCubies(drag.pivot, drag.cubies);
    if (commit) {
      stickers.value = applyMove(stickers.value, move);
      updateThreeCube();
      history.value = [...history.value, move.name];
    }
    activeMove.value = null;
    turnProgress.value = 0;
    isBusy.value = false;
    return;
  }

  activeMove.value = move.name;
  const remaining = Math.abs(targetAngle - drag.angle) / (Math.PI / 2);
  void new Promise<boolean>((resolve) => {
    activeTurn = {
      pivot: drag.pivot, cubies: drag.cubies, move,
      startedAt: performance.now(), duration: Math.max(120, 460 * remaining),
      fromAngle: drag.angle, targetAngle, commit, record: true, resolve,
    };
  }).then(() => {
    activeMove.value = null;
    turnProgress.value = 0;
    isBusy.value = false;
  });
}

function rotateRing(id: CircleGroup['id'], ringIndex: number, direction: number) {
  const move = moveForLayer(axisForCircleGroup(id), ringIndex - 1, direction);
  void runMove(move);
}

async function runMove(move: Move, shouldRecord = true) {
  if (isBusy.value) return false;
  isBusy.value = true;
  activeMove.value = move.name;

  try {
    return await animateLayerTurn(move, shouldRecord);
  } finally {
    isBusy.value = false;
    activeMove.value = null;
    turnProgress.value = 0;
  }
}

async function scramble() {
  if (isBusy.value) return;
  isBusy.value = true;
  const sequence = Array.from({ length: 16 }, () => moves[Math.floor(Math.random() * moves.length)]);

  try {
    for (const move of sequence) {
      activeMove.value = move.name;
      if (!(await animateLayerTurn(move, true))) break;
    }
  } finally {
    activeMove.value = null;
    turnProgress.value = 0;
    isBusy.value = false;
  }
}

function resetCube() {
  if (isBusy.value) return;
  stickers.value = createSolvedCube();
  history.value = [];
}

async function undoLast() {
  const last = history.value.at(-1);
  if (!last || isBusy.value) return;

  const inverseName = (last.endsWith("'") ? last.slice(0, 1) : `${last}'`) as MoveName;
  const move = allMoves.find((item) => item.name === inverseName);
  if (!move) return;

  if (await runMove(move, false)) history.value = history.value.slice(0, -1);
}

onMounted(() => {
  nextTick(initThreeScene);
});

watch(stickers, () => updateThreeCube(), { deep: true });

onBeforeUnmount(() => {
  if (frameId) window.cancelAnimationFrame(frameId);
  orbitControls?.dispose();
  activeTurn?.resolve(false);
  activeTurn = null;
  ringDrag = null;
  turnProgress.value = 0;
  resizeObserver?.disconnect();
  const geometries = new Set<THREE.BufferGeometry>();
  const materials = new Set<THREE.Material>();
  cubeGroup?.traverse((object) => {
    if (!(object instanceof THREE.Mesh || object instanceof THREE.LineSegments)) return;
    geometries.add(object.geometry);
    const objectMaterials = Array.isArray(object.material) ? object.material : [object.material];
    objectMaterials.forEach((material) => materials.add(material));
  });
  geometries.forEach((geometry) => geometry.dispose());
  materials.forEach((material) => material.dispose());
  renderer?.dispose();
});
</script>

<template>
  <main class="app-shell">
    <section class="stage">
      <div class="cube-panel" aria-label="平面魔方联动演示">
        <canvas ref="cubeCanvas" class="cube-canvas" />
      </div>

      <svg class="graph-panel" viewBox="55 -25 480 460" role="img" aria-label="mathflow 平面图">
        <g class="circle-web">
          <template v-for="group in circleGroups" :key="group.id">
            <circle
              v-for="radius in group.radii"
              :key="`${group.id}-${radius}`"
              :cx="group.cx"
              :cy="group.cy"
              :r="radius"
            />
          </template>
        </g>
        <g v-if="activeCircle" class="circle-turn">
          <path class="circle-turn-shadow" :d="turnArcPath(activeCircle.group, activeCircle.radius, 'shadow')" />
          <path class="circle-turn-accent" :stroke="turnAccent" :d="turnArcPath(activeCircle.group, activeCircle.radius, 'accent')" />
        </g>
        <g class="nodes">
          <circle
            v-for="node in stationaryGraphNodes"
            :key="node.id"
            class="graph-node"
            :cx="node.x"
            :cy="node.y"
            r="9"
            :fill="node.color"
          />
          <g v-if="activeCircle" class="rotating-nodes" :transform="rotatingNodesTransform">
            <circle
              v-for="node in rotatingGraphNodes"
              :key="node.id"
              class="graph-node active"
              :cx="node.x"
              :cy="node.y"
              r="9"
              :fill="node.color"
            />
          </g>
        </g>
        <g class="circle-hit-areas">
          <template v-for="group in circleGroups" :key="group.id">
            <circle
              v-for="(radius, index) in group.radii"
              :key="`${group.id}-${radius}`"
              class="circle-hit"
              :data-ring="`${group.id}-${index}`"
              :cx="group.cx"
              :cy="group.cy"
              :r="radius"
              tabindex="0"
              role="button"
              :aria-label="`${group.id} circle ${index + 1}`"
              @pointerdown="startRingDrag($event, group, index)"
              @pointermove="moveRingDrag($event, group)"
              @pointerup="finishRingDrag($event)"
              @pointercancel="finishRingDrag($event, true)"
              @keydown.left.prevent="rotateRing(group.id, index, -1)"
              @keydown.right.prevent="rotateRing(group.id, index, 1)"
            >
              <title>{{ group.id }} circle {{ index + 1 }}</title>
            </circle>
          </template>
        </g>
      </svg>
    </section>

    <section class="control-strip">
      <div class="move-pad">
        <button
          v-for="move in moves"
          :key="move.name"
          type="button"
          :class="{ selected: activeMove === move.name }"
          :disabled="isBusy"
          @click="runMove(move)"
        >
          {{ move.name }}
        </button>
      </div>

      <div class="utility-pad">
        <button type="button" :disabled="isBusy" @click="scramble">Scramble</button>
        <button type="button" :disabled="isBusy || !history.length" @click="undoLast">Undo</button>
        <button type="button" :disabled="isBusy" @click="resetCube">Reset</button>
      </div>
    </section>

    <section class="status-bar">
      <span>贴纸状态：{{ isSolved ? '复原' : '已打乱' }}</span>
      <span>最近路径：{{ inverseHistory.join(' ') || '等待第一次转动' }}</span>
    </section>
  </main>
</template>
