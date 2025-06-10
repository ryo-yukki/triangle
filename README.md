<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8" />
<title>色がゆっくり変わる三角形ネットワーク</title>
<style>body,html{margin:0;overflow:hidden;background:#000;}canvas{display:block;}</style>
</head>
<body>
<canvas id="glcanvas"></canvas>
<script>
(() => {
  const canvas = document.getElementById('glcanvas');
  const gl = canvas.getContext('webgl');

  if (!gl) {
    alert('WebGL非対応です');
    return;
  }

  function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);
  }
  window.addEventListener('resize', resize);
  resize();

  const vertexShaderSource = `
    attribute vec2 a_position;
    attribute vec3 a_color;
    uniform vec2 u_resolution;
    varying vec3 v_color;
    void main() {
      vec2 zeroToOne = a_position / u_resolution;
      vec2 clipSpace = zeroToOne * 2.0 - 1.0;
      gl_Position = vec4(clipSpace * vec2(1, -1), 0, 1);
      v_color = a_color;
    }
  `;

  const fragmentShaderSource = `
    precision mediump float;
    varying vec3 v_color;
    void main() {
      gl_FragColor = vec4(v_color, 1.0);
    }
  `;

  function createShader(type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    if(!gl.getShaderParameter(shader, gl.COMPILE_STATUS)){
      console.error(gl.getShaderInfoLog(shader));
      gl.deleteShader(shader);
      return null;
    }
    return shader;
  }

  function createProgram(vsSource, fsSource) {
    const program = gl.createProgram();
    const vShader = createShader(gl.VERTEX_SHADER, vsSource);
    const fShader = createShader(gl.FRAGMENT_SHADER, fsSource);
    gl.attachShader(program, vShader);
    gl.attachShader(program, fShader);
    gl.linkProgram(program);
    if(!gl.getProgramParameter(program, gl.LINK_STATUS)){
      console.error(gl.getProgramInfoLog(program));
      gl.deleteProgram(program);
      return null;
    }
    return program;
  }

  const program = createProgram(vertexShaderSource, fragmentShaderSource);
  gl.useProgram(program);

  const a_position = gl.getAttribLocation(program, "a_position");
  const a_color = gl.getAttribLocation(program, "a_color");
  const u_resolution = gl.getUniformLocation(program, "u_resolution");

  const POINT_COUNT = 120;
  const points = [];
  const velocities = [];

  for(let i=0; i<POINT_COUNT; i++){
    points.push([Math.random()*canvas.width, Math.random()*canvas.height]);
    velocities.push([(Math.random()-0.5)*0.3, (Math.random()-0.5)*0.2]);
  }

  let triangles = [];

  const positionBuffer = gl.createBuffer();
  const colorBuffer = gl.createBuffer();

  // HSV -> RGB変換
  function hsvToRgb(h, s, v){
    let c = v * s;
    let x = c * (1 - Math.abs((h / 60) % 2 - 1));
    let m = v - c;
    let r=0, g=0, b=0;
    if (0 <= h && h < 60) { r=c; g=x; b=0; }
    else if (60 <= h && h < 120) { r=x; g=c; b=0; }
    else if (120 <= h && h < 180) { r=0; g=c; b=x; }
    else if (180 <= h && h < 240) { r=0; g=x; b=c; }
    else if (240 <= h && h < 300) { r=x; g=0; b=c; }
    else { r=c; g=0; b=x; }
    return [r+m, g+m, b+m];
  }

  function buildTriangles() {
    triangles = [];
    for(let i=0; i<POINT_COUNT; i++){
      const p0 = points[i];
      const dists = [];
      for(let j=0; j<POINT_COUNT; j++){
        if(i !== j){
          const dx = p0[0]-points[j][0];
          const dy = p0[1]-points[j][1];
          const dist = Math.sqrt(dx*dx+dy*dy);
          dists.push({index:j, dist});
        }
      }
      dists.sort((a,b)=>a.dist-b.dist);
      if(dists.length>=2){
        const p1 = points[dists[0].index];
        const p2 = points[dists[1].index];
        triangles.push(...p0, ...p1, ...p2);
      }
    }
  }

  // 三角形数に応じた色の色相を持つ配列
  let triangleHue = [];

  function initTriangleHue() {
    const triCount = POINT_COUNT; // だいたいPOINT_COUNT個の三角形
    triangleHue = [];
    for(let i=0; i<triCount; i++){
      triangleHue.push(Math.random() * 360);
    }
  }
  initTriangleHue();

  // 三角形の色を作る(頂点3つとも同じ色)
  function buildColors() {
    const colorsOut = [];
    const triCount = triangles.length / 6;
    for(let i=0; i<triCount; i++){
      // 色相を少し変化させる
      triangleHue[i] += 0.3;
      if(triangleHue[i] > 360) triangleHue[i] -= 360;
      // 色彩を決めるパラメータ
      const rgb = hsvToRgb(triangleHue[i], 0.8, 0.9);
      for(let v=0; v<3; v++){
        colorsOut.push(...rgb);
      }
    }
    return colorsOut;
  }

  function updatePoints() {
    for(let i=0; i<POINT_COUNT; i++){
      points[i][0] += velocities[i][0];
      points[i][1] += velocities[i][1];

      if(points[i][0]<0 || points[i][0]>canvas.width) velocities[i][0] *= -1;
      if(points[i][1]<0 || points[i][1]>canvas.height) velocities[i][1] *= -1;

      points[i][0] = Math.min(canvas.width, Math.max(0, points[i][0]));
      points[i][1] = Math.min(canvas.height, Math.max(0, points[i][1]));
    }
  }

  function draw(){
    updatePoints();
    buildTriangles();

    const colors = buildColors();

    gl.clearColor(0,0,0,1);
    gl.clear(gl.COLOR_BUFFER_BIT);

    gl.uniform2f(u_resolution, canvas.width, canvas.height);

    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(triangles), gl.DYNAMIC_DRAW);
    gl.enableVertexAttribArray(a_position);
    gl.vertexAttribPointer(a_position, 2, gl.FLOAT, false, 0, 0);

    gl.bindBuffer(gl.ARRAY_BUFFER, colorBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(colors), gl.DYNAMIC_DRAW);
    gl.enableVertexAttribArray(a_color);
    gl.vertexAttribPointer(a_color, 3, gl.FLOAT, false, 0, 0);

    gl.drawArrays(gl.TRIANGLES, 0, triangles.length/2);

    requestAnimationFrame(draw);
  }

  draw();

})();
</script>
</body>
</html>
