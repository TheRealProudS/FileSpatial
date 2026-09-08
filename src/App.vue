<template>
  <div class="fs-app" :class="`theme-${currentTheme}`">
    <header class="fs-header">
      <div class="fs-header-left">
        <div class="fs-brand" @click="navigateToBoard('root')" title="Zum Hauptboard">
          <div class="fs-logo-box">
            <img src="/logo.svg" alt="FileSpatial" class="fs-logo" />
          </div>
          <div class="fs-brand-text">
            <span class="fs-name">File<span class="fs-accent">Spatial</span></span>
          </div>
        </div>
        <div class="fs-header-vsep"></div>
        <nav class="fs-breadcrumbs" aria-label="Whiteboard Navigation">
          <button 
            class="fs-crumb-btn" 
            :class="{ active: currentBoardId === 'root' }" 
            @click="navigateToBoard('root')"
          >
            <span>Hauptboard</span>
          </button>
          <template v-for="(crumb, cIdx) in breadcrumbPath" :key="crumb.id">
            <span class="fs-crumb-sep">/</span>
            <button 
              class="fs-crumb-btn" 
              :class="{ active: cIdx === breadcrumbPath.length - 1 }"
              @click="navigateToBoard(crumb.id)"
            >
              <span>{{ crumb.title }}</span>
            </button>
          </template>
        </nav>
      </div>
      <div class="fs-header-center">
        <div class="fs-toolbar-group">
          <button class="fs-tool-btn" @click="addNoteNode" title="Neue Notizkarte erstellen">
            <span class="fs-tool-icon">📝</span>
            <span>Notiz</span>
          </button>
          <button class="fs-tool-btn highlight" @click="addProjectNode" title="Neues Unter-Whiteboard / Projekt">
            <span class="fs-tool-icon">📁</span>
            <span>Projekt</span>
          </button>
          <button class="fs-tool-btn" @click="triggerFileUpload" title="Dateien (Bilder, PDFs, Text) ablegen">
            <span class="fs-tool-icon">📎</span>
            <span>Datei</span>
          </button>
          <input type="file" ref="fileInput" multiple style="display: none;" @change="handleFileInput" />
          <input type="file" ref="importFileInput" accept=".fspatial,.json" style="display: none;" @change="handleImportFile" />
        </div>
        <div class="fs-search-box">
          <span class="fs-search-icon">🔍</span>
          <input 
            type="text" 
            v-model="searchQuery" 
            placeholder="Suchen... (Enter)" 
            class="fs-search-input"
            @keydown.enter="focusNextSearchResult"
          />
          <span v-if="searchQuery" class="fs-search-count">{{ searchResults.length }}</span>
        </div>
      </div>
      <div class="fs-header-right">
        <div class="fs-util-group">
          <button 
            class="fs-icon-btn" 
            :class="{ active: snapToGrid }" 
            @click="snapToGrid = !snapToGrid" 
            :title="snapToGrid ? 'Raster aktiv (20px) – Klick zum Deaktivieren' : 'Raster aus – Klick zum Aktivieren'"
          >
            <span>▦</span>
          </button>
          <button class="fs-icon-btn" @click="resetView" title="Ansicht zentrieren (100% Zoom)">
            <span>⌖</span>
          </button>
        </div>
        <div class="fs-header-vsep"></div>
        <div class="fs-theme-pill" title="Erscheinungsbild wählen">
          <span class="fs-theme-icon">🎨</span>
          <select v-model="currentTheme" class="fs-theme-select" @change="saveTheme">
            <optgroup label="Dark">
              <option value="obsidian">Obsidian</option>
              <option value="midnight">Midnight OLED</option>
              <option value="cyberpunk">Cyber Neon</option>
            </optgroup>
            <optgroup label="Light">
              <option value="clean-light">Titanium Light</option>
              <option value="paper-light">Warm Paper</option>
            </optgroup>
          </select>
        </div>
        <div class="fs-header-vsep"></div>
        <div class="fs-workspace-group">
          <button class="fs-ws-btn import" @click="triggerImportFile" title="Workspace importieren (.fspatial)">
            <span>Import</span>
          </button>
          <button class="fs-ws-btn export" @click="exportBoard" title="Workspace herunterladen">
            <span>Export</span>
          </button>
          <button class="fs-ws-btn clear" @click="clearWorkspace" title="Whiteboard leeren">
            <span>Neu</span>
          </button>
        </div>
      </div>
    </header>
    <main
      class="fs-canvas-container"
      ref="canvasContainer"
      @mousedown="startPan"
      @wheel.prevent="handleZoom"
      @dragover.prevent="handleDragOver"
      @drop.prevent="handleDrop"
    >
      <div
        class="fs-canvas-layer"
        :style="{
          transform: `translate(${panX}px, ${panY}px) scale(${zoom})`,
          transformOrigin: '0 0'
        }"
      >
        <div class="fs-grid-background"></div>
        <svg class="fs-svg-connections">
          <line
            v-for="(con, idx) in connections"
            :key="idx"
            :x1="getNodeCenter(con.from).x"
            :y1="getNodeCenter(con.from).y"
            :x2="getNodeCenter(con.to).x"
            :y2="getNodeCenter(con.to).y"
            stroke="#6366f1"
            stroke-width="2"
            stroke-dasharray="6 4"
            opacity="0.6"
          />
        </svg>
        <div
          v-for="node in nodes"
          :key="node.id"
          class="fs-node glass-node"
          :class="{ 
            selected: selectedNodeId === node.id,
            'search-highlight': isNodeHighlighted(node)
          }"
          :style="{ 
            left: node.x + 'px', 
            top: node.y + 'px', 
            width: node.width + 'px',
            minHeight: node.height ? node.height + 'px' : undefined
          }"
          @mousedown.stop="startDragNode($event, node)"
        >
          <div class="fs-node-header">
            <div class="fs-node-tag-badge" :style="{ borderColor: node.color }">
              <span class="fs-node-tag-dot" :style="{ background: node.color }"></span>
              <span :style="{ color: node.color }">{{ node.type.toUpperCase() }}</span>
            </div>
            <span class="fs-node-title" :title="node.title">{{ node.title }}</span>
            <div class="fs-node-actions">
              <div class="fs-node-color-picker" title="Farbe ändern" @mousedown.stop>
                <button class="fs-node-action-btn fs-color-btn" :style="{ background: node.color }"></button>
                <div class="fs-color-palette">
                  <span 
                    v-for="c in colorOptions" 
                    :key="c" 
                    class="fs-color-swatch" 
                    :style="{ background: c }"
                    @click.stop="setNodeColor(node, c)"
                  ></span>
                </div>
              </div>
              <button 
                class="fs-node-action-btn" 
                @click.stop="duplicateNode(node)" 
                title="Karte duplizieren"
              >
                ⧉
              </button>
              <button class="fs-node-del" @click.stop="removeNode(node.id)" title="Löschen">✕</button>
            </div>
          </div>
          <div v-if="node.type === 'project'" class="fs-node-body project-body">
            <div class="fs-project-card" @dblclick.stop="openProjectBoard(node)">
              <div class="fs-project-icon-box">
                <svg viewBox="0 0 24 24" fill="none" stroke="#6366f1" stroke-width="2">
                  <path d="M22 19a2 2 0 0 1-2 2H4a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h5l2 3h9a2 2 0 0 1 2 2z"></path>
                </svg>
              </div>
              <div class="fs-project-details">
                <input 
                  v-model="node.title" 
                  class="fs-project-title-input" 
                  placeholder="Projekt Name..." 
                  @mousedown.stop 
                />
                <span class="fs-project-badge">{{ getSubBoardCount(node.targetBoardId) }} Elemente enthalten</span>
              </div>
            </div>
            <div class="fs-project-actions">
              <button class="fs-project-btn primary" @click.stop="openProjectBoard(node)">
                <span>Whiteboard öffnen →</span>
              </button>
            </div>
          </div>
          <div v-else-if="node.type === 'note'" class="fs-node-body">
            <textarea
              v-model="node.content"
              placeholder="Schreibe Notizen, Spezifikationen oder Ideen..."
              class="fs-textarea"
              :style="{ height: node.height ? (node.height - 75) + 'px' : '120px' }"
              @mousedown.stop
            ></textarea>
          </div>
          <div v-else-if="node.type === 'image'" class="fs-node-body img-body">
            <div class="fs-asset-frame" :style="{ height: node.height ? (node.height - 95) + 'px' : '150px' }">
              <img :src="node.url" alt="Asset" class="fs-node-img" draggable="false" />
            </div>
            <div class="fs-file-meta">
              <span>{{ node.fileName }}</span>
              <span class="fs-mini-tag">SVG Asset</span>
            </div>
          </div>
          <div v-else-if="node.type === 'pdf'" class="fs-node-body doc-body">
            <div class="fs-pdf-preview" @click.stop="openPdfInWindow(node)" title="PDF in separatem Fenster öffnen">
              <div class="fs-pdf-icon-frame">
                <svg viewBox="0 0 24 24" fill="none" stroke="#06b6d4" stroke-width="2"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/><polyline points="10 9 9 9 8 9"/></svg>
              </div>
              <div class="fs-pdf-info">
                <span class="fs-pdf-name">{{ node.fileName }}</span>
                <span class="fs-pdf-sub">{{ node.numPages ? node.numPages + ' Seiten · Bereit' : 'PDF Dokument' }}</span>
              </div>
            </div>
            <div class="fs-pdf-actions">
              <button class="fs-pdf-btn primary" @click.stop="openPdfInWindow(node)" title="PDF in separatem Fenster/Tab öffnen">
                <span class="fs-btn-icon">↗</span>
                <span>Im extra Fenster öffnen</span>
              </button>
              <button class="fs-pdf-btn secondary" @click.stop="openPdfReader(node)" title="Im Canvas-Overlay betrachten">
                <span>Vorschau</span>
              </button>
            </div>
          </div>
          <div 
            class="fs-node-resizer" 
            title="Größe anpassen"
            @mousedown.stop="startResizeNode($event, node)"
          >
            <svg viewBox="0 0 10 10" width="10" height="10" fill="currentColor">
              <circle cx="8" cy="8" r="1.5" />
              <circle cx="4" cy="8" r="1.5" />
              <circle cx="8" cy="4" r="1.5" />
            </svg>
          </div>
        </div>
      </div>
    </main>
    <div v-if="activePdfNode" class="fs-pdf-modal-backdrop" @click="closePdfReader">
      <div class="fs-pdf-modal" @click.stop>
        <div class="fs-pdf-modal-header">
          <div class="fs-pdf-modal-title">
            <span class="fs-pdf-modal-icon">📄</span>
            <span class="fs-pdf-modal-filename">{{ activePdfNode.fileName }}</span>
            <span v-if="activePdfTotalPages > 0" class="fs-pdf-modal-pages">Seite {{ activePdfCurrentPage }} von {{ activePdfTotalPages }}</span>
          </div>
          <div class="fs-pdf-modal-controls">
            <button class="fs-modal-btn" :disabled="activePdfCurrentPage <= 1" @click="prevPdfPage">‹ Vorherige</button>
            <button class="fs-modal-btn" :disabled="activePdfCurrentPage >= activePdfTotalPages" @click="nextPdfPage">Nächste ›</button>
            <div class="fs-sep"></div>
            <button class="fs-modal-btn" @click="zoomPdf(-0.2)">-</button>
            <span class="fs-modal-zoom-text">{{ Math.round(pdfScale * 100) }}%</span>
            <button class="fs-modal-btn" @click="zoomPdf(0.2)">+</button>
            <div class="fs-sep"></div>
            <a v-if="activePdfNode.url" :href="activePdfNode.url" target="_blank" class="fs-modal-btn secondary">Im Tab öffnen ↗</a>
            <button class="fs-modal-close" @click="closePdfReader" title="Schließen">✕</button>
          </div>
        </div>
        <div class="fs-pdf-modal-body" ref="pdfContainerRef">
          <div v-if="pdfLoading" class="fs-pdf-loader">
            <span class="fs-pdf-spinner"></span>
            <span>Dokument wird geladen...</span>
          </div>
          <div v-else-if="pdfError" class="fs-pdf-error-frame">
            <span class="fs-pdf-error-icon">⚠️</span>
            <span class="fs-pdf-error-text">{{ pdfError }}</span>
            <div class="fs-pdf-error-actions">
              <a v-if="activePdfNode.url" :href="activePdfNode.url" target="_blank" class="fs-modal-btn secondary">Im Browser-Tab öffnen ↗</a>
              <button class="fs-modal-btn" @click="retryLoadPdf">Erneut versuchen</button>
            </div>
            <iframe v-if="activePdfNode.url" :src="activePdfNode.url" class="fs-pdf-fallback-iframe"></iframe>
          </div>
          <canvas v-show="!pdfLoading && !pdfError" ref="pdfCanvasRef" class="fs-pdf-canvas"></canvas>
        </div>
      </div>
    </div>
    <footer class="fs-cockpit-bar">
      <div class="fs-cockpit-item">
        <span class="fs-c-label">Engine:</span>
        <span class="fs-c-val">Hardware Matrix 60FPS</span>
      </div>
      <div class="fs-c-divider"></div>
      <div class="fs-cockpit-item">
        <span class="fs-c-label">Zoom:</span>
        <span class="fs-c-val">{{ Math.round(zoom * 100) }}%</span>
      </div>
      <div class="fs-c-divider"></div>
      <div class="fs-cockpit-item">
        <span class="fs-c-label">Nodes:</span>
        <span class="fs-c-val">{{ nodes.length }} aktiv</span>
      </div>
      <div class="fs-cockpit-item fs-c-right">
        <span class="fs-hint-accent">Drag & Drop:</span>
        <span>Dateien aus dem Windows-Explorer direkt auf das Board ziehen</span>
      </div>
    </footer>
  </div>
