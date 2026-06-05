<template>
  <div class="ifc-viewer-container">
    <div id="ifc-container" class="h-full" aria-label="IFC Model Viewer"></div>

    <!-- WebGL not available -->
    <div v-if="webglError" class="empty-state">
      <div class="empty-state-card error-card">
        <div class="error-icon">⚠️</div>
        <h2>WebGL Unavailable</h2>
        <p>
          Chrome has blocked GPU access for this virtual machine (VMware SVGA II).
          Enable the software renderer with one of these steps:
        </p>
        <ol class="fix-steps">
          <li>
            Open <strong>chrome://flags/#ignore-gpu-blocklist</strong> in a new tab,
            set it to <strong>Enabled</strong>, then click <strong>Relaunch</strong>.
          </li>
          <li>
            Or restart Chrome from the terminal with:<br>
            <code>google-chrome --ignore-gpu-blocklist --use-gl=swiftshader</code>
          </li>
        </ol>
      </div>
    </div>

    <!-- Empty state -->
    <div v-else-if="!modelLoaded && !loading" class="empty-state">
      <div class="empty-state-card">
        <svg class="empty-icon" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64" fill="none">
          <rect x="8" y="12" width="48" height="40" rx="4" stroke="#4a90d9" stroke-width="3"/>
          <path d="M8 22h48" stroke="#4a90d9" stroke-width="2"/>
          <path d="M22 12v10" stroke="#4a90d9" stroke-width="2"/>
          <path d="M42 12v10" stroke="#4a90d9" stroke-width="2"/>
          <path d="M24 36l6 6 10-12" stroke="#4a90d9" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
        <h2>IFC Viewer</h2>
        <p>Select a local <code>.ifc</code> file to load and view the 3D model.</p>
        <button class="load-btn primary" @click="triggerFilePicker">
          <span class="btn-icon">📂</span> Load IFC File
        </button>
      </div>
    </div>

    <!-- Loading progress overlay -->
    <div v-if="loading" class="loading-overlay">
      <div class="loading-card">
        <div class="loading-title">Loading IFC Model…</div>
        <div v-if="loadingFileName" class="loading-filename">{{ loadingFileName }}</div>
        <div class="progress-track">
          <div class="progress-fill" :style="{ width: loadingProgress + '%' }"></div>
        </div>
        <div class="progress-pct">{{ Math.round(loadingProgress) }}%</div>
      </div>
    </div>

    <!-- Toolbar shown after model is loaded -->
    <div v-if="modelLoaded && !loading" class="toolbar">
      <button class="load-btn secondary" @click="triggerFilePicker">Load another file</button>
      <button class="load-btn accent" @click="handleHighlightButtonClick" :disabled="!world || !highlighter">
        Highlight Element
      </button>
    </div>

    <!-- Element properties panel -->
    <div v-if="selectedElement" class="element-properties">
      <h2>Properties</h2>
      <pre>{{ JSON.stringify(selectedElement, null, 2) }}</pre>
    </div>

    <!-- Hidden file input -->
    <input type="file" ref="fileInput" accept=".ifc" style="display:none" @change="onFileSelected" />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue';
import * as OBC from '@thatopen/components';
import * as OBCF from "@thatopen/components-front";
import * as BUI from '@thatopen/ui';
import * as CUI from '@thatopen/ui-obc';
import * as THREE from "three";
import * as FRAGS from "@thatopen/fragments";
import * as OBF from "@thatopen/components-front";

const props = defineProps({
  URL:         { type: String, default: '' },
  META:        { type: String, default: '' },
  fileName:    { type: String, default: '' },
  highlighId:  { type: Number, default: -1 }
});

const world      = ref<OBC.SimpleWorld<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer> | null>(null);
const loading    = ref(false);
const loadingProgress = ref(0);
const loadingFileName = ref('');
const modelLoaded = ref(false);
const selectedElement = ref<any>(null);
const fileInput  = ref<HTMLInputElement | null>(null);
const webglError = ref(false);

let resizeObserver: ResizeObserver | null = null;
let model: FRAGS.FragmentsGroup | null = null;
let ifcData: any[] = [];
let highlighter: OBF.Highlighter;
let relationsTree: any;
let components: OBC.Components;

