
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>DS26_OW Photo Metadata Portal</title>
<script src="https://cdn.jsdelivr.net/npm/tesseract.js@4/dist/tesseract.min.js"></script>
<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css"/>
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<style>
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin:0;
  min-height:100vh;
  display:flex;
  justify-content:center;
  align-items:center;
  background: url('https://images.unsplash.com/photo-1502082553048-f009c37129b9?auto=format&fit=crop&w=1950&q=80') no-repeat center center fixed;
  background-size: cover;
  position: relative;
}
body::before {
  content:"";
  position:absolute;
  top:0; left:0; right:0; bottom:0;
  background: rgba(0,0,0,0.45);
  z-index:0;
}
.container {
  position: relative;
  background: rgba(255,255,255,0.95);
  padding:25px;
  border-radius:20px;
  max-width:500px;
  width:100%;
  box-shadow:0 8px 25px rgba(0,0,0,0.2);
  text-align:center;
  z-index:1;
}
.title-banner {
  background: url('https://i.ibb.co/YTX5jq95/ow.jpg') center/cover no-repeat;
  border-radius:15px;
  padding:15px;
  margin-bottom:20px;
  color:white;
  font-size:35px;
  font-weight:bold;
  text-shadow: 1px 12px 4px rgba(0,0,0,0.7);
}
label {
  display:block;
  text-align:left;
  margin-top:15px;
  font-weight:bold;
  color:#004d40;
  font-size:16px;
}
input, textarea, input[type=range] {
  width:100%;
  padding:12px;
  margin-top:8px;
  border-radius:10px;
  border:1px solid #b2dfdb;
  font-size:16px;
  box-sizing:border-box;
  background: rgba(255,255,255,0.9);
}
img {
  max-width:100%;
  margin-top:15px;
  border-radius:12px;
  border:1px solid #b2dfdb;
}
button {
  cursor:pointer;
  background:#00796b;
  color:white;
  border:none;
  font-size:16px;
  padding:12px;
  margin-top:15px;
  border-radius:12px;
  transition:0.3s;
}
button:hover { background:#004d40; }
#map { height:250px; margin-top:15px; border-radius:12px; border:1px solid #b2dfdb; }
textarea { resize:none; }
</style>
</head>
<body>

<div class="container">
  <div class="title-banner">OSTROM CLIMATE</div>

  <input type="file" id="imageInput" accept="image/*">
  <img id="preview"/>
  <label for="qualitySlider">Compression Quality: <span id="qualityValue">80%</span></label>
  <input type="range" id="qualitySlider" min="10" max="100" value="80">

  <button id="downloadBtn">💾 Download Image</button>

  <h3>OCR Text Output</h3>
  <textarea id="ocrText" rows="5" placeholder="OCR text will appear here..."></textarea>

  <h3>Form Fields</h3>

  <label for="filename">Filename</label>
  <input type="text" id="filename">

  <label for="description">Description</label>
  <input type="text" id="description">

  <label for="software">Software</label>
  <input type="text" id="software" value="Timestamp Camera" readonly>

  <label for="model">Model</label>
  <input type="text" id="model">

  <label for="make">Make</label>
  <input type="text" id="make">

  <label for="latitude">Latitude</label>
  <input type="text" id="latitude">

  <label for="longitude">Longitude</label>
  <input type="text" id="longitude">

  <label for="dateTaken">Date Taken</label>
  <input type="text" id="dateTaken">

  <div id="map"></div>
</div>

<script>
let uploadedFile, originalSrc, compressedDataURL;
let imageElement = document.getElementById("preview");
let downloadBtn = document.getElementById("downloadBtn");
let qualitySlider = document.getElementById("qualitySlider");
let qualityValue = document.getElementById("qualityValue");
let map, marker;

qualitySlider.addEventListener("input", ()=>{
  qualityValue.textContent = qualitySlider.value + "%";
  if(uploadedFile) compressImage();
});

function updateMap() {
  const lat = parseFloat(document.getElementById("latitude").value);
  const lng = parseFloat(document.getElementById("longitude").value);
  if(isNaN(lat) || isNaN(lng)) return;
  if(!map){
    map = L.map('map').setView([lat,lng],13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{maxZoom:19}).addTo(map);
    marker = L.marker([lat,lng]).addTo(map);
  } else {
    map.setView([lat,lng],13);
    marker.setLatLng([lat,lng]);
  }
}

document.getElementById("latitude").addEventListener("input", updateMap);
document.getElementById("longitude").addEventListener("input", updateMap);

document.getElementById("imageInput").addEventListener("change", function(e){
  uploadedFile = e.target.files[0];
  if(!uploadedFile) return;

  document.getElementById("filename").value = "DS26_OW-" + uploadedFile.name;

  const reader = new FileReader();
  reader.onload = function(){
    const img = new Image();
    img.src = reader.result;
    img.onload = function(){
      originalSrc = reader.result;
      readOCR(originalSrc);
      compressImage();
    }
  }
  reader.readAsDataURL(uploadedFile);
});

function compressImage(){
  const img = new Image();
  img.src = originalSrc;
  img.onload = function(){
    const canvas = document.createElement('canvas');
    const maxSize = 1024;
    let width = img.width, height = img.height;
    if(width>height && width>maxSize){ height*=maxSize/width; width=maxSize;}
    if(height>width && height>maxSize){ width*=maxSize/height; height=maxSize;}
    canvas.width = width; canvas.height = height;
    const ctx = canvas.getContext('2d');
    ctx.drawImage(img,0,0,width,height);

    let quality = parseInt(qualitySlider.value)/100;
    compressedDataURL = canvas.toDataURL('image/jpeg', quality);
    imageElement.src = compressedDataURL;
  }
}

downloadBtn.onclick = function() {
  const link = document.createElement('a');
  link.href = compressedDataURL;
  let fname = document.getElementById("filename").value;
  if(!fname.toLowerCase().endsWith(".jpg")) fname += ".jpg";
  link.download = fname;
  link.click();
}

function readOCR(src){
  Tesseract.recognize(src,'eng',{logger:m=>console.log(m)})
  .then(({data:{text}})=>{
    document.getElementById("ocrText").value = text;
    fillMetadataFromOCR(text);
  });
}

function fillMetadataFromOCR(text){
  const lines = text.split("\n").map(l=>l.trim()).filter(l=>l);
  let numbers=[], dateSet=false;
  let descriptionSet=false, modelSet=false, makeSet=false;

  // Fixed software
  document.getElementById("software").value = "Timestamp Camera";

  lines.forEach(line=>{
    const l = line.toLowerCase();
    const matches = line.match(/-?\d{1,3}\.\d+/g);
    if(matches) numbers.push(...matches.map(n=>parseFloat(n).toFixed(6)));

    const dateTimeMatch = line.match(/(\d{2,4}[\/\-]\d{1,2}[\/\-]\d{2,4})(?:\s+(\d{1,2}:\d{2}(?::\d{2})?))?/);
    if(dateTimeMatch && !dateSet){
      let datePart = dateTimeMatch[1];
      let timePart = dateTimeMatch[2] ? dateTimeMatch[2] : "00:00:00";
      if(timePart.length===5) timePart+=":00";
      document.getElementById("dateTaken").value = `${datePart} ${timePart}`;
      dateSet=true;
    }

    if(!descriptionSet && !l.includes("model") && !l.includes("make") && !dateTimeMatch && !matches){
        document.getElementById("description").value = "DS26_OW-" + line;
        descriptionSet=true;
    }

    if(!modelSet && (l.includes("canon")||l.includes("nikon")||l.includes("sony")||l.includes("osmo")||l.includes("gopro"))){
        document.getElementById("model").value=line; modelSet=true;
    }
    if(!makeSet && (l.includes("canon")||l.includes("nikon")||l.includes("sony")||l.includes("dji")||l.includes("gopro"))){
        document.getElementById("make").value=line; makeSet=true;
    }
  });

  if(numbers.length>=2){
    document.getElementById("latitude").value=numbers[0];
    document.getElementById("longitude").value=numbers[1];
  } else if(numbers.length===1){
    document.getElementById("latitude").value=numbers[0];
    document.getElementById("longitude").value="";
  }

  updateMap();
}
</script>

</body>
</html>
