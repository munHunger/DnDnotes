<script>
  import { onMount } from "svelte";
  import yaml from "js-yaml";

  let camera = { x: 0, y: 0 };

  let canvas;
  let buffer;
  let notes;
  let noteFetch = window
    .fetch("/notes.yml")
    .then(data => data.text())
    .then(data => yaml.load(data, "utf-8").notes)
    .then(notes =>
      Promise.all(
        notes.map(note =>
          window
            .fetch(note.res)
            .then(data => data.text())
            .then(data => yaml.load(data, "utf-8"))
        )
      )
    )
    .then(n => {
      notes = n;
      return n;
    })
    .then(n => document.fonts.load("20px Indie Flower"))
    .then(d => {
      console.log(d);
      return notes;
    });

  let ctx;
  function render(notes) {
    const ctx = buffer.getContext("2d");
    let boundingBox = calculateBoundingBox();

    buffer.width = boundingBox.max.x - boundingBox.min.x;
    buffer.height = boundingBox.max.y - boundingBox.min.y;
    ctx.translate(-boundingBox.min.x, -boundingBox.min.y);
    // ctx.fillStyle = "#000";
    // ctx.fillRect(-9999, -9999, 9999 * 2, 9999 * 2);
    notes.forEach(note => {
      ctx.fillStyle = "#EEE";
      ctx.strokeStyle = "#AAA";
      ctx.translate(note.position.x, note.position.y);
      ctx.font = "20px Indie Flower";
      ctx.fillText(note.title, 0, 20);
      ctx.font = "16px Indie Flower";
      note.content
        .filter(c => c.type === "text")
        .forEach(text => {
          text.content
            .split("\n")
            .forEach((line, index) =>
              ctx.fillText(
                line,
                text.position.x,
                text.position.y + 20 * 2 + 18 * index
              )
            );
        });
      note.content
        .filter(c => c.type === "shape")
        .forEach(shape => {
          if (shape.position) ctx.translate(shape.position.x, shape.position.y);
          shape.stroke
            .split("\n")
            .map(line => {
              const [x1, y1, x2, y2, sx, sy] = line
                .split(" ")
                .map(part => parseInt(part));
              return { x1, y1, x2, y2, sx, sy };
            })
            .forEach(line => {
              drawLine(line, ctx);
            });
          if (shape.position)
            ctx.translate(-shape.position.x, -shape.position.y);
        });
      note.content
        .filter(c => c.type === "arrow")
        .forEach(arrow => {
          let p = arrow.content.split(" ").map(n => parseInt(n));
          console.log(p);
          ctx.moveTo(p[0], p[1]);
          ctx.quadraticCurveTo(p[2], p[3], p[4], p[5]);
          ctx.stroke();
          drawArrowhead(
            p[4],
            p[5],
            findAngle(p[2], p[3], p[4], p[5]),
            15,
            15,
            ctx
          );
        });
      ctx.translate(-note.position.x, -note.position.y);
      camera = {
        x: boundingBox.min.x + window.innerWidth / 2,
        y: boundingBox.min.y + window.innerHeight / 2
      };

      let latest = notes[notes.length - 1];
      camera.x -= latest.position.x;
      camera.y -= latest.position.y;
      renderCanvas();
    });
  }

  function drawArrowhead(locx, locy, angle, sizex, sizey, ctx) {
    var hx = sizex / 2;
    var hy = sizey / 2;

    ctx.translate(locx, locy);
    ctx.rotate(angle);
    ctx.translate(-hx, -hy);

    ctx.beginPath();
    ctx.moveTo(0, 0);
    ctx.lineTo(0, 1 * sizey);
    ctx.lineTo(1 * sizex, 1 * hy);
    ctx.closePath();
    ctx.fill();

    ctx.translate(hx, hy);
    ctx.rotate(-angle);
    ctx.translate(-locx, -locy);
  }

  // returns radians
  function findAngle(sx, sy, ex, ey) {
    // make sx and sy at the zero point
    return Math.atan2(ey - sy, ex - sx);
  }

  function drawLine(line, ctx) {
    ctx.moveTo(line.x1, line.y1);
    ctx.lineTo(line.x2, line.y2);
    ctx.stroke();

    let dist = Math.sqrt(
      Math.pow(line.x2 - line.x1, 2) + Math.pow(line.y2 - line.y1, 2)
    );
    let spacing = 10;
    let length = 5;
    for (let i = 0; i < dist / spacing; i++) {
      const progress = i / parseInt(dist / spacing);
      let dx = line.x1 + (line.x2 - line.x1) * progress;
      let dy = line.y1 + (line.y2 - line.y1) * progress;

      ctx.moveTo(dx, dy);
      ctx.lineTo(dx + length * line.sx, dy + length * line.sy);
      ctx.stroke();
    }
  }

  let dragging = false;
  let prev;
  function dragCamera(event) {
    const [x, y] = [
      event.x || event.touches[0].pageX,
      event.y || event.touches[0].pageY
    ];
    if (dragging) {
      camera.x += (x - prev.x) * 1;
      camera.y += (y - prev.y) * 1;
      renderCanvas();
      prev = { x, y };
    }
  }

  function renderCanvas() {
    const ctx = canvas.getContext("2d");
    ctx.translate(camera.x, camera.y);
    ctx.clearRect(
      -1000,
      -1000,
      buffer.width * 2 + 1000,
      buffer.height * 2 + 1000
    );
    ctx.drawImage(buffer, 0, 0);
    ctx.translate(-camera.x, -camera.y);
  }

  function startDrag(event) {
    const [x, y] = [
      event.x || event.touches[0].pageX,
      event.y || event.touches[0].pageY
    ];
    prev = { x, y };
    dragging = true;
  }

  function stopDrag() {
    dragging = false;
  }

  function calculateBoundingBox() {
    let min = { x: 1, y: 1 };
    let max = { x: -1, y: -1 };
    notes.forEach(note => {
      const [x1, y1, x2, y2] = note.boundingBox
        .split(" ")
        .map(n => parseInt(n));
      min = {
        x: Math.min(min.x, note.position.x + x1),
        y: Math.min(min.y, note.position.y + y1)
      };
      max = {
        x: Math.max(max.x, note.position.x + x2),
        y: Math.max(max.y, note.position.y + y2)
      };
    });
    return { min, max };
  }

  onMount(() => {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    if (!notes) noteFetch.then(n => render(n));
    else render(notes);
  });
  console.log(
    WebFont.load({
      google: {
        families: ["Indie Flower"]
      }
    })
  );
</script>

<style>
  canvas {
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0px;
    top: 0px;
    background-color: rgb(81, 87, 97);
  }
</style>

<canvas bind:this={buffer} style="display:none" />

<canvas
  bind:this={canvas}
  on:mousedown={startDrag}
  on:touchstart={startDrag}
  on:mouseup={stopDrag}
  on:touchend={stopDrag}
  on:mousemove={dragCamera}
  on:touchmove={dragCamera} />
