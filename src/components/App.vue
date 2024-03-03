<template>
    <div ref="container" class="container manrope-bold py-4">
        <h1 class="mb-3">Cellular Automata Compression</h1>
        <div class="d-flex align-items-center">
            <button @click="switch_mode">{{ editing ? "Edit mode" : "View mode" }}</button>
        </div>
        <div class="d-flex position-relative mt-3">
            <canvas @mousedown="mousedown" @mouseup="mouseup" @mousemove="mousemove" @mouseleave="mouseup"
                @click="click">
            </canvas>
            <div class="controls">
                <span>Step: {{ step }}</span>
                <div class="d-flex">
                    <span class="me-2">Speed: {{ delay }}</span>
                    <input type="range" min="0" max="1000" v-model="delay" @input="change_interval">
                </div>
                <div class="d-flex">
                    <span class="me-2">Zoom: {{ scale }}</span>
                    <input type="range" min="0" max="10" @input="change_zoom" value="1">
                </div>
            </div>
        </div>
        <div class="d-flex flex-column mt-3">
            <div class="d-flex align-items-center">
                <button @click="toggle_simulation">{{ running ? "Stop" : "Start" }}</button>
            </div>
        </div>
        <div class="d-flex flex-column"></div>
    </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const size = ref({
    dpi: 0,
    width: 0,
    height: 0,
})

const cells = ref({});
const editing = ref(true);
const running = ref(false);

const step = ref(0);
const delay = ref(250);
const interval = ref(null);
const container = ref(null);

const cellSize = 32;
const backgroundColor = "white";
const lineColor = "silver";
const lineWidth = 1;

let pressed = false;
let canvas = null;
let ctx = null;

let scale = 1;
let scaleStep = 0.1;
let lastScale = scale;
let maxScale = 10;
let minScale = 0.1;

let zoomPoint = {
    x: 0,
    y: 0
};
let lastZoomPoint = zoomPoint;

let rx = 0,
    ry = 0;

let left = 0,
    right = 0,
    top = 0,
    bottom = 0;

const rules = {
    "0000": 0,
    "0001": 1,
    "0010": 2,
    "0011": 3,
    "0100": 4,
    "0101": 5,
    "0110": 6,
    "0111": 7,
    "1000": 8,
    "1001": 9,
    "1010": 10,
    "1011": 11,
    "1100": 12,
    "1101": 13,
    "1110": 14,
    "1111": 15,
}

function resizeCanvas() {
    // Get maximum width and height
    size.value.dpi = window.devicePixelRatio;
    size.value.width = container.value.clientWidth;
    size.value.height = container.value.clientHeight;

    // Set canvas size
    canvas.width = size.value.width * size.value.dpi;
    canvas.height = size.value.height * size.value.dpi;
}

function calculate() {
    calculateDistancesToCellBorders();
    calculateDrawingPositions();
}

function calculateDistancesToCellBorders() {
    let dx = zoomPoint.x - lastZoomPoint.x + rx * lastScale;
    rx = dx - Math.floor(dx / (lastScale * cellSize)) * lastScale * cellSize;
    rx /= lastScale;

    let dy = zoomPoint.y - lastZoomPoint.y + ry * lastScale;
    ry = dy - Math.floor(dy / (lastScale * cellSize)) * lastScale * cellSize;
    ry /= lastScale;
}

function calculateDrawingPositions() {
    let scaledCellSize = cellSize * scale;

    left = zoomPoint.x - rx * scale;
    right = left + scaledCellSize;
    top = zoomPoint.y - ry * scale;
    bottom = top + scaledCellSize;
}

function draw() {
    ctx.save();
    ctx.scale(size.value.dpi, size.value.dpi);

    ctx.fillStyle = backgroundColor;
    ctx.fillRect(0, 0, size.value.width, size.value.height);

    ctx.lineWidth = lineWidth;
    ctx.translate(-0.5, -0.5);

    ctx.beginPath();

    // Draw red origin lines
    ctx.lineWidth = 1;
    ctx.strokeStyle = "red";
    ctx.moveTo(zoomPoint.x, 0);
    ctx.lineTo(zoomPoint.x, size.value.height);
    ctx.moveTo(0, zoomPoint.y);
    ctx.lineTo(size.value.width, zoomPoint.y);
    ctx.stroke();

    // Draw grid
    let scaledCellSize = cellSize * scale;
    ctx.strokeStyle = lineColor;

    for (let x = left; x > 0; x -= scaledCellSize) {
        ctx.moveTo(x, 0);
        ctx.lineTo(x, size.value.height);
    }

    for (let x = right; x < size.value.width; x += scaledCellSize) {
        ctx.moveTo(x, 0);
        ctx.lineTo(x, size.value.height);
    }

    for (let y = top; y > 0; y -= scaledCellSize) {
        ctx.moveTo(0, y);
        ctx.lineTo(size.value.width, y);
    }

    for (let y = bottom; y < size.value.height; y += scaledCellSize) {
        ctx.moveTo(0, y);
        ctx.lineTo(size.value.width, y);
    }

    // Draw cells
    const x_cells = Math.floor(size.value.width / (cellSize - lineWidth)) - 1;
    const y_cells = Math.floor(size.value.height / (cellSize - lineWidth)) - 1;

    // Get bounds for cells
    let x_scrolled = Math.floor(left / (cellSize));
    let y_scrolled = Math.floor(top / (cellSize));

    // Filter out cells that are out of bounds
    for (let key of Object.keys(cells.value)) {
        let [x, y] = key.split(",").map(Number);
        let val = cells.value[key];

        if (x >= -x_scrolled && x <= x_cells - x_scrolled && y >= -y_scrolled && y <= y_cells - y_scrolled) {
            let x_pos = (x * cellSize) + lineWidth;
            let y_pos = (y * cellSize) + lineWidth;

            ctx.fillStyle = val ? "black" : "white";
            ctx.fillRect(x_pos + left, y_pos + top, cellSize, cellSize);
        }
    }

    ctx.stroke();
    ctx.restore();
}

