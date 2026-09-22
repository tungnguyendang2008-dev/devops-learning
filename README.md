<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Voxel WebCraft - Minecraft Clone</title>
    <style>
        body {
            margin: 0; padding: 0; overflow: hidden;
            background-color: #87CEEB;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            user-select: none;
        }
        #game-canvas { display: block; width: 100vw; height: 100vh; }
        
        /* UI Overlays */
        #ui-layer {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            pointer-events: none;
        }
        
        /* Màn hình Start / Pause */
        #start-screen {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.7); display: flex; flex-direction: column;
            align-items: center; justify-content: center; pointer-events: auto;
            color: white; z-index: 10;
        }
        
        /* Màn hình Kho đồ (Inventory) */
        #inventory-screen {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.8); display: none; flex-direction: column;
            align-items: center; justify-content: center; pointer-events: auto;
            color: white; z-index: 20;
        }
        .inv-container {
            background: #c6c6c6; padding: 20px; border: 4px solid #555; border-radius: 8px;
            box-shadow: inset -2px -2px 0px 0px rgba(0,0,0,0.5), inset 2px 2px 0px 0px rgba(255,255,255,0.8);
            text-align: center; color: #333;
        }
        .inv-grid {
            display: grid; grid-template-columns: repeat(6, 50px); gap: 10px; margin-bottom: 30px;
        }
        .inv-hotbar-row {
            display: flex; justify-content: center; gap: 10px; margin-top: 10px;
        }
        .inv-slot {
            width: 50px; height: 50px; background-color: #8b8b8b; border: 2px solid #373737;
            box-shadow: inset -2px -2px 0px 0px #fff, inset 2px 2px 0px 0px #373737;
            cursor: pointer; position: relative; background-size: cover;
            background-repeat: no-repeat; image-rendering: pixelated;
        }
        .inv-slot:hover { background-color: #a0a0a0; }
        .inv-slot.selected { outline: 3px solid #ffff00; z-index: 2; }
        
        h1 { font-size: 4rem; margin-bottom: 10px; text-shadow: 4px 4px 0 #000; color: white;}
        h2 { margin-top: 0; font-size: 1.5rem; text-shadow: none; color: #333;}
        .btn {
            padding: 15px 40px; font-size: 1.5rem; font-weight: bold; background: #4CAF50;
            color: white; border: 4px solid #2E7D32; cursor: pointer; margin: 10px;
            text-transform: uppercase; box-shadow: 0 4px 0 #1b5e20; transition: transform 0.1s;
        }
        .btn:active { transform: translateY(4px); box-shadow: none; }
        .btn.danger { background: #f44336; border-color: #c62828; box-shadow: 0 4px 0 #b71c1c; }
        
        /* Crosshair */
        #crosshair {
            position: absolute; top: 50%; left: 50%; width: 20px; height: 20px;
            transform: translate(-50%, -50%); pointer-events: none;
        }
        #crosshair::before, #crosshair::after { content: ''; position: absolute; background: rgba(255,255,255,0.8); }
        #crosshair::before { top: 9px; left: 0; width: 20px; height: 2px; }
        #crosshair::after { top: 0; left: 9px; width: 2px; height: 20px; }
        
        /* Hotbar in-game */
        #hotbar {
            position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%);
            display: flex; background: rgba(0,0,0,0.5); padding: 5px; border: 2px solid #555; border-radius: 4px;
        }
        .slot {
            width: 50px; height: 50px; border: 2px solid #888; margin: 0 5px;
            position: relative; background-size: cover; background-repeat: no-repeat; image-rendering: pixelated;
        }
        .slot.active { border-color: #fff; box-shadow: 0 0 10px #fff; transform: scale(1.1); }
        .slot-num {
            position: absolute; top: 2px; left: 4px; color: white; font-weight: bold; font-size: 12px; text-shadow: 1px 1px 0 #000;
        }
        
        /* Thanh đập block & máu */
        #break-bar-container {
            position: absolute; top: calc(50% + 20px); left: 50%; transform: translateX(-50%);
            width: 100px; height: 10px; background: rgba(0,0,0,0.5); border: 1px solid #fff; display: none;
        }
        #break-bar { width: 0%; height: 100%; background: #fff; }

        #health-bar-container {
            position: absolute; top: 20px; left: 20px; width: 250px; height: 25px;
            background: rgba(0,0,0,0.6); border: 3px solid #333; border-radius: 5px; display: none;
        }
        #health-bar { width: 100%; height: 100%; background: linear-gradient(90deg, #ff0000, #ff4444); transition: width 0.2s; }
        #damage-overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(255, 0, 0, 0.3); pointer-events: none; opacity: 0; transition: opacity 0.1s;
        }
        
        #cursor-item {
            position: absolute; width: 40px; height: 40px; pointer-events: none; display: none;
            background-size: cover; image-rendering: pixelated; z-index: 100;
        }
    </style>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>

    <div id="ui-layer">
        <div id="damage-overlay"></div>
        <div id="health-bar-container"><div id="health-bar"></div></div>
        <div id="crosshair" style="display:none;"></div>
        <div id="break-bar-container"><div id="break-bar"></div></div>
        <div id="hotbar" style="display:none;"></div>

        <div id="start-screen">
            <h1>WebCraft</h1>
            <p style="margin-bottom: 20px;">Xây dựng. Sinh tồn. Khám phá.</p>
            <button class="btn" id="btn-play">Chơi ngay</button>
            <button class="btn danger" id="btn-reset">Reset Thế Giới</button>
            
            <div class="controls-info">
                <strong>Điều khiển:</strong><br>
                - W, A, S, D: Di chuyển<br>
                - Space: Nhảy | Shift: Chạy nhanh<br>
                - Chuột: Xoay góc nhìn<br>
                - Chuột trái (Giữ): Phá block / Đánh quái<br>
                - Chuột phải: Đặt block<br>
                - 1-5 hoặc Cuộn chuột: Chọn block trong hotbar<br>
                - <b>E: Mở Kho đồ (Inventory)</b><br>
                - Esc: Tạm dừng / Menu
            </div>
        </div>
        
        <div id="inventory-screen">
            <div class="inv-container">
                <h2>Kho Đồ</h2>
                <div class="inv-grid" id="inv-all-items"></div>
                <hr style="border-color: #888;">
                <h2>Hotbar (1-5)</h2>
                <div class="inv-hotbar-row" id="inv-hotbar-slots"></div>
                <p style="font-size: 12px; margin-top: 15px; color: #555;">Click chọn item bên trên, sau đó click vào ô Hotbar bên dưới để gán.</p>
            </div>
            <button class="btn" id="btn-close-inv" style="margin-top: 20px;">Đóng (E)</button>
        </div>
        
        <div id="cursor-item"></div>
    </div>

<script>
// ==========================================
// 1. CONSTANTS & CẤU HÌNH
// ==========================================
const CHUNK_SIZE = 32;
const CHUNK_HEIGHT = 64;
const WORLD_SIZE = 2; 
const GRAVITY = 25;
const JUMP_FORCE = 9;
const WALK_SPEED = 5;
const RUN_SPEED = 8;
const REACH_DISTANCE = 5;

const ITEMS = {
    AIR: 0, GRASS: 1, DIRT: 2, STONE: 3, WOOD: 4, SAND: 5, WATER: 6, LEAVES: 7,
    SWORD: 9, PICKAXE: 10, AXE: 11, SHOVEL: 12
};