const initializeWorld = () => {
  BUI.Manager.init();
  const container = document.getElementById('ifc-container');
  if (!container) return;

  // Build a plain-div layout so the renderer container is in the DOM immediately,
  // with no custom-element timing issues (bim-viewport connectedCallback race).
  const layoutEl = document.createElement('div');
  layoutEl.style.cssText = 'display:flex;width:100%;height:100%;overflow:hidden;';

  const panelEl = document.createElement('div');
  panelEl.style.cssText = 'width:300px;min-width:300px;overflow-y:auto;background:#1a2133;';

  const viewportEl = document.createElement('div');
  viewportEl.style.cssText = 'flex:1;position:relative;overflow:hidden;';

  layoutEl.appendChild(panelEl);
  layoutEl.appendChild(viewportEl);
  container.appendChild(layoutEl); // viewportEl is in the DOM before renderer is created

  components = new OBC.Components();

  const worlds = components.get(OBC.Worlds);
  const newWorld = worlds.create<OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer>();
  newWorld.scene = new OBC.SimpleScene(components);

  // VMware SVGA II / GPU-blocked environments need powerPreference:'low-power'
  // to force Chrome's SwiftShader software renderer instead of the blocked GPU.
  try {
    newWorld.renderer = new OBC.SimpleRenderer(components, viewportEl, { powerPreference: 'low-power' });
  } catch {
    try {
      newWorld.renderer = new OBC.SimpleRenderer(components, viewportEl);
    } catch {
      webglError.value = true;
      components.dispose();
      return;
    }
  }

  newWorld.camera = new OBC.SimpleCamera(components);

  components.init();
  newWorld.camera.controls.setLookAt(12, 6, 8, 0, 0, -10);
  newWorld.scene.setup();

  const cullers = components.get(OBC.Cullers);
  const culler  = cullers.create(newWorld);
  culler.config.threshold = 200;
  culler.needsUpdate = true;
  newWorld.camera.controls.addEventListener('controlend', () => { culler.needsUpdate = true; });

  const grids = components.get(OBC.Grids);
  grids.create(newWorld);

  const fragmentIfcLoader = components.get(OBC.IfcLoader);
  fragmentIfcLoader.settings.wasm = {
    path: 'https://unpkg.com/web-ifc@0.0.59/',
    absolute: true
  };
  fragmentIfcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;
  (fragmentIfcLoader.settings.webIfc as any).OPTIMIZE_PROFILES = true;

  const [tree] = CUI.tables.relationsTree({ components, models: [] });
  relationsTree = tree;
  relationsTree.preserveStructureOnFilter = true;

  highlighter = components.get(OBF.Highlighter);
  highlighter.setup({ world: newWorld, selectionColor: new THREE.Color(0xf0ff0f) });
  highlighter.zoomToSelection = true;

  highlighter.events.select.onHighlight.add((selection: any) => {
    if (Object.keys(selection).length > 0) {
      const firstKey = Object.keys(selection)[0];
      const expressID = parseInt(selection[firstKey].values().next().value);
      selectedElement.value = ifcData.find((item: any) => item.expressID === expressID) ?? null;
    } else {
      selectedElement.value = null;
    }
  });

  // Build side panel with bim-panel (works standalone, no bim-grid needed)
  const panel = BUI.Component.create(() => {
    const onSearch = (e: Event) => {
      const input = e.target as BUI.TextInput;
      relationsTree.queryString = input.value;
    };
    return BUI.html`
      <bim-panel label="Relations Tree" style="height:100%;width:100%;">
        <bim-panel-section label="Model Tree">
          <bim-text-input @input=${onSearch} placeholder="Search…" debounce="200"></bim-text-input>
          ${relationsTree}
        </bim-panel-section>
      </bim-panel>
    `;
  });
  panelEl.appendChild(panel);

  world.value = newWorld;

  resizeObserver = new ResizeObserver(
    debounce(() => {
      if (newWorld.renderer && newWorld.camera) {
        newWorld.renderer.resize();
        newWorld.camera.updateAspect();
      }
    }, 100)
  );
  resizeObserver.observe(viewportEl);
};