</template>
<script setup lang=ts>
import { ref, reactive, computed, onMounted, onUnmounted, nextTick } from 'vue';
import * as pdfjsLib from 'pdfjs-dist';

try {
  pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.js';
} catch (e) {
  pdfjsLib.GlobalWorkerOptions.workerSrc = `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.js`;
}

const THEME_STORAGE_KEY = 'filespatial_theme_v1';
const currentTheme = ref('obsidian');
function loadTheme() {
  const saved = localStorage.getItem(THEME_STORAGE_KEY);
  if (saved) {
    currentTheme.value = saved;
  }
}
function saveTheme() {
  localStorage.setItem(THEME_STORAGE_KEY, currentTheme.value);
}
interface CanvasNode {
  id: string;
  title: string;
  type: 'note' | 'image' | 'pdf' | 'project';
  x: number;
  y: number;
  width: number;
  height?: number;
  content?: string;
  url?: string;
  fileName?: string;
  numPages?: number;
  file?: File;
  targetBoardId?: string;
  color: string;
}
const colorOptions = ['#6366f1', '#06b6d4', '#10b981', '#f59e0b', '#ec4899', '#8b5cf6', '#3b82f6'];
const snapToGrid = ref(false);
const searchQuery = ref('');
const importFileInput = ref<HTMLInputElement | null>(null);
interface ProjectBoard {
  id: string;
  title: string;
  parentId: string | null;
  nodes: CanvasNode[];
  connections: { from: string; to: string }[];
  panX: number;
  panY: number;
  zoom: number;
}
const STORAGE_KEY = 'filespatial_boards_v2';
const boards = reactive<Record<string, ProjectBoard>>({
  'root': {
    id: 'root',
    title: 'Hauptboard',
    parentId: null,
    panX: 150,
    panY: 100,
    zoom: 1,
    nodes: [
      {
        id: 'n-1',
        title: 'Projekt-Architektur',
        type: 'note',
        x: 200,
        y: 180,
        width: 280,
        content: 'FileSpatial Core Engine:\n• 100% Local-First\n• Zero-Cloud\n• Free Software by Zenkner-Technology',
        color: '#6366f1'
      },
      {
        id: 'n-2',
        title: 'Brand Assets & Logo',
        type: 'image',
        x: 580,
        y: 150,
        width: 260,
        url: '/logo.svg',
        fileName: 'logo.svg',
        color: '#38bdf8'
      },
      {
        id: 'n-3',
        title: 'Spezifikation_2026.pdf',
        type: 'pdf',
        x: 400,
        y: 420,
        width: 280,
        url: '/Spezifikation_2026.pdf',
        fileName: 'Spezifikation_2026.pdf',
        numPages: 1,
        color: '#10b981'
      }
    ],
    connections: [
      { from: 'n-1', to: 'n-2' },
      { from: 'n-1', to: 'n-3' }
    ]
  }
});