const ITEM_PROPS = {
    [ITEMS.GRASS]: { name: 'Grass', breakTime: 0.3, transparent: false, top: 0, bottom: 1, side: 2, isTool: false, color: 0x59a341 },
    [ITEMS.DIRT]:  { name: 'Dirt', breakTime: 0.3, transparent: false, all: 1, isTool: false, color: 0x7a5a3b },
    [ITEMS.STONE]: { name: 'Stone', breakTime: 1.2, transparent: false, all: 3, isTool: false, toolType: 'pickaxe', color: 0x888888 },
    [ITEMS.WOOD]:  { name: 'Wood', breakTime: 0.8, transparent: false, top: 5, bottom: 5, side: 4, isTool: false, toolType: 'axe', color: 0x664523 },
    [ITEMS.SAND]:  { name: 'Sand', breakTime: 0.2, transparent: false, all: 6, isTool: false, toolType: 'shovel', color: 0xd9cd8c },
    [ITEMS.WATER]: { name: 'Water', breakTime: 999, transparent: true, fluid: true, all: 7, isTool: false, color: 0x3f76e4 },
    [ITEMS.LEAVES]:{ name: 'Leaves', breakTime: 0.1, transparent: true, all: 8, isTool: false, color: 0x3c8c27 },
    
    [ITEMS.SWORD]:   { name: 'Sword', isTool: true, all: 9, damage: 15, miningSpeedObj: {} },
    [ITEMS.PICKAXE]: { name: 'Pickaxe', isTool: true, all: 10, damage: 7, miningSpeedObj: { 'pickaxe': 4.0 } },
    [ITEMS.AXE]:     { name: 'Axe', isTool: true, all: 11, damage: 10, miningSpeedObj: { 'axe': 4.0 } },
    [ITEMS.SHOVEL]:  { name: 'Shovel', isTool: true, all: 12, damage: 6, miningSpeedObj: { 'shovel': 4.0 } }
};

const ALL_AVAILABLE_ITEMS = [ ITEMS.GRASS, ITEMS.DIRT, ITEMS.STONE, ITEMS.WOOD, ITEMS.SAND, ITEMS.LEAVES, ITEMS.SWORD, ITEMS.PICKAXE, ITEMS.AXE, ITEMS.SHOVEL ];

// ==========================================
// 2. TẠO TEXTURE TỪ CANVAS
// ==========================================
function generateTextureAtlas() {
    const canvas = document.createElement('canvas');
    canvas.width = 256; canvas.height = 256;
    const ctx = canvas.getContext('2d');
    
    function drawTile(id, baseColor, noiseColor1, noiseColor2) {
        const u = (id % 16) * 16; const v = Math.floor(id / 16) * 16;
        ctx.fillStyle = baseColor; ctx.fillRect(u, v, 16, 16);
        for(let i=0; i<40; i++) {
            ctx.fillStyle = Math.random() > 0.5 ? noiseColor1 : noiseColor2;
            ctx.fillRect(u + Math.floor(Math.random() * 16), v + Math.floor(Math.random() * 16), 1, 1);
        }
    }

    drawTile(0, '#59a341', '#4b8c36', '#6ac24d'); 
    drawTile(1, '#7a5a3b', '#63482f', '#8f6a45'); 
    const sx = 2 * 16, sy = 0;
    ctx.fillStyle = '#7a5a3b'; ctx.fillRect(sx, sy, 16, 16);
    ctx.fillStyle = '#59a341'; ctx.fillRect(sx, sy, 16, 5); 
    for(let i=0; i<8; i++) if(Math.random()>0.3) ctx.fillRect(sx + i*2, sy + 5, 2, 2);
    
    drawTile(3, '#888888', '#777777', '#999999'); 
    drawTile(4, '#664523', '#4e3318', '#7d542a'); 
    drawTile(5, '#a6855b', '#8c704c', '#bf9969'); 
    drawTile(6, '#d9cd8c', '#bfb57c', '#f2e59c'); 
    drawTile(7, '#3f76e4', '#3563bf', '#4a89ff'); 
    drawTile(8, '#3c8c27', '#2e6b1e', '#4bb030'); 
    
    function drawPixelArray(id, pixels, colors) {
        const u = (id % 16) * 16; const v = Math.floor(id / 16) * 16;
        for(let y=0; y<16; y++) {
            for(let x=0; x<16; x++) {
                const char = pixels[y][x];
                if(colors[char]) { ctx.fillStyle = colors[char]; ctx.fillRect(u + x, v + y, 1, 1); }
            }
        }
    }

    const cTool = { 'w': '#5e3a19', 'i': '#dddddd', 'o': '#222222' };
    drawPixelArray(9, ["               i","              ii","             ii ","            ii  ","           ii   ","          ii    ","         ii     ","        ii      ","       ii       ","  o   ii        ","   o ii         ","    oi          ","   ow           ","  ow o          "," wo   o         ","o               "], cTool);
    drawPixelArray(10, ["       iiiiiii  ","     iii     ii ","    ii  w       ","        ow      ","         ow     ","          ow    ","           ow   ","            ow  ","             ow ","              ow","               o","                ","                ","                ","                ","                "], cTool);
    drawPixelArray(11, ["     iiii       ","    iiiii       ","    ii w        ","     i ow       ","       ow       ","        ow      ","         ow     ","          ow    ","           ow   ","            ow  ","             ow ","              o ","                ","                ","                ","                "], cTool);
    drawPixelArray(12, ["          ii    ","         iiii   ","          ii    ","           w    ","          ow    ","         ow     ","        ow      ","       ow       ","      ow        ","     ow         ","    ow          ","   ow           ","  ow            ","  o             ","                ","                "], cTool);
    
    const texture = new THREE.CanvasTexture(canvas);
    texture.magFilter = THREE.NearestFilter; texture.minFilter = THREE.NearestFilter;
    texture.imageURL = canvas.toDataURL(); 
    return texture;
}
const textureAtlas = generateTextureAtlas();
const blockMaterial = new THREE.MeshLambertMaterial({ map: textureAtlas });
const waterMaterial = new THREE.MeshLambertMaterial({ map: textureAtlas, transparent: true, opacity: 0.8, color: 0xffffff, side: THREE.DoubleSide });

// ==========================================
// 3. GENERATION ĐỊA HÌNH
// ==========================================
class Random {
    constructor(seed) { this.seed = seed; }
    next() { this.seed = (this.seed * 9301 + 49297) % 233280; return this.seed / 233280; }
}
function createNoise2D(seed) {
    const r = new Random(seed); const p = new Uint8Array(512);
    for (let i = 0; i < 256; i++) p[i] = Math.floor(r.next() * 256);
    for (let i = 0; i < 256; i++) p[256 + i] = p[i];
    function fade(t) { return t * t * t * (t * (t * 6 - 15) + 10); }
    function lerp(t, a, b) { return a + t * (b - a); }
    function grad(hash, x, y) {
        const h = hash & 3; const u = h < 2 ? x : y, v = h < 2 ? y : x;
        return ((h & 1) === 0 ? u : -u) + ((h & 2) === 0 ? v : -v);
    }
    return function(x, y) {
        const X = Math.floor(x) & 255, Y = Math.floor(y) & 255;
        x -= Math.floor(x); y -= Math.floor(y);
        const u = fade(x), v = fade(y); const a = p[X] + Y, b = p[X + 1] + Y;
        const res = lerp(v, lerp(u, grad(p[a], x, y), grad(p[b], x - 1, y)), lerp(u, grad(p[a + 1], x, y - 1), grad(p[b + 1], x - 1, y - 1)));
        return (res + 1) / 2;
    };
}