// Load model from a Uint8Array buffer (used by both URL and file-picker paths)
const loadBuffer = async (buffer: Uint8Array, meta?: string) => {
  if (!world.value) return;

  loading.value = true;
  loadingProgress.value = 0;
  selectedElement.value = null;

  // Dispose old model if present
  if (model) {
    world.value.scene.three.remove(model);
    const fm = components.get(OBC.FragmentsManager);
    fm.dispose();
    model = null;
    ifcData = [];
    modelLoaded.value = false;
    if (relationsTree) {
      relationsTree.models = [];
      relationsTree.update?.();
    }
  }

  try {
    const fragmentIfcLoader = components.get(OBC.IfcLoader);

    loadingProgress.value = 15;

    // Simulate smooth progress while WASM parsing runs (the heavy step)
    const progressTimer = setInterval(() => {
      if (loadingProgress.value < 78) {
        loadingProgress.value = Math.min(78, loadingProgress.value + 1.5);
      }
    }, 150);

    model = await fragmentIfcLoader.load(buffer);
    clearInterval(progressTimer);

    loadingProgress.value = 85;
    world.value.scene.three.add(model);

    const indexer = components.get(OBC.IfcRelationsIndexer);
    await indexer.process(model);
    loadingProgress.value = 93;

    // Load metadata if a META url is provided
    if (meta) {
      const ifcResponse = await fetch(meta);
      const rawIfcData  = await ifcResponse.json();
      ifcData = Array.isArray(rawIfcData) ? rawIfcData : Object.values(rawIfcData);
    }

    if (relationsTree) {
      relationsTree.models = [model];
      relationsTree.update?.();
    }

    loadingProgress.value = 100;
    modelLoaded.value = true;
  } catch (error) {
    console.error('Error loading IFC model:', error);
  } finally {
    // Brief pause so the user sees 100% before the overlay disappears
    setTimeout(() => { loading.value = false; }, 400);
  }
};

// Load from URL prop
const loadModel = async () => {
  if (!world.value || !props.URL) return;
  loadingFileName.value = props.fileName || props.URL.split('/').pop() || 'model.ifc';
  const response = await fetch(props.URL);
  const data     = await response.arrayBuffer();
  await loadBuffer(new Uint8Array(data), props.META || undefined);
};

// File picker helpers
const triggerFilePicker = () => fileInput.value?.click();

const onFileSelected = async (e: Event) => {
  const input = e.target as HTMLInputElement;
  const file  = input.files?.[0];
  if (!file) return;

  loadingFileName.value = file.name;
  const arrayBuffer = await file.arrayBuffer();
  await loadBuffer(new Uint8Array(arrayBuffer));

  // Reset input so re-selecting the same file still fires change
  input.value = '';
};

// Highlight helpers
const handleHighlightButtonClick = () => {
  if (props.highlighId >= 0) highlightByExpressID(props.highlighId);
};

const highlightByExpressID = (expressID: number) => {
  if (!highlighter || !model || !world.value) {
    console.warn('Highlighter, model, or world not initialized');
    return;
  }
  highlighter.zoomToSelection = true;
  try {
    model.items.forEach((fragment) => {
      const id: number = Array.from(fragment.ids)[0];
      if (id.toString() === expressID.toString()) {
        const mesh = fragment.mesh;
        const boundingBox = new THREE.Box3().setFromObject(mesh);
        const center  = boundingBox.getCenter(new THREE.Vector3());
        const size    = boundingBox.getSize(new THREE.Vector3());
        const maxDim  = Math.max(size.x, size.y, size.z);
        const distance = maxDim * 0.5;
        world.value?.camera.controls.setLookAt(
          center.x + distance, center.y + distance, center.z + distance,
          center.x, center.y, center.z, true
        );
        const fragmentIdMap: any = { [fragment.id]: new Set([expressID]) };
        highlighter.events.select.onHighlight.trigger(fragmentIdMap);
        changeFragmentColor(fragment.id, 0xf0ff0f);
      }
    });
  } catch (error) {
    console.error('Error during highlight and zoom:', error);
  }
};

const changeFragmentColor = (fragmentId: string, color: number) => {
  model?.items.forEach((fragment) => {
    if (fragment.id !== fragmentId) return;
    const mesh = fragment.mesh;
    if (!mesh) return;
    if (Array.isArray(mesh.material)) {
      mesh.material = mesh.material.map((mat) => {
        const m: any = mat.clone();
        if (m.color) m.color.set(color);
        return m;
      });
    } else {
      const m: any = mesh.material;
      if (m.color) m.color.set(color);
    }
  });
  world.value?.scene.three.updateMatrixWorld(true);
};

