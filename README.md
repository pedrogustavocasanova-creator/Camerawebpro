<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Câmera Pro - Hosted</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <style>
        * {
            box-sizing: border-box;
            -webkit-tap-highlight-color: transparent;
        }

        body, html {
            background-color: #000;
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            position: fixed;
            font-family: sans-serif;
        }

        /* Container do vídeo no fundo */
        .camera-bg {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 1;
            background: #111;
        }

        #video-stream {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }

        /* Camada de Interface (Sempre por cima) */
        .ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: 10; /* Garante que os botões fiquem visíveis */
            display: flex;
            flex-direction: column;
            pointer-events: none; /* Deixa cliques passarem para o vídeo se necessário */
        }

        .top-bar {
            padding: 20px;
            pointer-events: auto;
        }

        .bottom-controls {
            margin-top: auto;
            height: 160px;
            background: linear-gradient(to top, rgba(0,0,0,0.8), transparent);
            display: flex;
            align-items: center;
            justify-content: space-around;
            padding-bottom: 20px;
            pointer-events: auto;
        }

        .shutter-btn {
            width: 80px;
            height: 80px;
            background: white;
            border-radius: 50%;
            border: 6px solid rgba(255,255,255,0.3);
            cursor: pointer;
            box-shadow: 0 0 20px rgba(0,0,0,0.5);
        }

        .shutter-btn:active {
            transform: scale(0.9);
            background: #ccc;
        }

        .circle-icon-btn {
            width: 55px;
            height: 55px;
            border-radius: 50%;
            background: rgba(255,255,255,0.1);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255,255,255,0.2);
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            cursor: pointer;
            background-size: cover;
            background-position: center;
        }

        .hidden-view {
            display: none !important;
        }

        .flash-overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: white;
            z-index: 100;
            opacity: 0;
            pointer-events: none;
        }

        .animate-flash {
            animation: flashAnim 0.3s forwards;
        }

        @keyframes flashAnim {
            0% { opacity: 0; }
            20% { opacity: 1; }
            100% { opacity: 0; }
        }
    </style>