class World {
    constructor() {
        this.chunks = new Map(); this.modifiedBlocks = new Map();
        this.seed = localStorage.getItem('webcraft_seed') || Math.floor(Math.random() * 10000);
        localStorage.setItem('webcraft_seed', this.seed);
        this.noise = createNoise2D(this.seed);
        const saved = localStorage.getItem('webcraft_save');
        if(saved) { try { const parsed = JSON.parse(saved); for(let key in parsed) this.modifiedBlocks.set(key, parsed[key]); } catch(e) {} }
    }
    getChunkKey(cx, cz) { return `${cx},${cz}`; }
    getBlockKey(x, y, z) { return `${x},${y},${z}`; }

    generateColumn(x, z) {
        const n1 = this.noise(x * 0.015, z * 0.015) * 1.5; const n2 = this.noise(x * 0.05, z * 0.05) * 0.5;
        const heightMap = Math.floor((n1 + n2) * 16) + 12; 
        let column = new Uint8Array(CHUNK_HEIGHT); const waterLevel = 26;

        for (let y = 0; y < CHUNK_HEIGHT; y++) {
            if (y > heightMap) {
                if (y <= waterLevel) column[y] = ITEMS.WATER; else column[y] = ITEMS.AIR;
            } else if (y === heightMap) {
                if (y <= waterLevel + 1) column[y] = ITEMS.SAND; else column[y] = ITEMS.GRASS;
            } else if (y > heightMap - 4) { column[y] = ITEMS.DIRT; } else { column[y] = ITEMS.STONE; }
        }
        
        if(heightMap > waterLevel + 1 && heightMap < CHUNK_HEIGHT - 8) {
            if(Math.random() < 0.04 && x % 2 === 0 && z % 2 === 0) { 
                const treeHeight = 4 + Math.floor(Math.random() * 2);
                for(let ty=1; ty<=treeHeight; ty++) column[heightMap+ty] = ITEMS.WOOD;
            }
        }
        return column;
    }

    getChunk(cx, cz) {
        const key = this.getChunkKey(cx, cz);
        if (!this.chunks.has(key)) {
            const chunkData = new Uint8Array(CHUNK_SIZE * CHUNK_HEIGHT * CHUNK_SIZE);
            for (let x = 0; x < CHUNK_SIZE; x++) {
                for (let z = 0; z < CHUNK_SIZE; z++) {
                    const column = this.generateColumn(cx * CHUNK_SIZE + x, cz * CHUNK_SIZE + z);
                    for (let y = 0; y < CHUNK_HEIGHT; y++) chunkData[x + z * CHUNK_SIZE + y * CHUNK_SIZE * CHUNK_SIZE] = column[y];
                }
            }
            this.chunks.set(key, { data: chunkData, mesh: null, waterMesh: null });
            
            for (let x = 2; x < CHUNK_SIZE-2; x++) {
                for (let z = 2; z < CHUNK_SIZE-2; z++) {
                    for(let y=1; y<CHUNK_HEIGHT-4; y++) {
                        const idx = x + z * CHUNK_SIZE + y * CHUNK_SIZE * CHUNK_SIZE;
                        if(chunkData[idx] === ITEMS.WOOD) {
                            const isTop = chunkData[idx + CHUNK_SIZE*CHUNK_SIZE] !== ITEMS.WOOD;
                            const isSecondTop = chunkData[idx + CHUNK_SIZE*CHUNK_SIZE] === ITEMS.WOOD && chunkData[idx + 2*CHUNK_SIZE*CHUNK_SIZE] !== ITEMS.WOOD;
                            if(isTop || isSecondTop) {
                                const radius = isTop ? 1 : 2;
                                for(let lx=-radius; lx<=radius; lx++){
                                    for(let lz=-radius; lz<=radius; lz++){
                                        if (Math.abs(lx) === radius && Math.abs(lz) === radius && Math.random() > 0.5) continue;
                                        if(lx===0 && lz===0) continue;
                                        const lIdx = (x+lx) + (z+lz)*CHUNK_SIZE + (y)*CHUNK_SIZE*CHUNK_SIZE;
                                        if(lIdx >= 0 && lIdx < chunkData.length && chunkData[lIdx] === ITEMS.AIR) chunkData[lIdx] = ITEMS.LEAVES;
                                    }
                                }
                                if (isTop) {
                                    const topIdx = x + z*CHUNK_SIZE + (y+1)*CHUNK_SIZE*CHUNK_SIZE;
                                    if(topIdx < chunkData.length && chunkData[topIdx] === ITEMS.AIR) chunkData[topIdx] = ITEMS.LEAVES;
                                }
                            }
                        }
                    }
                }
            }
        }
        return this.chunks.get(key);
    }

    getBlock(x, y, z) {
        if (y < 0 || y >= CHUNK_HEIGHT) return ITEMS.AIR;
        const mKey = this.getBlockKey(x, y, z);
        if (this.modifiedBlocks.has(mKey)) return this.modifiedBlocks.get(mKey);
        const cx = Math.floor(x / CHUNK_SIZE); const cz = Math.floor(z / CHUNK_SIZE);
        const chunk = this.getChunk(cx, cz);
        return chunk.data[(x - cx * CHUNK_SIZE) + (z - cz * CHUNK_SIZE) * CHUNK_SIZE + y * CHUNK_SIZE * CHUNK_SIZE];
    }

    setBlock(x, y, z, blockId) {
        if (y < 0 || y >= CHUNK_HEIGHT) return;
        this.modifiedBlocks.set(this.getBlockKey(x, y, z), blockId);
        const cx = Math.floor(x / CHUNK_SIZE); const cz = Math.floor(z / CHUNK_SIZE);
        this.updateChunkMesh(cx, cz);
        const lx = x - cx * CHUNK_SIZE; const lz = z - cz * CHUNK_SIZE;
        if (lx === 0) this.updateChunkMesh(cx - 1, cz); if (lx === CHUNK_SIZE - 1) this.updateChunkMesh(cx + 1, cz);
        if (lz === 0) this.updateChunkMesh(cx, cz - 1); if (lz === CHUNK_SIZE - 1) this.updateChunkMesh(cx, cz + 1);
        this.save();
    }
    save() { localStorage.setItem('webcraft_save', JSON.stringify(Object.fromEntries(this.modifiedBlocks))); }