function loadBoards() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) {
      const parsed = JSON.parse(raw);
      if (parsed && typeof parsed === 'object' && parsed.root) {
        Object.keys(boards).forEach(k => delete boards[k]);
        Object.assign(boards, parsed);
      }
    }
  } catch (err) {
    console.error('Konnte Boards nicht aus localStorage laden:', err);
  }
}
function saveBoards() {
  try {
    
    const copy: Record<string, any> = {};
    for (const [bId, bVal] of Object.entries(boards)) {
      copy[bId] = {
        ...bVal,
        nodes: bVal.nodes.map(n => {
          const { file, ...rest } = n;
          return rest;
        })
      };
    }
    localStorage.setItem(STORAGE_KEY, JSON.stringify(copy));
  } catch (err) {
    console.error('Konnte Boards nicht in localStorage speichern:', err);
  }
}
const currentBoardId = ref('root');
const currentBoard = computed<ProjectBoard>(() => {
  return boards[currentBoardId.value] || boards['root'];
});
const currentBoardTitle = computed(() => {
  return currentBoard.value?.title || 'Hauptboard';
});
const nodes = computed(() => {
  return currentBoard.value.nodes;
});
const connections = computed(() => {
  return currentBoard.value.connections;
});
const panX = computed({
  get: () => currentBoard.value.panX,
  set: (val: number) => { currentBoard.value.panX = val; }
});
const panY = computed({
  get: () => currentBoard.value.panY,
  set: (val: number) => { currentBoard.value.panY = val; }
});
const zoom = computed({
  get: () => currentBoard.value.zoom,
  set: (val: number) => { currentBoard.value.zoom = val; }
});
const breadcrumbPath = computed(() => {
  const path: { id: string; title: string }[] = [];
  let currId: string | null = currentBoardId.value;
  while (currId && currId !== 'root') {
    const b = boards[currId];
    if (!b) break;
    path.unshift({ id: b.id, title: b.title });
    currId = b.parentId;
  }
  return path;
});
function getSubBoardCount(targetId?: string) {
  if (!targetId || !boards[targetId]) return 0;
  return boards[targetId].nodes.length;
}
function navigateToBoard(boardId: string) {
  if (boards[boardId]) {
    currentBoardId.value = boardId;
    selectedNodeId.value = null;
    isDraggingNode.value = false;
    activeDragNode = null;
    saveBoards();
  }
}
function openProjectBoard(node: CanvasNode) {
  if (!node.targetBoardId) {
    const newBoardId = 'b-' + Date.now();
    boards[newBoardId] = {
      id: newBoardId,
      title: node.title || 'Neues Projekt',
      parentId: currentBoardId.value,
      panX: 150,
      panY: 100,
      zoom: 1,
      nodes: [
        {
          id: 'n-' + Date.now(),
          title: 'Projektstart',
          type: 'note',
          x: 200,
          y: 180,
          width: 280,
          content: `Projekt: ${node.title}\nHier Notizen und Dateien organisieren...`,
          color: '#6366f1'
        }
      ],
      connections: []
    };
    node.targetBoardId = newBoardId;
  }
  navigateToBoard(node.targetBoardId);
}
function addProjectNode() {
  const newBoardId = 'b-' + Date.now();
  const title = `Projekt ${Object.keys(boards).length}`;
  boards[newBoardId] = {
    id: newBoardId,
    title,
    parentId: currentBoardId.value,
    panX: 150,
    panY: 100,
    zoom: 1,
    nodes: [],
    connections: []
  };
  const id = 'n-' + Date.now();
  currentBoard.value.nodes.push({
    id,
    title,
    type: 'project',
    targetBoardId: newBoardId,
    x: (-panX.value + 300) / zoom.value,
    y: (-panY.value + 200) / zoom.value,
    width: 280,
    color: '#8b5cf6'
  });
  saveBoards();
}

const isPanning = ref(false);
const startMouseX = ref(0);
const startMouseY = ref(0);
const fileInput = ref<HTMLInputElement | null>(null);
const selectedNodeId = ref<string | null>(null);
const isDraggingNode = ref(false);
let dragOffset = { x: 0, y: 0 };
let activeDragNode: CanvasNode | null = null;

const isResizingNode = ref(false);
let activeResizeNode: CanvasNode | null = null;
let resizeStartWidth = 0;
let resizeStartHeight = 0;
let resizeStartX = 0;
let resizeStartY = 0;
function getNodeCenter(nodeId: string) {
  const node = currentBoard.value.nodes.find(n => n.id === nodeId);
  if (!node) return { x: 0, y: 0 };
  return {
    x: node.x + node.width / 2,
    y: node.y + (node.height ? node.height / 2 : 75)
  };
}
function startPan(e: MouseEvent) {
  if (e.button !== 0) return;
  if (e.target !== e.currentTarget && !(e.target as HTMLElement).classList.contains('fs-grid-background') && !(e.target as HTMLElement).classList.contains('fs-svg-connections')) return;
  isPanning.value = true;
  startMouseX.value = e.clientX - panX.value;
  startMouseY.value = e.clientY - panY.value;
}
function onWindowMouseMove(e: MouseEvent) {
  if (isPanning.value) {
    panX.value = e.clientX - startMouseX.value;
    panY.value = e.clientY - startMouseY.value;
  } else if (isDraggingNode.value && activeDragNode) {
    const rawX = (e.clientX - panX.value) / zoom.value;
    const rawY = (e.clientY - panY.value) / zoom.value;
    let nextX = Math.round(rawX - dragOffset.x);
    let nextY = Math.round(rawY - dragOffset.y);
    if (snapToGrid.value) {
      nextX = Math.round(nextX / 20) * 20;
      nextY = Math.round(nextY / 20) * 20;
    }
    activeDragNode.x = nextX;
    activeDragNode.y = nextY;
  } else if (isResizingNode.value && activeResizeNode) {
    const deltaX = (e.clientX - resizeStartX) / zoom.value;
    const deltaY = (e.clientY - resizeStartY) / zoom.value;
    let nextW = Math.max(220, Math.round(resizeStartWidth + deltaX));
    let nextH = Math.max(120, Math.round(resizeStartHeight + deltaY));
    if (snapToGrid.value) {
      nextW = Math.round(nextW / 20) * 20;
      nextH = Math.round(nextH / 20) * 20;
    }
    activeResizeNode.width = nextW;
    activeResizeNode.height = nextH;
  }
}
function onWindowMouseUp() {
  if (isDraggingNode.value || isResizingNode.value) {
    saveBoards();
  }
  isPanning.value = false;
  isDraggingNode.value = false;
  isResizingNode.value = false;
  activeDragNode = null;
  activeResizeNode = null;
}
function handleZoom(e: WheelEvent) {
  const zoomFactor = 1.1;
  let newZoom = e.deltaY < 0 ? zoom.value * zoomFactor : zoom.value / zoomFactor;
  newZoom = Math.min(Math.max(0.2, newZoom), 3);
  zoom.value = newZoom;
}
function resetView() {
  panX.value = 150;
  panY.value = 100;
  zoom.value = 1;
}
function startDragNode(e: MouseEvent, node: CanvasNode) {
  if (e.button !== 0) return;
  isDraggingNode.value = true;
  activeDragNode = node;
  selectedNodeId.value = node.id;
  const rawX = (e.clientX - panX.value) / zoom.value;
  const rawY = (e.clientY - panY.value) / zoom.value;
  dragOffset = {
    x: rawX - node.x,
    y: rawY - node.y
  };
}
function startResizeNode(e: MouseEvent, node: CanvasNode) {
  if (e.button !== 0) return;
  isResizingNode.value = true;
  activeResizeNode = node;
  selectedNodeId.value = node.id;
  resizeStartX = e.clientX;
  resizeStartY = e.clientY;
  resizeStartWidth = node.width || 280;
  resizeStartHeight = node.height || 180;
}