const debounce = (func: (...args: any[]) => void, wait: number) => {
  let timeout: ReturnType<typeof setTimeout> | null = null;
  return (...args: any[]) => {
    if (timeout) clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
};

onMounted(() => {
  initializeWorld();
  if (props.URL) loadModel();
});

onUnmounted(() => {
  resizeObserver?.disconnect();
  // components.dispose() iterates every registered component and calls
  // dispose() on each — including SimpleRenderer which calls forceContextLoss(),
  // releasing the WebGL context back to the browser pool.
  components?.dispose();
});

watch(() => props.URL, loadModel);
</script>

<style scoped>
.ifc-viewer-container {
  position: relative;
  height: 100%;
  width: 100%;
}

.h-full { height: 100%; }

/* ── Empty state ── */
.empty-state {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(15, 20, 30, 0.72);
  backdrop-filter: blur(4px);
  z-index: 10;
}

.empty-state-card {
  background: #1a2133;
  border: 1px solid #2d3f5c;
  border-radius: 16px;
  padding: 48px 56px;
  text-align: center;
  color: #e0eaf8;
  max-width: 380px;
}

.empty-icon {
  width: 64px;
  height: 64px;
  margin-bottom: 20px;
}

.empty-state-card h2 {
  margin: 0 0 8px;
  font-size: 1.6rem;
  font-weight: 600;
  color: #fff;
}

.empty-state-card p {
  margin: 0 0 28px;
  font-size: 0.95rem;
  color: #8fa8cc;
  line-height: 1.5;
}

.empty-state-card code {
  background: #243047;
  padding: 1px 6px;
  border-radius: 4px;
  font-size: 0.9em;
}

.error-card { border-color: #7a3030; max-width: 460px; text-align: left; }
.error-icon { font-size: 2.5rem; margin-bottom: 12px; text-align: center; }
.error-card h2 { text-align: center; color: #f87171; margin-bottom: 12px; }
.error-card p { color: #9db5d0; font-size: 0.9rem; line-height: 1.6; margin-bottom: 14px; }
.fix-steps { color: #8fa8cc; font-size: 0.875rem; line-height: 1.8; padding-left: 18px; margin: 0; }
.fix-steps li { margin-bottom: 10px; }
.fix-steps strong { color: #c8daf2; }
.fix-steps code {
  display: inline-block;
  background: #0d1520;
  border: 1px solid #2d3f5c;
  border-radius: 4px;
  padding: 4px 8px;
  font-size: 0.8rem;
  margin-top: 4px;
  word-break: break-all;
}

/* ── Loading overlay ── */
.loading-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(10, 14, 22, 0.80);
  backdrop-filter: blur(6px);
  z-index: 20;
}

.loading-card {
  background: #1a2133;
  border: 1px solid #2d3f5c;
  border-radius: 16px;
  padding: 40px 48px;
  text-align: center;
  color: #e0eaf8;
  min-width: 320px;
}

.loading-title {
  font-size: 1.1rem;
  font-weight: 600;
  margin-bottom: 6px;
  color: #fff;
}

.loading-filename {
  font-size: 0.82rem;
  color: #6b8ab0;
  margin-bottom: 20px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 280px;
}

.progress-track {
  height: 10px;
  background: #243047;
  border-radius: 99px;
  overflow: hidden;
  margin-bottom: 10px;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #3a7bd5, #4a90d9, #5aabff);
  border-radius: 99px;
  transition: width 0.2s ease;
}

.progress-pct {
  font-size: 0.85rem;
  color: #5aabff;
  font-weight: 600;
}

/* ── Toolbar ── */
.toolbar {
  position: absolute;
  top: 16px;
  right: 16px;
  display: flex;
  gap: 8px;
  z-index: 10;
}

/* ── Buttons ── */
.load-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 10px 22px;
  border: none;
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, transform 0.1s;
}

.load-btn:active { transform: scale(0.97); }

.load-btn.primary {
  background: #3a7bd5;
  color: #fff;
  font-size: 1rem;
  padding: 12px 28px;
  border-radius: 10px;
}

.load-btn.primary:hover { background: #4a8de8; }

.load-btn.secondary {
  background: #243047;
  color: #a8c4e8;
  border: 1px solid #2d3f5c;
  padding: 8px 16px;
  font-size: 0.85rem;
}

.load-btn.secondary:hover { background: #2d3f5c; }

.load-btn.accent {
  background: #1e4d8c;
  color: #d0e8ff;
  border: 1px solid #3a7bd5;
  padding: 8px 16px;
  font-size: 0.85rem;
}

.load-btn.accent:hover { background: #245ca8; }
.load-btn:disabled { opacity: 0.45; cursor: not-allowed; }

.btn-icon { font-size: 1.1em; }

/* ── Element properties ── */
.element-properties {
  position: absolute;
  top: 60px;
  right: 16px;
  background: rgba(20, 28, 44, 0.95);
  border: 1px solid #2d3f5c;
  color: #c8daf2;
  padding: 16px 20px;
  border-radius: 10px;
  max-width: 300px;
  max-height: 60%;
  overflow-y: auto;
  text-align: left;
  z-index: 10;
}

.element-properties h2 {
  margin: 0 0 10px;
  font-size: 0.95rem;
  color: #fff;
}

.element-properties pre {
  margin: 0;
  font-size: 0.75rem;
  white-space: pre-wrap;
  word-break: break-word;
}
</style>