    updateChunkMesh(cx, cz) {
        const chunk = this.chunks.get(this.getChunkKey(cx, cz)); if (!chunk) return;
        if (chunk.mesh) scene.remove(chunk.mesh); if (chunk.waterMesh) scene.remove(chunk.waterMesh);

        const positions = []; const normals = []; const uvs = []; const indices = [];
        const wPositions = []; const wNormals = []; const wUvs = []; const wIndices = [];
        let ndx = 0; let wNdx = 0;

        const dirs = [
            { dir: [1, 0, 0], corners: [[1,1,1], [1,0,1], [1,0,0], [1,1,0]], n: [1,0,0] },
            { dir: [-1, 0, 0], corners: [[0,1,0], [0,0,0], [0,0,1], [0,1,1]], n: [-1,0,0] },
            { dir: [0, 1, 0], corners: [[0,1,1], [1,1,1], [1,1,0], [0,1,0]], n: [0,1,0] },
            { dir: [0, -1, 0], corners: [[0,0,0], [1,0,0], [1,0,1], [0,0,1]], n: [0,-1,0] },
            { dir: [0, 0, 1], corners: [[0,1,1], [0,0,1], [1,0,1], [1,1,1]], n: [0,0,1] },
            { dir: [0, 0, -1], corners: [[1,1,0], [1,0,0], [0,0,0], [0,1,0]], n: [0,0,-1] }
        ];

        for (let y = 0; y < CHUNK_HEIGHT; y++) {
            for (let x = 0; x < CHUNK_SIZE; x++) {
                for (let z = 0; z < CHUNK_SIZE; z++) {
                    const wx = cx * CHUNK_SIZE + x; const wz = cz * CHUNK_SIZE + z;
                    const block = this.getBlock(wx, y, wz);
                    if (block === ITEMS.AIR || ITEM_PROPS[block].isTool) continue;
                    
                    const isWater = block === ITEMS.WATER; const bProps = ITEM_PROPS[block];
                    for (let f = 0; f < 6; f++) {
                        const d = dirs[f];
                        const neighbor = this.getBlock(wx + d.dir[0], y + d.dir[1], wz + d.dir[2]);
                        
                        let drawFace = false;
                        if (neighbor === ITEMS.AIR || ITEM_PROPS[neighbor]?.isTool) drawFace = true;
                        else if (ITEM_PROPS[neighbor] && ITEM_PROPS[neighbor].transparent) drawFace = !(isWater && neighbor === ITEMS.WATER);

                        if (drawFace) {
                            let texID = bProps.all !== undefined ? bProps.all : (f === 2 ? bProps.top : (f === 3 ? bProps.bottom : bProps.side));
                            const tv = 1.0 - Math.floor(texID / 16) / 16; const tu = (texID % 16) / 16; const s = 1/16; 
                            const uvsArr = [ [tu, tv], [tu, tv-s], [tu+s, tv-s], [tu+s, tv] ];
                            
                            const pArr = isWater ? wPositions : positions; const nArr = isWater ? wNormals : normals;
                            const uvArr = isWater ? wUvs : uvs; const iArr = isWater ? wIndices : indices;
                            const curNdx = isWater ? wNdx : ndx;

                            for (let i = 0; i < 4; i++) {
                                pArr.push(wx + d.corners[i][0], y + d.corners[i][1], wz + d.corners[i][2]);
                                nArr.push(...d.n); uvArr.push(uvsArr[i][0], uvsArr[i][1]);
                            }
                            iArr.push(curNdx, curNdx+1, curNdx+2, curNdx, curNdx+2, curNdx+3);
                            if (isWater) wNdx += 4; else ndx += 4;
                        }
                    }
                }
            }
        }

        function buildMesh(p, n, u, i, mat) {
            if (p.length === 0) return null;
            const geo = new THREE.BufferGeometry();
            geo.setAttribute('position', new THREE.Float32BufferAttribute(p, 3));
            geo.setAttribute('normal', new THREE.Float32BufferAttribute(n, 3));
            geo.setAttribute('uv', new THREE.Float32BufferAttribute(u, 2));
            geo.setIndex(i);
            const mesh = new THREE.Mesh(geo, mat); scene.add(mesh); return mesh;
        }

        chunk.mesh = buildMesh(positions, normals, uvs, indices, blockMaterial);
        chunk.waterMesh = buildMesh(wPositions, wNormals, wUvs, wIndices, waterMaterial);
    }
}

// ==========================================
// 4. THIẾT LẬP SCENE & MÂY (CLOUDS)
// ==========================================
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87CEEB);
scene.fog = new THREE.Fog(0x87CEEB, 20, (WORLD_SIZE*2)*CHUNK_SIZE);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: false });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const ambientLight = new THREE.AmbientLight(0xffffff, 0.6); scene.add(ambientLight);
const dirLight = new THREE.DirectionalLight(0xffffff, 0.8); dirLight.position.set(50, 100, 50); scene.add(dirLight);

// Setup Mây trôi
const cloudGroup = new THREE.Group();
const cGeo = new THREE.BoxGeometry(1, 1, 1);
const cMat = new THREE.MeshBasicMaterial({color: 0xffffff, transparent: true, opacity: 0.6});
for(let i=0; i<40; i++) {
    const mesh = new THREE.Mesh(cGeo, cMat);
    mesh.position.set((Math.random() - 0.5) * 300, 55 + Math.random() * 15, (Math.random() - 0.5) * 300);
    mesh.scale.set(10 + Math.random() * 20, 2 + Math.random() * 3, 10 + Math.random() * 20);
    cloudGroup.add(mesh);
}
scene.add(cloudGroup);

const world = new World();
for (let cx = -WORLD_SIZE; cx <= WORLD_SIZE; cx++) {
    for (let cz = -WORLD_SIZE; cz <= WORLD_SIZE; cz++) world.updateChunkMesh(cx, cz);
}

const highlightBox = new THREE.LineSegments(new THREE.EdgesGeometry(new THREE.BoxGeometry(1.01, 1.01, 1.01)), new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 2 }));
highlightBox.visible = false; scene.add(highlightBox);

// ==========================================
// 5. HIỆU ỨNG HẠT VỠ (PARTICLES)
// ==========================================
const particles = [];
const partMaterials = {};
function getPartMat(colorHex) {
    if(!partMaterials[colorHex]) partMaterials[colorHex] = new THREE.MeshLambertMaterial({color: colorHex});
    return partMaterials[colorHex];
}
const partGeo = new THREE.BoxGeometry(0.15, 0.15, 0.15);

class Particle {
    constructor(pos, colorHex) {
        this.mesh = new THREE.Mesh(partGeo, getPartMat(colorHex));
        this.mesh.position.copy(pos);
        this.mesh.position.x += (Math.random()-0.5)*0.8;
        this.mesh.position.y += (Math.random()-0.5)*0.8;
        this.mesh.position.z += (Math.random()-0.5)*0.8;
        this.vel = new THREE.Vector3((Math.random()-0.5)*5, Math.random()*3 + 3, (Math.random()-0.5)*5);
        this.life = 0.4 + Math.random()*0.4;
        scene.add(this.mesh);
    }
    update(dt) {
        this.life -= dt;
        this.vel.y -= GRAVITY * dt;
        this.mesh.position.addScaledVector(this.vel, dt);
        this.mesh.scale.setScalar(Math.max(0, this.life * 2)); // Shrink as it dies
        if(this.life <= 0) scene.remove(this.mesh);
        return this.life <= 0;
    }
}

// ==========================================
// 6. PLAYER VÀ VẬT LÝ (Có View Bobbing)
// ==========================================
const player = {
    pos: new THREE.Vector3(0, CHUNK_HEIGHT + 10, 0),
    vel: new THREE.Vector3(0, 0, 0),
    width: 0.6, height: 1.8,
    onGround: false, yaw: 0, pitch: 0,
    walkTimer: 0, // Dùng cho View bobbing
    hotbar: [ITEMS.SWORD, ITEMS.PICKAXE, ITEMS.DIRT, ITEMS.STONE, ITEMS.WOOD],
    selectedSlotIdx: 0,
    health: 100, maxHealth: 100
};

function checkEntityCollision(pos, width, height) {
    const minX = Math.floor(pos.x - width/2); const maxX = Math.floor(pos.x + width/2);
    const minY = Math.floor(pos.y); const maxY = Math.floor(pos.y + height);
    const minZ = Math.floor(pos.z - width/2); const maxZ = Math.floor(pos.z + width/2);
    for (let x = minX; x <= maxX; x++) {
        for (let y = minY; y <= maxY; y++) {
            for (let z = minZ; z <= maxZ; z++) {
                const b = world.getBlock(x, y, z);
                if (b !== ITEMS.AIR && b !== ITEMS.WATER && b !== ITEMS.LEAVES && !ITEM_PROPS[b]?.isTool) return true;
            }
        }
    }
    return false;
}

