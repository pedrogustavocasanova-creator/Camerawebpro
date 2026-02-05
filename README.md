# Camerawebpro
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Câmera Web Pro</title>
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

        .app-container {
            display: flex;
            flex-direction: column;
            height: 100%;
            width: 100%;
        }

        .preview-area {
            flex: 1;
            position: relative;
            background-color: #111;
            display: flex;
            align-items: center;
            justify-content: center;
            overflow: hidden;
        }

        #video-stream {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }

        .controls-bar {
            height: 150px;
            background-color: #000;
            display: flex;
            align-items: center;
            justify-content: space-around;
            z-index: 50;
            padding-bottom: 20px;
        }

        .shutter-btn {
            width: 80px;
            height: 80px;
            background: white;
            border-radius: 50%;
            border: 6px solid #333;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .shutter-btn:active {
            transform: scale(0.9);
            background: #ddd;
        }

        .gallery-preview-btn {
            width: 55px;
            height: 55px;
            border-radius: 12px;
            background-color: #222;
            background-size: cover;
            background-position: center;
            border: 2px solid #555;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #fff;
        }

        .hidden-view {
            display: none !important;
        }

        /* Efeito visual de captura */
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
            animation: flashKeyframes 0.4s ease-out;
        }

        @keyframes flashKeyframes {
            0% { opacity: 0; }
            20% { opacity: 1; }
            100% { opacity: 0; }
        }
    </style>