function duplicateNode(node: CanvasNode) {
  const newId = 'n-' + Date.now();
  const duplicate: CanvasNode = {
    ...JSON.parse(JSON.stringify(node)),
    id: newId,
    title: `${node.title} (Kopie)`,
    x: node.x + 30,
    y: node.y + 30
  };
  currentBoard.value.nodes.push(duplicate);
  selectedNodeId.value = newId;
  saveBoards();
}
function setNodeColor(node: CanvasNode, color: string) {
  node.color = color;
  saveBoards();
}

const searchResults = computed(() => {
  const query = searchQuery.value.trim().toLowerCase();
  if (!query) return [];
  return currentBoard.value.nodes.filter(n => {
    const inTitle = n.title?.toLowerCase().includes(query);
    const inContent = n.content?.toLowerCase().includes(query);
    const inFileName = n.fileName?.toLowerCase().includes(query);
    return inTitle || inContent || inFileName;
  });
});
let currentSearchFocusIdx = 0;
function focusNextSearchResult() {
  if (searchResults.value.length === 0) return;
  currentSearchFocusIdx = (currentSearchFocusIdx + 1) % searchResults.value.length;
  const targetNode = searchResults.value[currentSearchFocusIdx];
  if (targetNode) {
    focusOnNode(targetNode);
  }
}
function isNodeHighlighted(node: CanvasNode): boolean {
  if (!searchQuery.value.trim()) return false;
  return searchResults.value.some(n => n.id === node.id);
}
function focusOnNode(node: CanvasNode) {
  selectedNodeId.value = node.id;
  const containerWidth = window.innerWidth;
  const containerHeight = window.innerHeight;
  panX.value = Math.round(containerWidth / 2 - (node.x + node.width / 2) * zoom.value);
  panY.value = Math.round(containerHeight / 2 - (node.y + 100) * zoom.value);
}

function triggerImportFile() {
  importFileInput.value?.click();
}
function handleImportFile(e: Event) {
  const input = e.target as HTMLInputElement;
  if (!input.files || input.files.length === 0) return;
  const file = input.files[0];
  const reader = new FileReader();
  reader.onload = (event) => {
    try {
      const content = event.target?.result as string;
      const parsed = JSON.parse(content);
      if (parsed && typeof parsed === 'object' && parsed.boards && parsed.boards.root) {
        Object.keys(boards).forEach(k => delete boards[k]);
        Object.assign(boards, parsed.boards);
        currentBoardId.value = parsed.currentBoardId && boards[parsed.currentBoardId] ? parsed.currentBoardId : 'root';
        saveBoards();
        resetView();
        alert('Workspace erfolgreich importiert!');
      } else {
        alert('Ungültiges Workspace-Format (.fspatial Datei erwartet).');
      }
    } catch (err) {
      alert('Fehler beim Einlesen der Workspace-Datei.');
      console.error(err);
    } finally {
      input.value = '';
    }
  };
  reader.readAsText(file);
}
function clearWorkspace() {
  if (confirm('Möchtest du das aktuelle Whiteboard wirklich zurücksetzen? Alle nicht exportierten Notizen gehen verloren.')) {
    boards['root'].nodes = [];
    boards['root'].connections = [];
    navigateToBoard('root');
    resetView();
    saveBoards();
  }
}
function addNoteNode() {
  const id = 'n-' + Date.now();
  currentBoard.value.nodes.push({
    id,
    title: 'Neue Notiz',
    type: 'note',
    x: (-panX.value + 300) / zoom.value,
    y: (-panY.value + 200) / zoom.value,
    width: 260,
    content: '',
    color: '#818cf8'
  });
  saveBoards();
}
function removeNode(id: string) {
  const b = currentBoard.value;
  const index = b.nodes.findIndex(n => n.id === id);
  if (index !== -1) b.nodes.splice(index, 1);
  const conIndices = b.connections.map((c, i) => (c.from === id || c.to === id ? i : -1)).filter(i => i !== -1);
  for (let i = conIndices.length - 1; i >= 0; i--) {
    b.connections.splice(conIndices[i], 1);
  }
  saveBoards();
}
function triggerFileUpload() {
  fileInput.value?.click();
}
function handleFileInput(e: Event) {
  const target = e.target as HTMLInputElement;
  if (target.files) {
    processFiles(Array.from(target.files));
  }
}
function handleDragOver(e: DragEvent) {
  e.dataTransfer!.dropEffect = 'copy';
}
function handleDrop(e: DragEvent) {
  if (e.dataTransfer?.files) {
    processFiles(Array.from(e.dataTransfer.files));
  }
}
function processFiles(files: File[]) {
  files.forEach(async (file, index) => {
    const isImg = file.type.startsWith('image/');
    const isPdf = file.name.toLowerCase().endsWith('.pdf') || file.type === 'application/pdf';
    const id = 'n-' + Date.now() + '-' + index;
    const posX = (-panX.value + 250 + index * 40) / zoom.value;
    const posY = (-panY.value + 150 + index * 40) / zoom.value;
    if (isImg) {
      const url = URL.createObjectURL(file);
      currentBoard.value.nodes.push({
        id,
        title: file.name,
        type: 'image',
        x: posX,
        y: posY,
        width: 280,
        url,
        fileName: file.name,
        color: '#38bdf8'
      });
      saveBoards();
    } else if (isPdf) {
      const url = URL.createObjectURL(file);
      const pdfNode: CanvasNode = {
        id,
        title: file.name,
        type: 'pdf',
        x: posX,
        y: posY,
        width: 280,
        url,
        file,
        fileName: file.name,
        numPages: undefined,
        content: `Dateiname: ${file.name}\nGroesse: ${(file.size / 1024).toFixed(1)} KB`,
        color: '#10b981'
      };
      currentBoard.value.nodes.push(pdfNode);
      saveBoards();
      try {
        const arrayBuffer = await file.arrayBuffer();
        const loadingTask = pdfjsLib.getDocument({ data: arrayBuffer });
        const pdfDoc = await loadingTask.promise;
        pdfNode.numPages = pdfDoc.numPages;
        saveBoards();
      } catch (err) {
        console.error('Konnte PDF Metadaten nicht lesen:', err);
      }
    } else {
      currentBoard.value.nodes.push({
        id,
        title: file.name,
        type: 'note',
        x: posX,
        y: posY,
        width: 280,
        fileName: file.name,
        content: `Dateiname: ${file.name}\nGroesse: ${(file.size / 1024).toFixed(1)} KB`,
        color: '#f59e0b'
      });
      saveBoards();
    }
  });
}