function updatePhysics(dt) {
    player.vel.y -= GRAVITY * dt;
    if (player.vel.y < -30) player.vel.y = -30;

    let nextY = player.pos.y + player.vel.y * dt;
    if (checkEntityCollision(new THREE.Vector3(player.pos.x, nextY, player.pos.z), player.width, player.height)) {
        if(player.vel.y < -15) takePlayerDamage(Math.floor(Math.abs(player.vel.y) - 10));
        player.vel.y = 0; player.onGround = nextY < player.pos.y;
    } else { player.pos.y = nextY; player.onGround = false; }

    const speed = keys['ShiftLeft'] ? RUN_SPEED : WALK_SPEED;
    const moveDir = new THREE.Vector3();
    if (keys['KeyW']) moveDir.z -= 1; if (keys['KeyS']) moveDir.z += 1;
    if (keys['KeyA']) moveDir.x -= 1; if (keys['KeyD']) moveDir.x += 1;
    if (moveDir.lengthSq() > 0) moveDir.normalize();

    const cosY = Math.cos(player.yaw); const sinY = Math.sin(player.yaw);
    const vx = (moveDir.x * cosY + moveDir.z * sinY) * speed * dt;
    const vz = (-moveDir.x * sinY + moveDir.z * cosY) * speed * dt;

    if (!checkEntityCollision(new THREE.Vector3(player.pos.x + vx, player.pos.y, player.pos.z), player.width, player.height)) player.pos.x += vx;
    if (!checkEntityCollision(new THREE.Vector3(player.pos.x, player.pos.y, player.pos.z + vz), player.width, player.height)) player.pos.z += vz;

    // View Bobbing (Nhịp bước chân)
    if (player.onGround && moveDir.lengthSq() > 0) {
        player.walkTimer += dt * (keys['ShiftLeft'] ? 15 : 10);
    } else {
        player.walkTimer += (0 - player.walkTimer) * 10 * dt; // Mượt mà về 0
    }
    const bobbing = Math.sin(player.walkTimer) * 0.1;

    camera.position.set(player.pos.x, player.pos.y + player.height * 0.8 + bobbing, player.pos.z);
    
    const qx = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1,0,0), player.pitch);
    const qy = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,1,0), player.yaw);
    camera.quaternion.copy(qy).multiply(qx);

    if (player.pos.y < -10) respawnPlayer();
}

function takePlayerDamage(amount) {
    if(!isLocked) return;
    player.health -= amount; document.getElementById('health-bar').style.width = Math.max(0, (player.health / player.maxHealth) * 100) + '%';
    const overlay = document.getElementById('damage-overlay');
    overlay.style.opacity = 1; setTimeout(() => overlay.style.opacity = 0, 200);
    if (player.health <= 0) respawnPlayer();
}

function respawnPlayer() {
    player.pos.set(0, CHUNK_HEIGHT + 20, 0); player.vel.set(0,0,0); player.walkTimer = 0;
    player.health = player.maxHealth; document.getElementById('health-bar').style.width = '100%';
}

// ==========================================
// 7. AI QUÁI VẬT, MŨI TÊN & NỔ
// ==========================================
const mobs = []; const arrows = []; const explosions = [];

class Explosion {
    constructor(pos, radius) {
        this.pos = pos.clone(); this.maxLife = 0.3; this.life = 0;
        const mat = new THREE.MeshBasicMaterial({ color: 0xffaa00, transparent: true, opacity: 0.8 });
        this.mesh = new THREE.Mesh(new THREE.SphereGeometry(radius, 16, 16), mat); this.mesh.position.copy(pos); scene.add(this.mesh);
    }
    update(dt) {
        this.life += dt; const t = this.life / this.maxLife; this.mesh.scale.set(t, t, t); this.mesh.material.opacity = 0.8 * (1 - t);
        if (this.life >= this.maxLife) { scene.remove(this.mesh); return true; } return false;
    }
}

const arrowGeo = new THREE.BoxGeometry(0.05, 0.05, 0.6); const arrowMat = new THREE.MeshLambertMaterial({ color: 0x333333 });
class Arrow {
    constructor(pos, dir) {
        this.pos = pos.clone(); this.vel = dir.normalize().multiplyScalar(25);
        this.mesh = new THREE.Mesh(arrowGeo, arrowMat); this.mesh.position.copy(this.pos);
        this.mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0, 0, 1), dir);
        this.isDead = false; this.lifeTime = 0; scene.add(this.mesh);
    }
    update(dt) {
        if(this.isDead) return;
        this.lifeTime += dt; if(this.lifeTime > 5) { this.destroy(); return; }
        this.vel.y -= GRAVITY * 0.3 * dt; this.pos.addScaledVector(this.vel, dt); this.mesh.position.copy(this.pos);
        this.mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0, 0, 1), this.vel.clone().normalize());
        const b = world.getBlock(Math.floor(this.pos.x), Math.floor(this.pos.y), Math.floor(this.pos.z));
        if (b !== ITEMS.AIR && b !== ITEMS.WATER && b !== ITEMS.LEAVES && !ITEM_PROPS[b]?.isTool) { this.destroy(); return; }
        const distToPlayer = this.pos.distanceTo(new THREE.Vector3(player.pos.x, player.pos.y + player.height/2, player.pos.z));
        if (distToPlayer < 0.8) {
            takePlayerDamage(15); player.vel.add(this.vel.clone().normalize().multiplyScalar(5)); player.vel.y += 3; this.destroy();
        }
    }
    destroy() { this.isDead = true; scene.remove(this.mesh); }
}

function createMobMesh(type) {
    const group = new THREE.Group();
    const mat1 = new THREE.MeshLambertMaterial({ color: type === 'skeleton' ? 0xdddddd : (type === 'creeper' ? 0x1fad34 : 0x2e8f44) });
    const mat2 = new THREE.MeshLambertMaterial({ color: type === 'skeleton' ? 0xdddddd : (type === 'creeper' ? 0x1fad34 : 0x00aaff) });
    const mat3 = new THREE.MeshLambertMaterial({ color: type === 'skeleton' ? 0xdddddd : (type === 'creeper' ? 0x1fad34 : 0x5500aa) });
    
    const h = new THREE.Mesh(new THREE.BoxGeometry(0.45, 0.45, 0.45), mat1); h.position.y = type==='creeper'? 1.25 : 1.6; group.add(h);
    const b = new THREE.Mesh(new THREE.BoxGeometry(0.4, 0.7, 0.2), mat2); b.position.y = type==='creeper'? 0.6 : 1.0; group.add(b);
    
    if (type === 'creeper') {
        for(let i=0; i<4; i++) {
            const leg = new THREE.Mesh(new THREE.BoxGeometry(0.25, 0.25, 0.25), mat3);
            leg.position.set(i%2==0?0.125:-0.125, 0.125, i<2?0.15:-0.15); group.add(leg);
        }
    } else {
        const al = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.7, 0.15), mat1); al.position.set(0.3, 1.0, 0.15); al.rotation.x = Math.PI/2; group.add(al);
        const ar = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.7, 0.15), mat1); ar.position.set(-0.3, 1.0, 0.15); ar.rotation.x = Math.PI/2; group.add(ar);
        const ll = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.7, 0.15), mat3); ll.position.set(0.15, 0.35, 0); group.add(ll);
        const lr = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.7, 0.15), mat3); lr.position.set(-0.15, 0.35, 0); group.add(lr);
        if(type === 'skeleton') { const bow = new THREE.Mesh(new THREE.BoxGeometry(0.05, 0.8, 0.1), new THREE.MeshLambertMaterial({color: 0x333})); bow.position.set(-0.3, 1.0, 0.45); group.add(bow); }
    }
    group.traverse(child => { if(child.isMesh) { child.castShadow = true; child.receiveShadow = true; }});
    return group;
}