function update() {
    ctx.clearRect(0, 0, size.value.width * size.value.dpi, size.value.height * size.value.dpi);
    draw();
}

function move(dx, dy) {
    zoomPoint.x += dx;
    zoomPoint.y += dy;
}

function mousedown(event) {
    pressed = true;
}

function mouseup(event) {
    pressed = false;
}

function mousemove(event) {
    if (!pressed) {
        return;
    }

    move(event.movementX, event.movementY);

    // do not recalculate the distances again, this wil lead to wrong drawing
    calculateDrawingPositions();
    update();
}

function wheel(e) {
    zoom(e.deltaY > 0 ? -1 : 1, {
        x: e.offsetX,
        y: e.offsetY
    });

    calculate();
    update();
}

function zoom(amt, point) {
    lastScale = scale;
    scale += amt * scaleStep;

    if (scale < minScale) {
        scale = minScale;
    }

    if (scale > maxScale) {
        scale = maxScale;
    }

    lastZoomPoint = zoomPoint;
    zoomPoint = point;
}


async function click(event) {
    // Check if editing
    if (!editing.value) return;

    // get cell at position
    let x = Math.floor((event.offsetX - left) / (cellSize - lineWidth));
    let y = Math.floor((event.offsetY - top) / (cellSize - lineWidth));

    // Change value of cell
    let key = `${x},${y}`;
    console.log("Cell:", key);

    if (cells.value[key]) delete cells.value[key];
    else cells.value[key] = 1;

    // Redraw
    update();
}

async function switch_mode() {
    editing.value = !editing.value;
}

async function toggle_simulation() {
    editing.value = false;
    running.value = !running.value;

    if (running.value) {
        // Start simulation
        interval.value = setInterval(simulation_step, 1000 - delay.value);
    } else {
        // Stop simulation
        clearInterval(interval.value);
    }
}

async function change_interval() {
    if (!running.value) return;

    clearInterval(interval.value);
    interval.value = setInterval(simulation_step, 1000 - delay.value);
}

async function change_zoom() {
    //
}

function simulation_step() {
    console.log("Step:", step.value);
    // Calculate next step
    let blocks = {};

    // For each cell, find its corresponding 2x2 block
    for (let key of Object.keys(cells.value)) {
        let [x, y] = key.split(",").map(Number);
        let val = cells.value[key];

        let block_x = Math.floor(x / 2);
        let block_y = Math.floor(y / 2);
        let block = `${block_x},${block_y}`;

        // Create block if it does not exist
        if (!blocks[block]) blocks[block] = [0, 0, 0, 0];

        // Add own value to block
        let index = (x % 2) + (y % 2) * 2;
        blocks[block][index] = val;
    }

    // Calculate next step
    for (let key of Object.keys(blocks)) {
        let configuration = blocks[key].join("");
        let rule = rules[configuration];
        enforce_rules(key, rule);
    }

    // Redraw
    update();

    // Increase step
    step.value++;
}

function enforce_rules(block, rule) {
    // Coordinates for the upper left cell on block
    let [x, y] = block.split(",").map(Number);
    x *= 2;
    y *= 2;

    // 16 rules
    switch (rule) {
        case 0:
            x -= 1;
            y -= 1;
            break;
        case 1:
            y -= 1;
            break;
        case 2:
            x += 1;
            y -= 1;
            break;
        case 3:
            x += 2;
            y -= 1;
            break;
        case 4:
            x += 2;
            break;
        case 5:
            x += 2;
            y += 1;
            break;
        case 6:
            x += 2;
            y += 2;
            break;
        case 7:
            x += 1;
            y += 2;
            break;
        case 8:
            y += 2;
            break;
        case 9:
            x -= 1;
            y += 2;
            break;
        case 10:
            x -= 1;
            y += 1;
            break;
        case 11:
            x -= 1;
            break;
        case 12:
            break;
        case 13:
            x += 1;
            break;
        case 14:
            x += 1;
            y += 1;
            break;
        case 15:
            y += 1;
            break;
    }

    // cell to change
    let cell = `${x},${y}`;

    // Change value of cell
    if (cells.value[cell]) delete cells.value[cell];
    else cells.value[cell] = 1;
}

function shuffle(array) {
    let currentIndex = array.length, randomIndex;

    // While there remain elements to shuffle.
    while (currentIndex > 0) {

        // Pick a remaining element.
        randomIndex = Math.floor(Math.random() * currentIndex);
        currentIndex--;

        // And swap it with the current element.
        [array[currentIndex], array[randomIndex]] = [
            array[randomIndex], array[currentIndex]];
    }

    return array;
}

onMounted(() => {
    // Randomize rules values
    let values = Array.from({ length: 16 }, (_, i) => i);
    values = shuffle(values);
    for (let i = 0; i < 16; i++) {
        rules[Object.keys(rules)[i]] = values[i];
    }
    console.log("Rules:", rules);

    // Get canvas and context
    canvas = document.querySelector("canvas");
    ctx = canvas.getContext("2d");

    // Set canvas size
    resizeCanvas();

    // Move slightly
    move(32, 32);
    calculate();

    // Draw grid
    draw();
})
</script>