</head>
<body>

    <div id="flash" class="flash-overlay"></div>

    <!-- TELA DE PERMISSÃO -->
    <div id="permission-view" class="fixed inset-0 z-[200] bg-black flex items-center justify-center p-8 text-center">
        <div class="max-w-xs">
            <div class="w-20 h-20 bg-blue-600 rounded-2xl flex items-center justify-center mx-auto mb-6 rotate-12">
                <i class="fas fa-camera text-3xl text-white"></i>
            </div>
            <h1 class="text-white text-2xl font-bold mb-4">Acessar Câmera</h1>
            <p class="text-gray-400 mb-8 text-sm">Clique no botão abaixo para liberar os controles de captura.</p>
            <button id="enable-btn" class="w-full bg-blue-600 text-white font-bold py-4 rounded-xl shadow-lg active:scale-95 transition-all">
                ABRIR CÂMERA
            </button>
        </div>
    </div>

    <!-- BACKGROUND DA CÂMERA -->
    <div class="camera-bg">
        <video id="video-stream" autoplay playsinline muted></video>
    </div>

    <!-- INTERFACE DO USUÁRIO -->
    <div class="ui-layer">
        <div class="top-bar">
            <div class="inline-flex items-center gap-2 bg-black/40 backdrop-blur-md px-4 py-2 rounded-full border border-white/10">
                <div class="w-2 h-2 bg-red-500 rounded-full animate-pulse"></div>
                <span class="text-white text-[10px] font-black uppercase tracking-widest">Live Browser</span>
            </div>
        </div>

        <div class="bottom-controls">
            <!-- Galeria -->
            <button id="gallery-trigger" class="flex flex-col items-center gap-2">
                <div id="latest-thumb" class="circle-icon-btn">
                    <i class="fas fa-images"></i>
                </div>
                <span class="text-[9px] text-white font-bold uppercase">Galeria</span>
            </button>

            <!-- Gatilho -->
            <button id="shutter-btn" class="shutter-btn"></button>

            <!-- Configurações (Download Rápido da Última) -->
            <div class="flex flex-col items-center gap-2 opacity-50">
                <div class="circle-icon-btn">
                    <i class="fas fa-cog"></i>
                </div>
                <span class="text-[9px] text-white font-bold uppercase">Ajustes</span>
            </div>
        </div>
    </div>

    <!-- TELA DE GALERIA / DOWNLOADS -->
    <div id="gallery-view" class="hidden-view fixed inset-0 z-[300] bg-black flex flex-col">
        <header class="p-6 flex items-center justify-between border-b border-white/10">
            <div class="flex items-center gap-4">
                <button id="close-gallery" class="text-white p-2"><i class="fas fa-arrow-left"></i></button>
                <h2 class="text-white font-bold">Meus Downloads</h2>
            </div>
            <span id="photo-count" class="text-xs text-blue-400 font-bold">0 FOTOS</span>
        </header>
        
        <div id="gallery-list" class="flex-grow overflow-y-auto p-4 space-y-4 pb-20">
            <!-- Lista de fotos -->
        </div>

        <div id="empty-state" class="absolute inset-0 flex items-center justify-center text-zinc-700 pointer-events-none">
            <p>Nenhuma foto capturada</p>
        </div>
    </div>

    <canvas id="capture-canvas" class="hidden"></canvas>

    <script>
        const video = document.getElementById('video-stream');
        const shutterBtn = document.getElementById('shutter-btn');
        const enableBtn = document.getElementById('enable-btn');
        const permissionView = document.getElementById('permission-view');
        const galleryTrigger = document.getElementById('gallery-trigger');
        const latestThumb = document.getElementById('latest-thumb');
        const galleryView = document.getElementById('gallery-view');
        const closeGallery = document.getElementById('close-gallery');
        const galleryList = document.getElementById('gallery-list');
        const flash = document.getElementById('flash');

        let db;
        let photos = [];

        // Inicializar Banco de Dados
        const request = indexedDB.open("HostCameraDB", 1);
        request.onupgradeneeded = (e) => e.target.result.createObjectStore('photos', { keyPath: 'id', autoIncrement: true });
        request.onsuccess = (e) => {
            db = e.target.result;
            loadPhotos();
        };

        async function startCamera() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { facingMode: "environment", width: { ideal: 1920 }, height: { ideal: 1080 } },
                    audio: false
                });
                video.srcObject = stream;
                video.onloadedmetadata = () => {
                    video.play();
                    permissionView.classList.add('hidden-view');
                };
            } catch (err) {
                alert("Erro de Câmera: " + err.message);
            }
        }

        function capturePhoto() {
            flash.classList.remove('animate-flash');
            void flash.offsetWidth;
            flash.classList.add('animate-flash');

            const canvas = document.getElementById('capture-canvas');
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            const ctx = canvas.getContext('2d');
            ctx.drawImage(video, 0, 0);

            const dataUrl = canvas.toDataURL('image/jpeg', 0.95);
            const entry = {
                data: dataUrl,
                name: `FOTO_${Date.now()}.jpg`,
                time: new Date().toLocaleTimeString()
            };

            const store = db.transaction(['photos'], 'readwrite').objectStore('photos');
            store.add(entry).onsuccess = loadPhotos;
        }

        function loadPhotos() {
            const store = db.transaction(['photos'], 'readonly').objectStore('photos');
            store.getAll().onsuccess = (e) => {
                photos = e.target.result;
                renderUI();
            };
        }

        function renderUI() {
            document.getElementById('photo-count').innerText = `${photos.length} FOTOS`;
            if (photos.length > 0) {
                const last = photos[photos.length - 1];
                latestThumb.style.backgroundImage = `url(${last.data})`;
                latestThumb.innerHTML = '';
                document.getElementById('empty-state').classList.add('hidden-view');
            }

            galleryList.innerHTML = '';
            [...photos].reverse().forEach(p => {
                const div = document.createElement('div');
                div.className = "bg-zinc-900/80 p-4 rounded-3xl flex items-center justify-between border border-white/5";
                div.innerHTML = `
                    <div class="flex items-center gap-4">
                        <img src="${p.data}" class="w-16 h-16 object-cover rounded-xl">
                        <div>
                            <p class="text-white font-bold text-sm">${p.name}</p>
                            <p class="text-[10px] text-zinc-500">${p.time}</p>
                        </div>
                    </div>
                    <button onclick="downloadAction('${p.data}', '${p.name}')" class="bg-blue-600 text-white px-5 py-3 rounded-2xl text-xs font-black">
                        BAIXAR
                    </button>
                `;
                galleryList.appendChild(div);
            });
        }

        window.downloadAction = (data, name) => {
            fetch(data)
                .then(res => res.blob())
                .then(blob => {
                    const url = window.URL.createObjectURL(blob);
                    const a = document.createElement('a');
                    a.href = url;
                    a.download = name;
                    document.body.appendChild(a);
                    a.click();
                    setTimeout(() => {
                        window.URL.revokeObjectURL(url);
                        document.body.removeChild(a);
                    }, 100);
                });
        };

        enableBtn.addEventListener('click', startCamera);
        shutterBtn.addEventListener('click', capturePhoto);
        galleryTrigger.addEventListener('click', () => galleryView.classList.remove('hidden-view'));
        closeGallery.addEventListener('click', () => galleryView.classList.add('hidden-view'));
    </script>
</body>
</html>