class Mob {
    constructor(x, y, z, type = 'zombie') {
        this.pos = new THREE.Vector3(x, y, z); this.vel = new THREE.Vector3(0, 0, 0); this.width = 0.6; this.height = 1.8; this.type = type;
        if (type === 'skeleton') { this.speed = 2.0; this.health = 15; }
        else if (type === 'creeper') { this.speed = 2.2; this.health = 20; this.height = 1.5; this.isCharging = false; this.chargeTime = 0; }
        else { this.speed = 2.5; this.health = 20; }
        
        this.isDead = false; this.lastAttackTime = 0;
        this.mesh = createMobMesh(type); this.mesh.position.copy(this.pos);
        this.mesh.traverse(child => { if(child.isMesh) child.userData.mob = this; }); scene.add(this.mesh);
    }

    explode() {
        if(this.isDead) return; this.isDead = true; scene.remove(this.mesh);
        explosions.push(new Explosion(this.pos, 4));
        const distToPlayer = this.pos.distanceTo(player.pos);
        if (distToPlayer < 5) {
            takePlayerDamage(Math.floor((5 - distToPlayer) * 10));
            player.vel.add(new THREE.Vector3().subVectors(player.pos, this.pos).normalize().multiplyScalar((5 - distToPlayer) * 4)); player.vel.y += (5 - distToPlayer) * 2;
        }
        const r = 3; const cx = Math.floor(this.pos.x); const cy = Math.floor(this.pos.y + 0.5); const cz = Math.floor(this.pos.z);
        const affectedChunks = new Set();
        for(let x = -r; x <= r; x++) for(let y = -r; y <= r; y++) for(let z = -r; z <= r; z++) {
            if (x*x + y*y + z*z <= r*r) {
                const bx = cx+x, by = cy+y, bz = cz+z;
                if (by < 0 || by >= CHUNK_HEIGHT) continue;
                const b = world.getBlock(bx, by, bz);
                if (b !== ITEMS.AIR && b !== ITEMS.WATER && !ITEM_PROPS[b]?.isTool) {
                    world.modifiedBlocks.set(world.getBlockKey(bx, by, bz), ITEMS.AIR);
                    const chunkX = Math.floor(bx / CHUNK_SIZE); const chunkZ = Math.floor(bz / CHUNK_SIZE); affectedChunks.add(`${chunkX},${chunkZ}`);
                    const lx = bx - chunkX*CHUNK_SIZE; const lz = bz - chunkZ*CHUNK_SIZE;
                    if (lx === 0) affectedChunks.add(`${chunkX-1},${chunkZ}`); if (lx === CHUNK_SIZE - 1) affectedChunks.add(`${chunkX+1},${chunkZ}`);
                    if (lz === 0) affectedChunks.add(`${chunkX},${chunkZ-1}`); if (lz === CHUNK_SIZE - 1) affectedChunks.add(`${chunkX},${chunkZ+1}`);
                    
                    // Spawn particles from explosion
                    if(ITEM_PROPS[b].color && Math.random() > 0.5) particles.push(new Particle(new THREE.Vector3(bx+0.5, by+0.5, bz+0.5), ITEM_PROPS[b].color));
                }
            }
        }
        world.save(); affectedChunks.forEach(key => { const [px, pz] = key.split(',').map(Number); world.updateChunkMesh(px, pz); });
    }

    takeDamage(amount, knockbackDir) {
        this.health -= amount; this.vel.x = knockbackDir.x * 8; this.vel.y = 5; this.vel.z = knockbackDir.z * 8;
        this.mesh.traverse(child => {
            if(child.isMesh) {
                const oldColor = child.material.color.getHex(); child.material.color.setHex(0xff0000);
                setTimeout(() => { if(!this.isDead) child.material.color.setHex(oldColor); }, 200);
            }
        });
        if (this.health <= 0) { this.isDead = true; scene.remove(this.mesh); }
    }

    update(dt) {
        if(this.isDead) return;
        this.vel.y -= GRAVITY * dt; let targetVelX = 0; let targetVelZ = 0;
        const distToPlayer = this.pos.distanceTo(player.pos);
        if (distToPlayer < 24) {
            const targetPos = player.pos.clone(); targetPos.y = this.pos.y; this.mesh.lookAt(targetPos);
            if (this.type === 'skeleton') {
                if (distToPlayer > 12) { const dir = new THREE.Vector3().subVectors(player.pos, this.pos).normalize(); targetVelX = dir.x * this.speed; targetVelZ = dir.z * this.speed; }
                else if (distToPlayer < 6) { const dir = new THREE.Vector3().subVectors(this.pos, player.pos).normalize(); targetVelX = dir.x * this.speed; targetVelZ = dir.z * this.speed; }
                if (performance.now() - this.lastAttackTime > 2500) { 
                    arrows.push(new Arrow(this.pos.clone().setY(this.pos.y+1.2), new THREE.Vector3().subVectors(player.pos.clone().setY(player.pos.y+0.9), this.pos.clone().setY(this.pos.y+1.2)).setY(distToPlayer * 0.05)));
                    this.lastAttackTime = performance.now();
                }
            } else if (this.type === 'creeper') {
                if (distToPlayer < 3) { this.isCharging = true; targetVelX = 0; targetVelZ = 0; } 
                else if (distToPlayer > 5) { this.isCharging = false; this.chargeTime = 0; this.mesh.scale.set(1, 1, 1); this.mesh.traverse(child => { if(child.isMesh) child.material.emissive.setHex(0x000000); }); }
                if (this.isCharging) {
                    this.chargeTime += dt; this.mesh.scale.setScalar(1 + this.chargeTime * 0.15);
                    this.mesh.traverse(child => { if(child.isMesh) child.material.emissive.setHex(Math.floor(this.chargeTime * 8) % 2 === 0 ? 0xffffff : 0x000000); });
                    if (this.chargeTime >= 1.5) { this.explode(); return; }
                } else { const dir = new THREE.Vector3().subVectors(player.pos, this.pos).normalize(); targetVelX = dir.x * this.speed; targetVelZ = dir.z * this.speed; }
            } else {
                if (distToPlayer > 1.2) { const dir = new THREE.Vector3().subVectors(player.pos, this.pos).normalize(); targetVelX = dir.x * this.speed; targetVelZ = dir.z * this.speed; }
                else if (performance.now() - this.lastAttackTime > 1000) { takePlayerDamage(10); this.lastAttackTime = performance.now(); }
            }
        }

        this.vel.x += (targetVelX - this.vel.x) * 5 * dt; this.vel.z += (targetVelZ - this.vel.z) * 5 * dt;

        let nextY = this.pos.y + this.vel.y * dt;
        if (checkEntityCollision(new THREE.Vector3(this.pos.x, nextY, this.pos.z), this.width, this.height)) {
            this.vel.y = 0; if(distToPlayer < 24 && distToPlayer > 1.2 && (Math.abs(this.vel.x) > 0.1 || Math.abs(this.vel.z) > 0.1)) this.vel.y = JUMP_FORCE * 0.8;
        } else { this.pos.y = nextY; }
        if (!checkEntityCollision(new THREE.Vector3(this.pos.x + this.vel.x * dt, this.pos.y, this.pos.z), this.width, this.height)) this.pos.x += this.vel.x * dt;
        if (!checkEntityCollision(new THREE.Vector3(this.pos.x, this.pos.y, this.pos.z + this.vel.z * dt), this.width, this.height)) this.pos.z += this.vel.z * dt;

        this.mesh.position.copy(this.pos);
        if (Math.abs(this.vel.x) > 0.1 || Math.abs(this.vel.z) > 0.1) {
            const time = performance.now() * 0.01;
            if (this.type === 'creeper') { for(let i=2;i<6;i++) this.mesh.children[i].rotation.x = Math.sin(time) * 0.5 * (i%2==0?1:-1); }
            else { this.mesh.children[4].rotation.x = Math.sin(time) * 0.5; this.mesh.children[5].rotation.x = -Math.sin(time) * 0.5; }
        }
    }
}

