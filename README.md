
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>DS26_OW OCR → EXIF Portal</title>

<script src="https://cdn.jsdelivr.net/npm/tesseract.js@4/dist/tesseract.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/piexifjs@1.0.6/piexif.min.js"></script>
<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css"/>
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>

<style>
body {font-family: 'Segoe UI', sans-serif; margin:0; min-height:100vh; display:flex; justify-content:center; align-items:center; background:url('https://images.unsplash.com/photo-1502082553048-f009c37129b9?auto=format&fit=crop&w=1950&q=80') no-repeat center center fixed; background-size:cover; position:relative;}
body::before {content:""; position:absolute; top:0; left:0; right:0; bottom:0; background: rgba(0,0,0,0.45); z-index:0;}
.container {position: relative; background: rgba(255,255,255,0.95); padding:25px; border-radius:20px; max-width:500px; width:100%; box-shadow:0 8px 25px rgba(0,0,0,0.2); text-align:center; z-index:1;}
.title-banner {background:url('https://images.unsplash.com/photo-1503023345310-bd7c1de61c7d?auto=format&fit=crop&w=800&q=80') center/cover no-repeat; border-radius:15px; padding:15px; margin-bottom:20px; color:white; font-size:35px; font-weight:bold; text-shadow:1px 12px 4px rgba(0,0,0,0.7);}
label {display:block; text-align:left; margin-top:15px; font-weight:bold; color:#004d40; font-size:16px;}
input, textarea, input[type=range] {width:100%; padding:12px; margin-top:8px; border-radius:10px; border:1px solid #b2dfdb; font-size:16px; box-sizing:border-box; background: rgba(255,255,255,0.9);}
img {max-width:100%; margin-top:15px; border-radius:12px; border:1px solid #b2dfdb;}
button {cursor:pointer; background:#00796b; color:white; border:none; font-size:16px; padding:12px; margin-top:15px; border-radius:12px; transition:0.3s;}
button:hover {background:#004d40;}
#map {height:250px; margin-top:15px; border-radius:12px; border:1px solid #b2dfdb;}
textarea {resize:none;}
</style>
</head>
<body>

<div class="container">
  <div class="title-banner">OSTROM CLIMATE</div>

  <input type="file" id="imageInput" accept="image/*">
  <img id="preview"/>
  <label for="qualitySlider">Compression Quality: <span id="qualityValue">80%</span></label>
  <input type="range" id="qualitySlider" min="10" max="100" value="80">

  <button id="downloadBtn">💾 Download Image (EXIF Updated)</button>

  <h3>OCR Text Output</h3>
  <textarea id="ocrText" rows="5" placeholder="OCR text will appear here..."></textarea>

  <h3>Form Fields (EXIF)</h3>
  <label for="filename">Filename</label>
  <input type="text" id="filename">

  <label for="description">Description (from OCR)</label>
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

function updateMap(){
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

  const reader = new FileReader();
  reader.onload = function(){
    originalSrc = reader.result;
    imageElement.src = originalSrc;
    document.getElementById("filename").value = "DS26_OW-" + uploadedFile.name;

    readOCRAndFillEXIF(originalSrc);

    compressImage();
  }
  reader.readAsDataURL(uploadedFile);
});

function readOCRAndFillEXIF(src){
  Tesseract.recognize(src,'eng',{logger:m=>console.log(m)})
  .then(({data:{text}})=>{
    document.getElementById("ocrText").value = text;
    const lines = text.split("\n").map(l=>l.trim()).filter(l=>l);
    if(lines.length>0){
      // Last non-empty line → description
      const lastLine = lines[lines.length-1];
      document.getElementById("description").value = "DS26_OW-" + lastLine;
    }
    // Optional: try to extract GPS, Date, Model from OCR
    const numMatches = text.match(/-?\d{1,3}\.\d+/g);
    if(numMatches && numMatches.length>=2){
      document.getElementById("latitude").value = parseFloat(numMatches[0]).toFixed(6);
      document.getElementById("longitude").value = parseFloat(numMatches[1]).toFixed(6);
      updateMap();
    }
    const dateMatch = text.match(/(\d{2,4}[\/\-]\d{1,2}[\/\-]\d{2,4})\s*(\d{1,2}:\d{2}(?::\d{2})?)/);
    if(dateMatch){
      document.getElementById("dateTaken").value = `${dateMatch[1]} ${dateMatch[2]}`;
    }
  });
}

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

// Download with updated EXIF (piexif)
downloadBtn.onclick = function(){
  if(!compressedDataURL) return;
  let exifObj = {"0th":{},"Exif":{},"GPS":{}};

  // Fill EXIF 0th
  exifObj["0th"][piexif.ImageIFD.Make] = document.getElementById("make").value || "";
  exifObj["0th"][piexif.ImageIFD.Model] = document.getElementById("model").value || "";
  exifObj["0th"][piexif.ImageIFD.Software] = document.getElementById("software").value;

  // Description
  exifObj["0th"][piexif.ImageIFD.ImageDescription] = document.getElementById("description").value || "";

  // DateTime
  if(document.getElementById("dateTaken").value)
    exifObj["0th"][piexif.ImageIFD.DateTime] = document.getElementById("dateTaken").value;

  // GPS
  const lat = parseFloat(document.getElementById("latitude").value);
  const lng = parseFloat(document.getElementById("longitude").value);
  if(!isNaN(lat) && !isNaN(lng)){
    exifObj["GPS"][piexif.GPSIFD.GPSLatitude] = decimalToDMS(lat);
    exifObj["GPS"][piexif.GPSIFD.GPSLatitudeRef] = lat>=0?"N":"S";
    exifObj["GPS"][piexif.GPSIFD.GPSLongitude] = decimalToDMS(lng);
    exifObj["GPS"][piexif.GPSIFD.GPSLongitudeRef] = lng>=0?"E":"W";
  }

  const exifBytes = piexif.dump(exifObj);
  const newData = piexif.insert(exifBytes, compressedDataURL);
  const link = document.createElement('a');
  link.href = newData;
  let fname = document.getElementById("filename").value;
  if(!fname.toLowerCase().endsWith(".jpg")) fname += ".jpg";
  link.download = fname;
  link.click();
}

// Convert decimal to [deg, min, sec] format for piexif
function decimalToDMS(dec){
  dec = Math.abs(dec);
  let deg = Math.floor(dec);
  let min = Math.floor((dec-deg)*60);
  let sec = Math.round((dec-deg-min/60)*3600*100)/100;
  return [deg,min,sec];
}
</script>

</body>
</html>