const activePdfNode = ref<CanvasNode | null>(null);
const activePdfDoc = ref<any>(null);
const activePdfCurrentPage = ref(1);
const activePdfTotalPages = ref(1);
const pdfScale = ref(1.3);
const pdfLoading = ref(false);
const pdfError = ref<string | null>(null);
const pdfCanvasRef = ref<HTMLCanvasElement | null>(null);
const pdfContainerRef = ref<HTMLDivElement | null>(null);
let currentRenderTask: any = null;
async function retryLoadPdf() {
  if (activePdfNode.value) {
    await openPdfReader(activePdfNode.value);
  }
}
function openPdfInWindow(node: CanvasNode) {
  let url = node.url;
  if (!url && node.file) {
    url = URL.createObjectURL(node.file);
    node.url = url;
  }
  if (url) {
    const win = window.open(url, '_blank');
    if (!win) {
      window.location.href = url;
    }
  } else {
    
    openPdfReader(node);
  }
}
async function openPdfReader(node: CanvasNode) {
  activePdfNode.value = node;
  activePdfCurrentPage.value = 1;
  pdfScale.value = 1.3;
  pdfLoading.value = true;
  pdfError.value = null;
  try {
    let source: any;
    if (node.file) {
      const buffer = await node.file.arrayBuffer();
      source = { data: buffer };
    } else if (node.url) {
      
      try {
        const res = await fetch(node.url);
        if (res.ok) {
          const buffer = await res.arrayBuffer();
          source = { data: buffer };
        } else {
          source = { url: node.url };
        }
      } catch {
        source = { url: node.url };
      }
    }
    if (!source) {
      pdfError.value = 'PDF-Quelldatei konnte nicht gefunden werden.';
      pdfLoading.value = false;
      return;
    }
    const loadingTask = pdfjsLib.getDocument(source);
    activePdfDoc.value = await loadingTask.promise;
    activePdfTotalPages.value = activePdfDoc.value.numPages;
    node.numPages = activePdfDoc.value.numPages;
    await nextTick();
    await renderPdfPage();
  } catch (err: any) {
    console.error('Fehler beim Laden des PDFs:', err);
    pdfError.value = err?.message || 'Fehler beim Laden des Dokuments.';
  } finally {
    pdfLoading.value = false;
  }
}
async function renderPdfPage() {
  if (!activePdfDoc.value || !pdfCanvasRef.value) return;
  if (currentRenderTask) {
    try {
      currentRenderTask.cancel();
    } catch {}
    currentRenderTask = null;
  }
  try {
    const page = await activePdfDoc.value.getPage(activePdfCurrentPage.value);
    const viewport = page.getViewport({ scale: pdfScale.value });
    const canvas = pdfCanvasRef.value;
    const context = canvas.getContext('2d');
    if (!context) return;
    
    const outputScale = window.devicePixelRatio || 1;
    canvas.width = Math.floor(viewport.width * outputScale);
    canvas.height = Math.floor(viewport.height * outputScale);
    canvas.style.width = Math.floor(viewport.width) + 'px';
    canvas.style.height = Math.floor(viewport.height) + 'px';
    const transform = outputScale !== 1 ? [outputScale, 0, 0, outputScale, 0, 0] : undefined;
    const renderContext = {
      canvasContext: context,
      transform: transform,
      viewport: viewport
    };
    currentRenderTask = page.render(renderContext);
    await currentRenderTask.promise;
    currentRenderTask = null;
  } catch (err: any) {
    if (err?.name !== 'RenderingCancelledException') {
      console.error('Fehler beim Rendern der PDF-Seite:', err);
    }
  }
}
async function prevPdfPage() {
  if (activePdfCurrentPage.value > 1) {
    activePdfCurrentPage.value--;
    await renderPdfPage();
  }
}
async function nextPdfPage() {
  if (activePdfCurrentPage.value < activePdfTotalPages.value) {
    activePdfCurrentPage.value++;
    await renderPdfPage();
  }
}
async function zoomPdf(delta: number) {
  const newScale = Math.min(Math.max(pdfScale.value + delta, 0.6), 3.0);
  pdfScale.value = Math.round(newScale * 10) / 10;
  await renderPdfPage();
}
function closePdfReader() {
  if (currentRenderTask) {
    try { currentRenderTask.cancel(); } catch {}
    currentRenderTask = null;
  }
  activePdfNode.value = null;
  activePdfDoc.value = null;
  pdfError.value = null;
}
function exportBoard() {
  const data = JSON.stringify({ version: '2.0', currentBoardId: currentBoardId.value, boards }, null, 2);
  const blob = new Blob([data], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `workspace_${currentBoard.value.title.toLowerCase().replace(/\s+/g, '_')}.fspatial`;
  a.click();
}
onMounted(() => {
  loadTheme();
  loadBoards();
  window.addEventListener('mousemove', onWindowMouseMove);
  window.addEventListener('mouseup', onWindowMouseUp);
});
onUnmounted(() => {
  window.removeEventListener('mousemove', onWindowMouseMove);
  window.removeEventListener('mouseup', onWindowMouseUp);
});
</script>
<style scoped>
/* ZenTech Enterprise Theme Tokens */
.fs-app {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #04070d;
  color: #f8fafc;
  overflow: hidden;
  position: relative;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
}
/* Modern Header Architecture */
.fs-header {
  height: 58px;
  background: rgba(9, 13, 22, 0.88);
  backdrop-filter: blur(20px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  z-index: 50;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.45);
  gap: 16px;
}
.fs-header-left,
.fs-header-center,
.fs-header-right {
  display: flex;
  align-items: center;
}
.fs-header-left {
  gap: 12px;
  min-width: 0;
}
.fs-header-center {
  gap: 12px;
  flex-shrink: 0;
}
.fs-header-right {
  gap: 10px;
  flex-shrink: 0;
}
.fs-header-vsep {
  width: 1px;
  height: 20px;
  background: rgba(255, 255, 255, 0.1);
}
/* Brand */
.fs-brand {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
  user-select: none;
}
.fs-logo-box {
  width: 34px;
  height: 34px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 9px;
  background: rgba(6, 182, 212, 0.1);
  border: 1px solid rgba(6, 182, 212, 0.3);
  box-shadow: 0 0 12px rgba(6, 182, 212, 0.25);
  transition: transform 0.15s ease;
}
.fs-brand:hover .fs-logo-box {
  transform: scale(1.05);
}
.fs-logo {
  width: 22px;
  height: 22px;
}
.fs-brand-text {
  display: flex;
  flex-direction: column;
}
.fs-name {
  font-size: 1.05rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  color: #fff;
  line-height: 1;
}
.fs-accent {
  color: #06b6d4;
  text-shadow: 0 0 10px rgba(6, 182, 212, 0.4);
}
/* Breadcrumbs */
.fs-breadcrumbs {
  display: flex;
  align-items: center;
  gap: 4px;
  background: rgba(255, 255, 255, 0.03);
  padding: 3px 8px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.06);
}
.fs-crumb-btn {
  background: none;
  border: none;
  color: #94a3b8;
  font-size: 0.76rem;
  font-weight: 600;
  cursor: pointer;
  padding: 3px 8px;
  border-radius: 5px;
  transition: all 0.15s ease;
  white-space: nowrap;
}
.fs-crumb-btn:hover {
  color: #fff;
  background: rgba(255, 255, 255, 0.08);
}
.fs-crumb-btn.active {
  color: #06b6d4;
  font-weight: 700;
}
.fs-crumb-sep {
  color: #475569;
  font-size: 0.7rem;
}
/* Center Toolbar Group */
.fs-toolbar-group {
  display: flex;
  align-items: center;
  background: rgba(15, 23, 42, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 9px;
  padding: 3px;
  gap: 2px;
}
.fs-tool-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  background: transparent;
  border: 1px solid transparent;
  color: #cbd5e1;
  padding: 5px 11px;
  border-radius: 6px;
  font-size: 0.78rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
}
.fs-tool-btn:hover {
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
}
.fs-tool-btn.highlight {
  color: #a5b4fc;
}
.fs-tool-btn.highlight:hover {
  background: rgba(99, 102, 241, 0.18);
  color: #c7d2fe;
}
.fs-tool-icon {
  font-size: 0.85rem;
}
/* Search Box */
.fs-search-box {
  display: flex;
  align-items: center;
  gap: 6px;
  background: rgba(15, 23, 42, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 9px;
  padding: 4px 10px;
  height: 32px;
  box-sizing: border-box;
}
.fs-search-icon {
  font-size: 0.75rem;
  opacity: 0.6;
}
.fs-search-input {
  background: transparent;
  border: none;
  color: #f8fafc;
  font-size: 0.78rem;
  width: 120px;
  outline: none;
}
.fs-search-input:focus {
  width: 150px;
  transition: width 0.2s ease;
}
.fs-search-input::placeholder {
  color: #64748b;
}
.fs-search-count {
  font-size: 0.65rem;
  background: rgba(99, 102, 241, 0.25);
  color: #a5b4fc;
  border-radius: 10px;
  padding: 1px 5px;
  font-weight: 700;
}
/* Utilities (Grid & Reset View) */
.fs-util-group {
  display: flex;
  align-items: center;
  background: rgba(15, 23, 42, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 9px;
  padding: 3px;
  gap: 2px;
}
.fs-icon-btn {
  width: 28px;
  height: 26px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  color: #94a3b8;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.15s ease;
}
.fs-icon-btn:hover {
  color: #fff;
  background: rgba(255, 255, 255, 0.08);
}
.fs-icon-btn.active {
  color: #22d3ee;
  background: rgba(6, 182, 212, 0.2);
}
/* Theme Pill */
.fs-theme-pill {
  display: flex;
  align-items: center;
  gap: 6px;
  background: rgba(15, 23, 42, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 9px;
  padding: 3px 8px;
  height: 32px;
  box-sizing: border-box;
}
.fs-theme-icon {
  font-size: 0.75rem;
}
.fs-theme-select {
  background: transparent;
  border: none;
  color: #cbd5e1;
  font-size: 0.75rem;
  font-weight: 600;
  outline: none;
  cursor: pointer;
}
.fs-theme-select option,
.fs-theme-select optgroup {
  background: #0f172a;
  color: #f8fafc;
}
/* Workspace Group (Import, Export, Clear) */
.fs-workspace-group {
  display: flex;
  align-items: center;
  background: rgba(15, 23, 42, 0.7);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 9px;
  padding: 3px;
  gap: 3px;
}
.fs-ws-btn {
  background: transparent;
  border: 1px solid transparent;
  font-size: 0.74rem;
  font-weight: 600;
  padding: 4px 9px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.15s ease;
  color: #94a3b8;
}
.fs-ws-btn:hover {
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
}
.fs-ws-btn.export {
  background: rgba(6, 182, 212, 0.12);
  border-color: rgba(6, 182, 212, 0.3);
  color: #22d3ee;
  font-weight: 700;
}
.fs-ws-btn.export:hover {
  background: #06b6d4;
  color: #04070d;
  box-shadow: 0 0 12px rgba(6, 182, 212, 0.4);
}
.fs-ws-btn.clear {
  color: #f87171;
}
.fs-ws-btn.clear:hover {
  background: rgba(239, 68, 68, 0.15);
  color: #fca5a5;
  border-color: rgba(239, 68, 68, 0.25);
}
/* Viewport */
.fs-canvas-container {
  flex: 1;
  width: 100%;
  height: calc(100vh - 58px - 36px);
  position: relative;
  overflow: hidden;
  cursor: grab;
  user-select: none;
  -webkit-user-select: none;
}
.fs-canvas-container:active {
  cursor: grabbing;
}
.fs-canvas-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
.fs-grid-background {
  position: absolute;
  top: -5000px;
  left: -5000px;
  width: 10000px;
  height: 10000px;
  background-image: radial-gradient(rgba(255, 255, 255, 0.12) 1px, transparent 1px);
  background-size: 32px 32px;
}
.fs-svg-connections {
  position: absolute;
  top: -5000px;
  left: -5000px;
  width: 10000px;
  height: 10000px;
  pointer-events: none;
}
/* Nodes */
.fs-node {
  position: absolute;
  background: rgba(13, 18, 31, 0.92);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 14px;
  box-shadow: 0 14px 36px rgba(0, 0, 0, 0.6), 0 0 0 1px rgba(255, 255, 255, 0.03);
  cursor: grab;
  user-select: none;
  -webkit-user-select: none;
  -webkit-user-drag: none;
  overflow: visible;
  transition: box-shadow 0.2s ease, border-color 0.2s ease;
}
.fs-node:active {
  cursor: grabbing;
}
.fs-node:hover {
  border-color: rgba(6, 182, 212, 0.3);
}
.fs-node.selected {
  border-color: #06b6d4;
  box-shadow: 0 0 24px rgba(6, 182, 212, 0.35), 0 16px 40px rgba(0, 0, 0, 0.7);
}
.fs-node.search-highlight {
  border-color: #f59e0b !important;
  box-shadow: 0 0 24px rgba(245, 158, 11, 0.6) !important;
  animation: pulseHighlight 2s infinite alternate;
}
@keyframes pulseHighlight {
  0% { transform: scale(1); }
  100% { transform: scale(1.02); }
}
.fs-node-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background: rgba(255, 255, 255, 0.02);
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);
}
.fs-node-tag-badge {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 0.62rem;
  font-weight: 800;
  letter-spacing: 0.04em;
  padding: 2px 7px;
  border-radius: 4px;
  border: 1px solid;
  background: rgba(255, 255, 255, 0.03);
}
.fs-node-tag-dot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
}
.fs-node-title {
  flex: 1;
  font-size: 0.82rem;
  font-weight: 600;
  color: #f1f5f9;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.fs-node-actions {
  display: flex;
  align-items: center;
  gap: 4px;
}
.fs-node-action-btn {
  background: none;
  border: none;
  color: #94a3b8;
  cursor: pointer;
  font-size: 0.8rem;
  padding: 3px 5px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s ease;
}
.fs-node-action-btn:hover {
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
}
.fs-node-color-picker {
  position: relative;
  display: inline-block;
}
.fs-color-btn {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.2);
  padding: 0;
  cursor: pointer;
}
.fs-color-palette {
  display: none;
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  background: #0f172a;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 8px;
  padding: 6px;
  gap: 6px;
  z-index: 100;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.5);
}
.fs-node-color-picker:hover .fs-color-palette {
  display: flex;
}
.fs-color-swatch {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  cursor: pointer;
  transition: transform 0.1s ease;
}
.fs-color-swatch:hover {
  transform: scale(1.3);
}
.fs-node-resizer {
  position: absolute;
  right: 2px;
  bottom: 2px;
  width: 14px;
  height: 14px;
  cursor: se-resize;
  color: #64748b;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
  opacity: 0.6;
  transition: opacity 0.15s ease, color 0.15s ease;
}
.fs-node-resizer:hover {
  opacity: 1;
  color: #06b6d4;
}
.fs-node-del {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  font-size: 0.8rem;
  padding: 2px 6px;
  border-radius: 4px;
  transition: all 0.15s ease;
}
.fs-node-del:hover {
  color: #ef4444;
  background: rgba(239, 68, 68, 0.1);
}
.fs-node-body {
  padding: 14px;
}
.fs-textarea {
  width: 100%;
  height: 120px;
  background: #080c14;
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 8px;
  color: #e2e8f0;
  padding: 10px;
  font-size: 0.8rem;
  line-height: 1.5;
  resize: none;
  font-family: inherit;
}
.fs-textarea:focus {
  outline: none;
  border-color: #06b6d4;
  box-shadow: 0 0 0 2px rgba(6, 182, 212, 0.2);
}
.fs-asset-frame {
  width: 100%;
  height: 150px;
  background: #080c14;
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 12px;
}
.fs-node-img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}
.fs-file-meta {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 8px;
  font-size: 0.72rem;
  color: #94a3b8;
}
.fs-mini-tag {
  background: rgba(6, 182, 212, 0.1);
  color: #06b6d4;
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 0.65rem;
  font-weight: 600;
}
/* Project / Sub-Board Card Styling */
.project-body {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.fs-project-card {
  display: flex;
  align-items: center;
  gap: 12px;
  background: rgba(99, 102, 241, 0.08);
  border: 1px solid rgba(99, 102, 241, 0.25);
  border-radius: 10px;
  padding: 12px;
  cursor: pointer;
  transition: all 0.15s ease;
}
.fs-project-card:hover {
  background: rgba(99, 102, 241, 0.15);
  border-color: rgba(99, 102, 241, 0.4);
}
.fs-project-icon-box {
  width: 44px;
  height: 44px;
  border-radius: 10px;
  background: rgba(99, 102, 241, 0.15);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.fs-project-icon-box svg {
  width: 24px;
  height: 24px;
}
.fs-project-details {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex: 1;
  min-width: 0;
}
.fs-project-title-input {
  background: transparent;
  border: 1px solid transparent;
  color: #f8fafc;
  font-size: 0.85rem;
  font-weight: 700;
  border-radius: 4px;
  padding: 2px 4px;
  width: 100%;
  font-family: inherit;
  transition: all 0.15s ease;
}
.fs-project-title-input:hover {
  border-color: rgba(255, 255, 255, 0.15);
}
.fs-project-title-input:focus {
  outline: none;
  border-color: #6366f1;
  background: rgba(0, 0, 0, 0.4);
}
.fs-project-badge {
  font-size: 0.68rem;
  color: #a5b4fc;
}
.fs-project-actions {
  display: flex;
}
.fs-project-btn {
  width: 100%;
  padding: 7px 12px;
  font-size: 0.75rem;
  font-weight: 700;
  border-radius: 6px;
  border: 1px solid rgba(99, 102, 241, 0.4);
  background: rgba(99, 102, 241, 0.18);
  color: #c7d2fe;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.15s ease;
}
.fs-project-btn:hover {
  background: #6366f1;
  color: #fff;
  box-shadow: 0 0 16px rgba(99, 102, 241, 0.4);
}
.fs-pdf-preview {
  display: flex;
  align-items: center;
  gap: 12px;
  background: #080c14;
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 10px;
  padding: 12px;
  cursor: pointer;
  transition: all 0.15s ease;
}
.fs-pdf-preview:hover {
  border-color: rgba(6, 182, 212, 0.3);
  background: rgba(6, 182, 212, 0.04);
}
.fs-pdf-icon-frame {
  width: 40px;
  height: 40px;
  border-radius: 8px;
  background: rgba(6, 182, 212, 0.08);
  display: flex;
  align-items: center;
  justify-content: center;
}
.fs-pdf-icon-frame svg {
  width: 22px;
  height: 22px;
}
.fs-pdf-info {
  display: flex;
  flex-direction: column;
}
.fs-pdf-name {
  font-size: 0.82rem;
  font-weight: 600;
  color: #f1f5f9;
}
.fs-pdf-sub {
  font-size: 0.68rem;
  color: #64748b;
}
.fs-pdf-actions {
  display: flex;
  gap: 8px;
  margin-top: 10px;
}
.fs-pdf-btn {
  flex: 1;
  padding: 7px 10px;
  font-size: 0.74rem;
  font-weight: 600;
  border-radius: 6px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  background: rgba(255, 255, 255, 0.04);
  color: #cbd5e1;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  text-decoration: none;
  transition: all 0.15s ease;
}
.fs-pdf-btn:hover {
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
  border-color: rgba(255, 255, 255, 0.2);
}
.fs-pdf-btn.primary {
  background: rgba(6, 182, 212, 0.15);
  border-color: rgba(6, 182, 212, 0.35);
  color: #22d3ee;
  font-weight: 700;
}
.fs-pdf-btn.primary:hover {
  background: #06b6d4;
  color: #04070d;
}
.fs-pdf-btn.secondary {
  flex: 0 0 auto;
  padding: 7px 10px;
}
.fs-btn-icon {
  font-size: 0.85rem;
  line-height: 1;
}
/* Interactive PDF Modal */
.fs-pdf-modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.75);
  backdrop-filter: blur(8px);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}