function spawnMobs() {
    for(let i = 0; i < 20; i++) {
        const x = (Math.random() - 0.5) * (WORLD_SIZE * CHUNK_SIZE * 1.5); const z = (Math.random() - 0.5) * (WORLD_SIZE * CHUNK_SIZE * 1.5);
        for(let y = CHUNK_HEIGHT - 1; y > 0; y--) {
            const b = world.getBlock(x, y, z);
            if(b !== ITEMS.AIR && b !== ITEMS.WATER && !ITEM_PROPS[b]?.isTool) {
                const r = Math.random(); mobs.push(new Mob(x, y + 2, z, r > 0.66 ? 'creeper' : (r > 0.33 ? 'skeleton' : 'zombie'))); break;
            }
        }
    }
}
setTimeout(spawnMobs, 1000);

// ==========================================
// 8. INPUT, RAYCAST & KHO ĐỒ
// ==========================================
const keys = {}; let isLocked = false; let isInventoryOpen = false; let rayHit = null; let breakTimer = 0; let selectedInvItem = null;

function getItemBackgroundStyle(itemId) {
    const props = ITEM_PROPS[itemId]; if (!props) return '';
    const texID = props.all !== undefined ? props.all : props.side;
    return `background-image: url(${textureAtlas.imageURL}); background-position: -${(texID % 16) * 50}px -${Math.floor(texID / 16) * 50}px; background-size: 800px 800px;`;
}

function updateHotbarUI() {
    const hotbarEl = document.getElementById('hotbar'); hotbarEl.innerHTML = '';
    player.hotbar.forEach((itemId, idx) => {
        const div = document.createElement('div'); div.className = 'slot';
        if (idx === player.selectedSlotIdx) div.classList.add('active');
        if (itemId !== ITEMS.AIR) div.style = getItemBackgroundStyle(itemId);
        div.innerHTML += `<span class="slot-num">${idx + 1}</span>`; hotbarEl.appendChild(div);
    });
}

function renderInventoryScreen() {
    const grid = document.getElementById('inv-all-items'); grid.innerHTML = '';
    ALL_AVAILABLE_ITEMS.forEach(itemId => {
        const div = document.createElement('div'); div.className = 'inv-slot'; div.style = getItemBackgroundStyle(itemId); div.title = ITEM_PROPS[itemId].name;
        div.onclick = (e) => {
            selectedInvItem = itemId; const cursorObj = document.getElementById('cursor-item');
            cursorObj.style = getItemBackgroundStyle(itemId) + '; display: block; position: absolute; pointer-events: none; z-index: 100; left: '+(e.clientX - 20)+'px; top: '+(e.clientY - 20)+'px;';
            Array.from(grid.children).forEach(c => c.classList.remove('selected')); div.classList.add('selected');
        }; grid.appendChild(div);
    });
    const hotbarRow = document.getElementById('inv-hotbar-slots'); hotbarRow.innerHTML = '';
    player.hotbar.forEach((itemId, idx) => {
        const div = document.createElement('div'); div.className = 'inv-slot';
        if (itemId !== ITEMS.AIR) div.style = getItemBackgroundStyle(itemId); div.innerHTML += `<span class="slot-num">${idx + 1}</span>`;
        div.onclick = () => {
            if (selectedInvItem !== null) {
                player.hotbar[idx] = selectedInvItem; selectedInvItem = null; document.getElementById('cursor-item').style.display = 'none';
                Array.from(grid.children).forEach(c => c.classList.remove('selected')); renderInventoryScreen(); updateHotbarUI();
            }
        }; hotbarRow.appendChild(div);
    });
}

document.addEventListener('mousemove', e => {
    if (!isLocked) {
        if (isInventoryOpen && selectedInvItem !== null) { const c = document.getElementById('cursor-item'); c.style.left = e.clientX - 20 + 'px'; c.style.top = e.clientY - 20 + 'px'; } return;
    }
    player.yaw -= (e.movementX || 0) * 0.002; player.pitch -= Math.max(-Math.PI/2, Math.min(Math.PI/2, (e.movementY || 0) * 0.002));
    player.pitch = Math.max(-Math.PI/2, Math.min(Math.PI/2, player.pitch));
});

document.addEventListener('keydown', e => {
    keys[e.code] = true;
    if (e.code === 'Space' && player.onGround && isLocked) { player.vel.y = JUMP_FORCE; player.onGround = false; }
    if (isLocked && e.key >= '1' && e.key <= '5') { player.selectedSlotIdx = parseInt(e.key) - 1; updateHotbarUI(); }
    if (e.code === 'KeyE') {
        if (isLocked) {
            document.exitPointerLock(); isInventoryOpen = true; document.getElementById('inventory-screen').style.display = 'flex';
            document.getElementById('hotbar').style.display = 'none'; document.getElementById('crosshair').style.display = 'none'; selectedInvItem = null; renderInventoryScreen();
        } else if (isInventoryOpen) {
            isInventoryOpen = false; document.getElementById('inventory-screen').style.display = 'none'; document.getElementById('cursor-item').style.display = 'none'; document.body.requestPointerLock();
        }
    }
});
document.addEventListener('keyup', e => keys[e.code] = false);
document.addEventListener('wheel', e => { if(!isLocked) return; player.selectedSlotIdx = (player.selectedSlotIdx + (e.deltaY > 0 ? 1 : 4)) % 5; updateHotbarUI(); });

const meshRaycaster = new THREE.Raycaster(); const mouse = { left: false };
document.addEventListener('mousedown', e => {
    if (!isLocked) return;
    const heldItemId = player.hotbar[player.selectedSlotIdx]; const heldProps = ITEM_PROPS[heldItemId] || {};
    
    if (e.button === 0) {
        meshRaycaster.setFromCamera(new THREE.Vector2(0, 0), camera);
        const intersects = meshRaycaster.intersectObjects(mobs.filter(m => !m.isDead).map(m => m.mesh), true);
        if (intersects.length > 0 && intersects[0].distance < REACH_DISTANCE) {
            if(intersects[0].object.userData.mob) { intersects[0].object.userData.mob.takeDamage(heldProps.damage || 5, new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion).normalize()); return; }
        }
        mouse.left = true; 
    }
    
    if (e.button === 2 && rayHit && heldItemId !== ITEMS.AIR && !heldProps.isTool) {
        const np = rayHit.placePos;
        if (!(player.pos.x - player.width/2 < np.x + 1 && player.pos.x + player.width/2 > np.x && player.pos.y < np.y + 1 && player.pos.y + player.height > np.y && player.pos.z - player.width/2 < np.z + 1 && player.pos.z + player.width/2 > np.z)) {
            world.setBlock(np.x, np.y, np.z, heldItemId);
        }
    }
});
document.addEventListener('mouseup', e => { if (e.button === 0) { mouse.left = false; breakTimer = 0; } });