</head>
<body>

    <div id="flash" class="flash-overlay"></div>

    <!-- TELA INICIAL -->
    <div id="permission-view" class="fixed inset-0 z-[100] bg-black flex items-center justify-center p-8 text-center">
        <div>
            <div class="w-24 h-24 bg-indigo-600 rounded-3xl flex items-center justify-center mx-auto mb-6 shadow-lg shadow-indigo-500/20">
                <i class="fas fa-camera text-4xl text-white"></i>
            </div>
            <h1 class="text-white text-2xl font-bold mb-2">Web Camera</h1>
            <p id="status-text" class="text-gray-400 mb-10 text-sm">Toque no botão para ativar a câmera e começar a fotografar pelo site.</p>
            <button id="enable-btn" class="w-full max-w-xs bg-white text-black font-black py-5 rounded-2xl active:scale-95 transition-all uppercase tracking-widest">
                Ativar Câmera
            </button>
        </div>
    </div>

    <!-- CAMERA INTERFACE -->
    <div class="app-container">
        <div class="preview-area">
            <video id="video-stream" autoplay playsinline muted></video>
            <div class="absolute top-6 left-6 flex items-center gap-2 bg-black/40 backdrop-blur-md px-4 py-2 rounded-full border border-white/10">
                <div class="w-2 h-2 bg-red-500 rounded-full animate-pulse"></div>
                <span class="text-white text-[10px] font-bold tracking-widest uppercase">Live Browser</span>
            </div>
        </div>

        <div class="controls-bar">
            <!-- Galeria -->
            <button id="gallery-trigger" class="flex flex-col items-center gap-2">
                <div id="latest-thumb" class="gallery-preview-btn">
                    <i class="fas fa-images opacity-40"></i>
                </div>
                <span class="text-[9px] text-gray-500 font-bold uppercase">Galeria</span>
            </button>

            <!-- Disparador -->
            <button id="shutter-btn" class="shutter-btn">
                <div class="w-14 h-14 rounded-full border-2 border-black/10"></div>
            </button>

            <!-- Switcher (Visual) -->
            <div class="flex flex-col items-center gap-2 opacity-30">
                <div class="w-[55px] h-[55px] rounded-full bg-zinc-800 flex items-center justify-center text-white">
                    <i class="fas fa-sync-alt"></i>
                </div>
                <span class="text-[9px] text-gray-500 font-bold uppercase">Trocar</span>
            </div>
        </div>
    </div>

    <!-- GALERIA MODAL -->
    <div id="gallery-view" class="hidden-view fixed inset-0 z-[200] bg-black flex flex-col">
        <header class="p-6 flex items-center justify-between bg-zinc-900 border-b border-white/5">
            <div class="flex items-center gap-4">
                <button id="close-gallery" class="text-white p-2"><i class="fas fa-arrow-left text-xl"></i></button>
                <h2 class="text-white font-bold text-xl">Arquivos</h2>
            </div>
            <span id="photo-count" class="text-xs text-indigo-400 font-bold">0 FOTOS</span>
        </header>
        
        <div id="gallery-list" class="flex-grow overflow-y-auto p-4 space-y-4 pb-24">
            <!-- Itens injetados -->
        </div>

        <div id="empty-state" class="absolute inset-0 flex flex-col items-center justify-center text-zinc-700 pointer-events-none">
            <i class="fas fa-cloud-download-alt text-5xl mb-4 opacity-20"></i>
            <p class="font-bold">Nenhum arquivo para baixar</p>
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
        const photoCountLabel = document.getElementById('photo-count');
        const flash = document.getElementById('flash');

        let db;
        let photos = [];

        // Inicializa Banco Local (Sessão)
        const dbReq = indexedDB.open("SystemCamDB", 1);
        dbReq.onupgradeneeded = (e) => e.target.result.createObjectStore('photos', { keyPath: 'id', autoIncrement: true });
        dbReq.onsuccess = (e) => {
            db = e.target.result;
            loadStoredPhotos();
        };

        async function startSystemCamera() {
            document.getElementById('status-text').innerText = "Iniciando sistema de captura...";
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { facingMode: "environment", width: { ideal: 1920 }, height: { ideal: 1080 } },
                    audio: false
                });
                video.srcObject = stream;
                video.onloadedmetadata = () => {
                    video.play().then(() => {
                        permissionView.classList.add('hidden-view');
                    });
                };
            } catch (err) {
                document.getElementById('status-text').innerHTML = `<span class="text-red-500 font-bold">Acesso Negado.</span><br>Permita a câmera no Chrome.`;
            }
        }

        function takeActionCapture() {
            // Efeito de Flash
            flash.classList.remove('animate-flash');
            void flash.offsetWidth;
            flash.classList.add('animate-flash');

            const canvas = document.getElementById('capture-canvas');
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            const ctx = canvas.getContext('2d');
            ctx.drawImage(video, 0, 0);

            const dataUrl = canvas.toDataURL('image/jpeg', 0.9);
            const entry = {
                data: dataUrl,
                fileName: `IMG_${Date.now()}.jpg`,
                size: (dataUrl.length / 1024).toFixed(1) + " KB",
                date: new Date().toLocaleTimeString()
            };

            const store = db.transaction(['photos'], 'readwrite').objectStore('photos');
            store.add(entry).onsuccess = loadStoredPhotos;
        }

        function loadStoredPhotos() {
            const store = db.transaction(['photos'], 'readonly').objectStore('photos');
            store.getAll().onsuccess = (e) => {
                photos = e.target.result;
                updateAppUI();
            };
        }

        function updateAppUI() {
            photoCountLabel.innerText = `${photos.length} ARQUIVOS`;
            if (photos.length > 0) {
                const last = photos[photos.length - 1];
                latestThumb.style.backgroundImage = `url(${last.data})`;
                latestThumb.innerHTML = '';
                document.getElementById('empty-state').classList.add('hidden-view');
            }

            galleryList.innerHTML = '';
            [...photos].reverse().forEach(p => {
                const item = document.createElement('div');
                item.className = "bg-zinc-900/80 border border-white/5 p-4 rounded-3xl flex items-center justify-between gap-4";
                item.innerHTML = `
                    <div class="flex items-center gap-4 overflow-hidden">
                        <img src="${p.data}" class="w-14 h-14 object-cover rounded-xl border border-white/10">
                        <div class="overflow-hidden">
                            <p class="text-white font-bold text-sm truncate">${p.fileName}</p>
                            <p class="text-[10px] text-zinc-500 uppercase font-bold tracking-tighter">${p.date} • ${p.size}</p>
                        </div>
                    </div>
                    <button onclick="triggerNativeDownload('${p.data}', '${p.fileName}')" 
                            class="bg-indigo-600 hover:bg-indigo-500 text-white px-5 py-3 rounded-2xl flex items-center gap-2 active:scale-90 transition-all">
                        <i class="fas fa-download text-xs"></i>
                        <span class="text-xs font-black uppercase">Baixar</span>
                    </button>
                `;
                galleryList.appendChild(item);
            });
        }

        // --- FUNÇÃO DE DOWNLOAD NATIVO (SIMULA COMPORTAMENTO DE SITE DE DOWNLOAD) ---
        window.triggerNativeDownload = (base64, name) => {
            // 1. Converter Base64 para um Array de bytes (Binary Data)
            const byteString = atob(base64.split(',')[1]);
            const mimeString = base64.split(',')[0].split(':')[1].split(';')[0];
            const ab = new ArrayBuffer(byteString.length);
            const ia = new Uint8Array(ab);
            for (let i = 0; i < byteString.length; i++) {
                ia[i] = byteString.charCodeAt(i);
            }

            // 2. Criar um Blob real (Arquivo em memória que o navegador reconhece como arquivo do sistema)
            const blob = new Blob([ab], { type: 'application/octet-stream' }); // Força o download como arquivo binário
            
            // 3. Gerar URL temporária para o Blob
            const blobUrl = window.URL.createObjectURL(blob);

            // 4. Criar o elemento de link e disparar o comportamento padrão do Chrome
            const link = document.createElement('a');
            link.href = blobUrl;
            link.download = name; // Nome que aparecerá na pasta de downloads
            
            // Adiciona ao documento para o Chrome permitir o clique programático
            document.body.appendChild(link);
            link.click();
            
            // Limpeza: remove o link e revoga a URL para liberar memória do tablet
            setTimeout(() => {
                document.body.removeChild(link);
                window.URL.revokeObjectURL(blobUrl);
            }, 100);

            // Nota: No Chrome para Android, isso disparará a barra de progresso nativa.
        };

        enableBtn.addEventListener('click', startSystemCamera);
        shutterBtn.addEventListener('click', takeActionCapture);
        galleryTrigger.addEventListener('click', () => galleryView.classList.remove('hidden-view'));
        closeGallery.addEventListener('click', () => galleryView.classList.add('hidden-view'));

    </script>
</body>
</html>