.fs-pdf-modal {
  width: 90vw;
  max-width: 960px;
  height: 85vh;
  background: #0b0f19;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 14px;
  display: flex;
  flex-direction: column;
  box-shadow: 0 24px 64px rgba(0, 0, 0, 0.8), 0 0 0 1px rgba(6, 182, 212, 0.15);
  overflow: hidden;
}
.fs-pdf-modal-header {
  height: 52px;
  padding: 0 16px;
  background: #0e1422;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}
.fs-pdf-modal-title {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
}
.fs-pdf-modal-filename {
  font-size: 0.85rem;
  font-weight: 600;
  color: #f1f5f9;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.fs-pdf-modal-pages {
  font-size: 0.72rem;
  color: #06b6d4;
  background: rgba(6, 182, 212, 0.1);
  padding: 2px 8px;
  border-radius: 4px;
  white-space: nowrap;
}
.fs-pdf-modal-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}
.fs-modal-btn {
  padding: 5px 10px;
  font-size: 0.75rem;
  font-weight: 500;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: #e2e8f0;
  border-radius: 6px;
  cursor: pointer;
  text-decoration: none;
  transition: all 0.15s ease;
}
.fs-modal-btn:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}
.fs-modal-btn:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.1);
  border-color: rgba(255, 255, 255, 0.2);
}
.fs-modal-btn.secondary {
  color: #06b6d4;
  border-color: rgba(6, 182, 212, 0.3);
}
.fs-modal-zoom-text {
  font-size: 0.72rem;
  color: #94a3b8;
  min-width: 44px;
  text-align: center;
}
.fs-modal-close {
  background: none;
  border: none;
  color: #94a3b8;
  font-size: 1rem;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 6px;
  transition: all 0.15s ease;
}
.fs-modal-close:hover {
  color: #ef4444;
  background: rgba(239, 68, 68, 0.1);
}
.fs-pdf-modal-body {
  flex: 1;
  overflow: auto;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding: 24px;
  background: #070a10;
  position: relative;
}
.fs-pdf-canvas {
  background: #ffffff;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  border-radius: 4px;
  max-width: 100%;
}
.fs-pdf-loader {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  color: #94a3b8;
  font-size: 0.85rem;
}
.fs-pdf-spinner {
  width: 24px;
  height: 24px;
  border: 2px solid rgba(6, 182, 212, 0.2);
  border-top-color: #06b6d4;
  border-radius: 50%;
  animation: fs-spin 0.7s linear infinite;
}
@keyframes fs-spin {
  to { transform: rotate(360deg); }
}
.fs-theme-select-box {
  display: flex;
  align-items: center;
  gap: 6px;
  background: rgba(255, 255, 255, 0.04);
  padding: 4px 8px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.08);
}
.fs-theme-icon {
  font-size: 0.85rem;
}
.fs-theme-select {
  background: transparent;
  border: none;
  color: #e2e8f0;
  font-size: 0.76rem;
  font-weight: 600;
  cursor: pointer;
  outline: none;
  font-family: inherit;
}
.fs-theme-select optgroup {
  background: #0f172a;
  color: #94a3b8;
}
.fs-theme-select option {
  background: #0b0f19;
  color: #f8fafc;
}
/* PDF Error & Fallback Frame */
.fs-pdf-error-frame {
  width: 100%;
  max-width: 720px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  padding: 24px;
  background: rgba(15, 23, 42, 0.6);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  text-align: center;
}
.fs-pdf-error-icon {
  font-size: 2rem;
}
.fs-pdf-error-text {
  font-size: 0.85rem;
  color: #e2e8f0;
}
.fs-pdf-error-actions {
  display: flex;
  gap: 10px;
}
.fs-pdf-fallback-iframe {
  width: 100%;
  height: 500px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  background: #fff;
  margin-top: 10px;
}
/* Cockpit Footer */
.fs-cockpit-bar {
  height: 36px;
  background: rgba(9, 13, 22, 0.96);
  border-top: 1px solid rgba(255, 255, 255, 0.06);
  display: flex;
  align-items: center;
  padding: 0 20px;
  font-size: 0.72rem;
  color: #64748b;
  z-index: 50;
}
.fs-cockpit-item {
  display: flex;
  align-items: center;
  gap: 6px;
}
.fs-c-label {
  color: #475569;
  text-transform: uppercase;
  font-weight: 700;
  font-size: 0.64rem;
}
.fs-c-val {
  color: #cbd5e1;
  font-weight: 600;
}
.fs-c-divider {
  width: 1px;
  height: 14px;
  background: rgba(255, 255, 255, 0.08);
  margin: 0 14px;
}
.fs-c-right {
  margin-left: auto;
}
.fs-hint-accent {
  color: #06b6d4;
  font-weight: 600;
}
/* =========================================================
   THEMES: DARK MODES & LIGHT MODES
   ========================================================= */
