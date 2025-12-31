# norns_midi0s_gol
<!DOCTYPE html>
<html>
<head>
    <title>Fib Journey Touch</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <style>
        body { margin:0; background:black; overflow:hidden; }
        canvas { width:100vw; height:100vh; touch-action:none; }
        #controls { position:absolute; top:10px; left:10px; color:white; }
    </style>
</head>
<body>
    <canvas id="c"></canvas>
    <div id="controls">
        Speed: <input type="range" id="speed" min="50" max="1000" value="200">
    </div>
    <script>
        const canvas = document.getElementById('c');
        const ctx = canvas.getContext('2d');
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        const size = 40;  // Grid cells
        const cellW = canvas.width / size;
        const cellH = canvas.height / size;
        let grid = Array(size).fill().map(() => Array(size).fill(0));
        let history = [];
        const fib_set = [1,2,3,5,8,13,21,34];
        let running = true;

        // Chromatic colors (C red to high violet)
        const colors = ['#ff0000', '#ff8800', '#ffff00', '#00ff00', '#0088ff', '#0000ff', '#8800ff'];

        function randomSeed() {
            for (let y=0; y<size; y++) for (let x=0; x<size; x++) {
                grid[y][x] = Math.random() > 0.9 ? 1 : 0;
            }
        }
        randomSeed();

        function countNeighbors(x, y) {
            let n = 0;
            for (let dy=-1; dy<=1; dy++) for (let dx=-1; dx<=1; dx++) {
                if (dx==0 && dy==0) continue;
                let nx = (x + dx + size) % size;
                let ny = (y + dy + size) % size;
                n += grid[ny][nx];
            }
            return n;
        }

        function evolve() {
            let next = Array(size).fill().map(() => Array(size).fill(0));
            let live = 0;
            for (let y=0; y<size; y++) for (let x=0; x<size; x++) {
                let neighbors = countNeighbors(x, y);
                next[y][x] = fib_set.includes(neighbors) ? 1 : 0;
                live += next[y][x];
            }
            grid = next;
            history.unshift(grid.map(row => row.slice()));  // Save for overlay
            if (history.length > 8) history.pop();

            // Sonification (Web Audio chromatic)
            let note = 60 + (live % 24);  // C4 base + mod
            let freq = 440 * Math.pow(2, (note - 69)/12);
            let osc = new OscillatorNode(actx);
            osc.frequency.value = freq;
            osc.connect(actx.destination);
            osc.start();
            osc.stop(actx.currentTime + 0.2 + (live / (size*size)));
        }

        const actx = new AudioContext();

        setInterval(() => { if (running) evolve(); draw(); }, document.getElementById('speed').value);

        function draw() {
            ctx.fillStyle = 'black';
            ctx.fillRect(0,0,canvas.width,canvas.height);

            // Base 2D vector
            for (let y=0; y<size; y++) for (let x=0; x<size; x++) {
                if (grid[y][x]) {
                    let hue = countNeighbors(x,y) * 30;  // Chromatic shift
                    ctx.fillStyle = `hsl(${hue}, 100%, 50%)`;  // Red low, violet high
                    ctx.fillRect(x*cellW, y*cellH, cellW-2, cellH-2);
                }
            }

            // Higher D overlay (faded journey)
            ctx.globalAlpha = 0.3;
            for (let i=1; i<history.length; i++) {
                ctx.globalAlpha = 0.3 / i;
                let old = history[i];
                for (let y=0; y<size; y++) for (let x=0; x<size; x++) {
                    if (old[y][x]) {
                        ctx.fillStyle = 'cyan';
                        ctx.fillRect(x*cellW + i*2, y*cellH, cellW-2, cellH-2);  // Shift for 3D feel
                    }
                }
            }
            ctx.globalAlpha = 1;
        }

        // Touch poke
        canvas.addEventListener('pointerdown', e => {
            let x = Math.floor(e.clientX / cellW);
            let y = Math.floor(e.clientY / cellH);
            grid[y][x] = 1 - grid[y][x];
            draw();
        });

        window.onresize = () => { canvas.width = innerWidth; canvas.height = innerHeight; draw(); };
        draw();
    </script>
</body>
</html>

-- lazy_controller.lua
-- Virtual Grid + Arc + Knobs/Sliders MIDI/OSC Controller
-- Lazy build: Params assign targets; touch encoders/screen simulate

local midi_out = midi.connect(1)  -- Change device if needed
local osc_targets = {}  -- Table for OSC sends

function init()
  params:add_group("MIDI/OSC Assign", 20)
  for i=1,16 do  -- Virtual grid buttons
    params:add_binary("grid_"..i, "Grid Btn "..i, 0)
    params:set_action("grid_"..i, function(v) if v==1 then send_grid(i) end end)
  end
  for r=1,4 do  -- Virtual arc rings
    params:add_control("arc_"..r, "Arc Ring "..r, controlspec.new(0,127,'lin',1,64))
    params:set_action("arc_"..r, function(v) send_arc(r, v) end)
  end
  params:add_control("knob1", "Knob 1", controlspec.UNIPOLAR)
  params:add_control("slider1", "Slider 1", controlspec.UNIPOLAR)
end

function send_grid(btn)
  local note = 60 + btn  -- Example: Chromatic
  midi_out:note_on(note, 100, 1)
  clock.run(function() clock.sleep(0.1); midi_out:note_off(note,0,1) end)
  osc.send({"localhost:57120", "/grid", btn})  -- Local or external
end

function send_arc(ring, val)
  midi_out:cc(ring*10, val, 1)  -- CC per ring
  osc.send({"localhost:57120", "/arc/"..ring, val/127})
end

-- Simulate input (for lazy testing; replace with real grid/arc if connected)
enc = function(n,d)
  if n==2 then params:delta("arc_1", d) end
  if n==3 then params:delta("knob1", d/10) end
end

key = function(n,z)
  if n==3 and z==1 then params:set("grid_"..math.random(16),1) end  -- Random poke
end

redraw = function()
  screen.clear()
  screen.level(15)
  screen.move(10,20)
  screen.text("Lazy Controller Active")
  screen.move(10,40)
  screen.text("E2: Arc1 | E3: Knob1 | K3: Random Grid Poke")
  screen.update()
end