function raycastVoxel(origin, dir, maxDist) {
    let x = Math.floor(origin.x); let y = Math.floor(origin.y); let z = Math.floor(origin.z);
    const stepX = Math.sign(dir.x); const stepY = Math.sign(dir.y); const stepZ = Math.sign(dir.z);
    const tDeltaX = stepX !== 0 ? Math.abs(1 / dir.x) : Infinity; const tDeltaY = stepY !== 0 ? Math.abs(1 / dir.y) : Infinity; const tDeltaZ = stepZ !== 0 ? Math.abs(1 / dir.z) : Infinity;
    let tMaxX = stepX > 0 ? (Math.floor(origin.x) + 1 - origin.x) * tDeltaX : (origin.x - Math.floor(origin.x)) * tDeltaX;
    let tMaxY = stepY > 0 ? (Math.floor(origin.y) + 1 - origin.y) * tDeltaY : (origin.y - Math.floor(origin.y)) * tDeltaY;
    let tMaxZ = stepZ > 0 ? (Math.floor(origin.z) + 1 - origin.z) * tDeltaZ : (origin.z - Math.floor(origin.z)) * tDeltaZ;
    let face = new THREE.Vector3();

    for (let i = 0; i < maxDist * 3; i++) {
        if (tMaxX < tMaxY) {
            if (tMaxX < tMaxZ) { x += stepX; tMaxX += tDeltaX; face.set(-stepX, 0, 0); } else { z += stepZ; tMaxZ += tDeltaZ; face.set(0, 0, -stepZ); }
        } else {
            if (tMaxY < tMaxZ) { y += stepY; tMaxY += tDeltaY; face.set(0, -stepY, 0); } else { z += stepZ; tMaxZ += tDeltaZ; face.set(0, 0, -stepZ); }
        }
        const b = world.getBlock(x, y, z);
        if (b !== ITEMS.AIR && b !== ITEMS.WATER && !ITEM_PROPS[b]?.isTool) {
            if(Math.sqrt((x-origin.x)**2 + (y-origin.y)**2 + (z-origin.z)**2) > maxDist) return null;
            return { pos: new THREE.Vector3(x, y, z), placePos: new THREE.Vector3(x + face.x, y + face.y, z + face.z), blockId: b };
        }
    } return null;
}

// ==========================================
// 9. UI EVENT LISTENERS
// ==========================================
document.addEventListener('contextmenu', e => e.preventDefault());
document.getElementById('btn-play').addEventListener('click', () => document.body.requestPointerLock());
document.getElementById('btn-close-inv').addEventListener('click', () => {
    isInventoryOpen = false; document.getElementById('inventory-screen').style.display = 'none'; document.getElementById('cursor-item').style.display = 'none'; document.body.requestPointerLock();
});
document.getElementById('btn-reset').addEventListener('click', () => { localStorage.removeItem('webcraft_seed'); localStorage.removeItem('webcraft_save'); location.reload(); });

document.addEventListener('pointerlockchange', () => {
    isLocked = document.pointerLockElement === document.body;
    if (isLocked) {
        document.getElementById('start-screen').style.display = 'none'; document.getElementById('inventory-screen').style.display = 'none'; isInventoryOpen = false;
        document.getElementById('crosshair').style.display = 'block'; document.getElementById('hotbar').style.display = 'flex'; document.getElementById('health-bar-container').style.display = 'block';
    } else {
        if (!isInventoryOpen) { document.getElementById('start-screen').style.display = 'flex'; document.getElementById('btn-play').innerText = "Tiếp tục chơi"; }
        document.getElementById('crosshair').style.display = 'none'; document.getElementById('hotbar').style.display = 'none'; document.getElementById('health-bar-container').style.display = 'none';
        mouse.left = false; breakTimer = 0;
    }
});
updateHotbarUI();

// ==========================================
// 10. VÒNG LẶP GAME (MAIN LOOP)
// ==========================================
const clock = new THREE.Clock(); let dayTime = 0;
const breakBarContainer = document.getElementById('break-bar-container'); const breakBar = document.getElementById('break-bar');

function animate() {
    requestAnimationFrame(animate);
    const dt = Math.min(clock.getDelta(), 0.1); 
    
    if (isLocked) {
        dayTime += dt * 0.05; const sunLight = Math.max(0.2, Math.sin(dayTime));
        scene.background.setHSL(0.55, 0.6, sunLight * 0.5 + 0.1); scene.fog.color.copy(scene.background);
        ambientLight.intensity = sunLight * 0.5 + 0.2; dirLight.intensity = sunLight; dirLight.position.set(Math.cos(dayTime)*100, Math.sin(dayTime)*100, 50);

        // Update mây trôi
        cloudGroup.children.forEach(c => { c.position.x += dt * 2; if (c.position.x > 150) c.position.x -= 300; });

        updatePhysics(dt);

        for(let i = mobs.length - 1; i >= 0; i--) { mobs[i].update(dt); if(mobs[i].isDead) mobs.splice(i, 1); }
        for(let i = arrows.length - 1; i >= 0; i--) { arrows[i].update(dt); if(arrows[i].isDead) arrows.splice(i, 1); }
        for(let i = explosions.length - 1; i >= 0; i--) { if(explosions[i].update(dt)) explosions.splice(i, 1); }
        for(let i = particles.length - 1; i >= 0; i--) { if(particles[i].update(dt)) particles.splice(i, 1); } // Update Particles

        rayHit = raycastVoxel(camera.position, new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion), REACH_DISTANCE);

        if (rayHit) {
            highlightBox.visible = true; highlightBox.position.set(rayHit.pos.x + 0.5, rayHit.pos.y + 0.5, rayHit.pos.z + 0.5);
            if (mouse.left) {
                breakTimer += dt;
                let reqTime = ITEM_PROPS[rayHit.blockId].breakTime;
                const heldProps = ITEM_PROPS[player.hotbar[player.selectedSlotIdx]]; const targetBlockType = ITEM_PROPS[rayHit.blockId].toolType;
                if (heldProps && heldProps.isTool && heldProps.miningSpeedObj && targetBlockType && heldProps.miningSpeedObj[targetBlockType]) reqTime /= heldProps.miningSpeedObj[targetBlockType];

                breakBarContainer.style.display = 'block'; breakBar.style.width = `${Math.min(100, (breakTimer/reqTime)*100)}%`;
                
                if (breakTimer >= reqTime) {
                    // SPAWN HẠT VỠ (PARTICLES) KHI ĐẬP XONG
                    const hitProps = ITEM_PROPS[rayHit.blockId];
                    if (hitProps && hitProps.color) {
                        for(let i=0; i<12; i++) particles.push(new Particle(rayHit.pos.clone().addScalar(0.5), hitProps.color));
                    }
                    world.setBlock(rayHit.pos.x, rayHit.pos.y, rayHit.pos.z, ITEMS.AIR);
                    breakTimer = 0; breakBarContainer.style.display = 'none';
                }
            } else { breakTimer = 0; breakBarContainer.style.display = 'none'; }
        } else { highlightBox.visible = false; breakTimer = 0; breakBarContainer.style.display = 'none'; }
    }
    renderer.render(scene, camera);
}

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight; camera.updateProjectionMatrix(); renderer.setSize(window.innerWidth, window.innerHeight);
});

animate();
</script>
</body>
</html>