/* Theme 1: Midnight OLED (Dark) */
.fs-app.theme-midnight {
  background: #000000;
  color: #f1f5f9;
}
.fs-app.theme-midnight .fs-header {
  background: rgba(0, 0, 0, 0.95);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}
.fs-app.theme-midnight .fs-node {
  background: rgba(10, 10, 10, 0.95);
  border: 1px solid rgba(255, 255, 255, 0.12);
  box-shadow: 0 16px 40px rgba(0, 0, 0, 0.9);
}
.fs-app.theme-midnight .fs-grid-background {
  background-image: radial-gradient(rgba(255, 255, 255, 0.15) 1px, transparent 1px);
}
.fs-app.theme-midnight .fs-cockpit-bar {
  background: #000000;
  border-top: 1px solid rgba(255, 255, 255, 0.08);
}
/* Theme 2: Cyber Neon (Dark) */
.fs-app.theme-cyberpunk {
  background: #090514;
  color: #f5f3ff;
}
.fs-app.theme-cyberpunk .fs-header {
  background: rgba(15, 8, 35, 0.92);
  border-bottom: 1px solid rgba(168, 85, 247, 0.25);
  box-shadow: 0 4px 20px rgba(168, 85, 247, 0.15);
}
.fs-app.theme-cyberpunk .fs-node {
  background: rgba(20, 12, 45, 0.94);
  border: 1px solid rgba(168, 85, 247, 0.25);
  box-shadow: 0 14px 36px rgba(0, 0, 0, 0.7), 0 0 15px rgba(168, 85, 247, 0.1);
}
.fs-app.theme-cyberpunk .fs-node.selected {
  border-color: #d946ef;
  box-shadow: 0 0 25px rgba(217, 70, 239, 0.4);
}
.fs-app.theme-cyberpunk .fs-grid-background {
  background-image: radial-gradient(rgba(168, 85, 247, 0.2) 1px, transparent 1px);
}
.fs-app.theme-cyberpunk .fs-btn.primary {
  background: #a855f7;
  border-color: #c084fc;
  box-shadow: 0 0 16px rgba(168, 85, 247, 0.4);
}
.fs-app.theme-cyberpunk .fs-cockpit-bar {
  background: rgba(15, 8, 35, 0.96);
  border-top: 1px solid rgba(168, 85, 247, 0.2);
}
/* Theme 3: Clean Titanium (Light Mode) */
.fs-app.theme-clean-light {
  background: #f1f5f9;
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-header {
  background: rgba(255, 255, 255, 0.92);
  border-bottom: 1px solid #e2e8f0;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}
.fs-app.theme-clean-light .fs-name {
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-sub {
  color: #64748b;
}
.fs-app.theme-clean-light .fs-header-vsep {
  background: #e2e8f0;
}
.fs-app.theme-clean-light .fs-toolbar-group,
.fs-app.theme-clean-light .fs-search-box,
.fs-app.theme-clean-light .fs-util-group,
.fs-app.theme-clean-light .fs-theme-pill,
.fs-app.theme-clean-light .fs-workspace-group {
  background: #f8fafc;
  border-color: #e2e8f0;
}
.fs-app.theme-clean-light .fs-tool-btn,
.fs-app.theme-clean-light .fs-search-input,
.fs-app.theme-clean-light .fs-theme-select,
.fs-app.theme-clean-light .fs-ws-btn,
.fs-app.theme-clean-light .fs-icon-btn {
  color: #334155;
}
.fs-app.theme-clean-light .fs-theme-select option,
.fs-app.theme-clean-light .fs-theme-select optgroup {
  background: #ffffff;
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-tool-btn:hover,
.fs-app.theme-clean-light .fs-ws-btn:hover,
.fs-app.theme-clean-light .fs-icon-btn:hover {
  background: #e2e8f0;
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-tool-btn.highlight {
  color: #4f46e5;
}
.fs-app.theme-clean-light .fs-tool-btn.highlight:hover {
  background: rgba(99, 102, 241, 0.12);
}
.fs-app.theme-clean-light .fs-ws-btn.export {
  background: rgba(2, 132, 199, 0.1);
  border-color: rgba(2, 132, 199, 0.25);
  color: #0284c7;
}
.fs-app.theme-clean-light .fs-ws-btn.export:hover {
  background: #0284c7;
  color: #ffffff;
}
.fs-app.theme-clean-light .fs-breadcrumbs {
  background: #ffffff;
  border-color: #e2e8f0;
}
.fs-app.theme-clean-light .fs-crumb-btn {
  color: #64748b;
}
.fs-app.theme-clean-light .fs-crumb-btn:hover {
  color: #0f172a;
  background: #e2e8f0;
}
.fs-app.theme-clean-light .fs-grid-background {
  background-image: radial-gradient(rgba(15, 23, 42, 0.15) 1px, transparent 1px);
}
.fs-app.theme-clean-light .fs-node {
  background: rgba(255, 255, 255, 0.94);
  border: 1px solid #cbd5e1;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.08);
}
.fs-app.theme-clean-light .fs-node-title {
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-node-header {
  background: #f8fafc;
  border-bottom: 1px solid #e2e8f0;
}
.fs-app.theme-clean-light .fs-textarea {
  background: #ffffff;
  border-color: #cbd5e1;
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-asset-frame {
  background: #f8fafc;
  border-color: #e2e8f0;
}
.fs-app.theme-clean-light .fs-pdf-preview {
  background: #f8fafc;
  border-color: #e2e8f0;
}
.fs-app.theme-clean-light .fs-pdf-name {
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-pdf-btn {
  background: #f1f5f9;
  border-color: #cbd5e1;
  color: #334155;
}
.fs-app.theme-clean-light .fs-project-card {
  background: #f5f3ff;
  border-color: #ddd6fe;
}
.fs-app.theme-clean-light .fs-project-title-input {
  color: #0f172a;
}
.fs-app.theme-clean-light .fs-project-title-input:focus {
  background: #ffffff;
}
.fs-app.theme-clean-light .fs-cockpit-bar {
  background: #ffffff;
  border-top: 1px solid #e2e8f0;
  color: #64748b;
}
/* Theme 4: Warm Paper (Light Mode) */
.fs-app.theme-paper-light {
  background: #fbf8f2;
  color: #292524;
}
.fs-app.theme-paper-light .fs-header {
  background: rgba(254, 252, 248, 0.95);
  border-bottom: 1px solid #e7e5e4;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.03);
}
.fs-app.theme-paper-light .fs-name {
  color: #1c1917;
}
.fs-app.theme-paper-light .fs-sub {
  color: #78716c;
}
.fs-app.theme-paper-light .fs-header-vsep {
  background: #e7e5e4;
}
.fs-app.theme-paper-light .fs-toolbar-group,
.fs-app.theme-paper-light .fs-search-box,
.fs-app.theme-paper-light .fs-util-group,
.fs-app.theme-paper-light .fs-theme-pill,
.fs-app.theme-paper-light .fs-workspace-group {
  background: #f5f5f4;
  border-color: #e7e5e4;
}
.fs-app.theme-paper-light .fs-tool-btn,
.fs-app.theme-paper-light .fs-search-input,
.fs-app.theme-paper-light .fs-theme-select,
.fs-app.theme-paper-light .fs-ws-btn,
.fs-app.theme-paper-light .fs-icon-btn {
  color: #44403c;
}
.fs-app.theme-paper-light .fs-theme-select option,
.fs-app.theme-paper-light .fs-theme-select optgroup {
  background: #fafaf9;
  color: #1c1917;
}
.fs-app.theme-paper-light .fs-tool-btn:hover,
.fs-app.theme-paper-light .fs-ws-btn:hover,
.fs-app.theme-paper-light .fs-icon-btn:hover {
  background: #e7e5e4;
  color: #1c1917;
}
.fs-app.theme-paper-light .fs-ws-btn.export {
  background: #d97706;
  border-color: #f59e0b;
  color: #ffffff;
}
.fs-app.theme-paper-light .fs-breadcrumbs {
  background: #ffffff;
  border-color: #e7e5e4;
}
.fs-app.theme-paper-light .fs-crumb-btn {
  color: #78716c;
}
.fs-app.theme-paper-light .fs-crumb-btn:hover {
  color: #1c1917;
  background: #f5f5f4;
}
.fs-app.theme-paper-light .fs-grid-background {
  background-image: radial-gradient(rgba(68, 64, 60, 0.15) 1px, transparent 1px);
}
.fs-app.theme-paper-light .fs-node {
  background: rgba(255, 255, 255, 0.96);
  border: 1px solid #d6d3d1;
  box-shadow: 0 8px 24px rgba(68, 64, 60, 0.08);
}
.fs-app.theme-paper-light .fs-node-title {
  color: #1c1917;
}
.fs-app.theme-paper-light .fs-node-header {
  background: #fafaf9;
  border-bottom: 1px solid #e7e5e4;
}
.fs-app.theme-paper-light .fs-textarea {
  background: #ffffff;
  border-color: #d6d3d1;
  color: #1c1917;
}
.fs-app.theme-paper-light .fs-cockpit-bar {
  background: #fafaf9;
  border-top: 1px solid #e7e5e4;
  color: #78716c;
}
</style>
