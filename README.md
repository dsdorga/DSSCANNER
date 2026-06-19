<!DOCTYPE html>
<html lang="bn">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DS SCANNER - ID Card to A4 PDF</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <script async src="opencv.js" onload="onOpenCvReady();" onerror="onOpenCvError();"></script>
    <link href="https://fonts.googleapis.com/css2?family=Hind+Siliguri:wght@400;600;700&family=Poppins:wght@500;700&display=swap" rel="stylesheet">
    
    <style>
        body {
            font-family: 'Poppins', 'Hind Siliguri', sans-serif;
            background-color: #f4f6f8;
        }
        @media print {
            body * { visibility: hidden; }
            #printArea, #printArea * { visibility: visible; }
            #printArea { position: absolute; left: 0; top: 0; width: 210mm; height: 297mm; margin: 0; padding: 20mm; box-sizing: border-box; background: white; }
            @page { size: A4 portrait; margin: 0; }
        }
        .a4-preview { width: 210mm; min-height: 297mm; background: white; padding: 20mm; box-sizing: border-box; }
        .id-card-img { max-width: 85mm; max-height: 55mm; object-fit: fill; border: 1px solid #111; border-radius: 4px; cursor: pointer; transition: transform 0.2s; }
        .id-card-img:hover { transform: scale(1.02); }
        
        /* ক্যানভাস ও ড্রাগেবল পয়েন্টের সিএসএস */
        #canvasContainer { position: relative; display: inline-block; max-height: 50vh; max-width: 100%; overflow: hidden; }
        .drag-point { position: absolute; width: 26px; height: 26px; background-color: rgba(37, 99, 235, 0.85); border: 2px solid white; border-radius: 50%; cursor: move; transform: translate(-13px, -13px); z-index: 10; box-shadow: 0 0 10px rgba(0,0,0,0.5); }
        .scale-ticks { height: 8px; background-image: repeating-linear-gradient(90deg, #94a3b8 0px, #94a3b8 1px, transparent 1px, transparent 8px); width: 100%; margin-top: 4px; opacity: 0.6; }
    </style>
</head>
<body class="min-h-screen flex flex-col items-center justify-start">

    <div id="splashScreen" class="fixed inset-0 bg-[#fdfdfd] z-50 flex flex-col items-center justify-center cursor-pointer transition-opacity duration-500" onclick="hideSplash()">
        <div class="text-center p-4">
            <div class="max-w-[280px] mx-auto mb-4 drop-shadow-lg">
                <img src="logo.png" alt="DS SCANNER" onerror="this.style.display='none'; document.getElementById('altLogo').classList.remove('hidden')" class="w-full h-auto object-contain">
                <div id="altLogo" class="hidden w-24 h-24 bg-blue-600 rounded-2xl flex items-center justify-center mx-auto shadow-xl shadow-blue-200 animate-bounce">
                    <span class="text-white text-4xl font-bold tracking-tighter">DS</span>
                </div>
            </div>
            <h1 class="text-4xl font-extrabold text-gray-800 tracking-wider font-mono mt-2">DS SCANNER</h1>
            <p class="text-blue-600 font-semibold mt-2 text-sm bg-blue-50 px-4 py-1 rounded-full inline-block">আইডিカード টু এ৪ পিডিএফ</p>
            
            <div id="cvStatus" class="mt-6 text-xs font-semibold text-amber-600 bg-amber-50 px-4 py-2 rounded-xl inline-flex items-center space-x-2">
                <span id="loaderIcon" class="animate-spin">⏳</span>
                <span id="loaderText">OpenCV ইঞ্জিন লোড হচ্ছে, দয়া করে অপেক্ষা করুন...</span>
            </div>
        </div>
    </div>


    <div id="mainWorkspace" class="w-full max-w-xl px-4 py-6 opacity-0 transition-opacity duration-500 hidden">
        
        <div class="flex items-center justify-between mb-6 no-print bg-white p-4 rounded-xl shadow-sm border border-gray-100">
            <div class="flex items-center space-x-3">
                <div class="w-10 h-10 bg-blue-600 rounded-lg flex items-center justify-center text-white font-bold text-sm">DS</div>
                <div>
                    <h2 class="text-lg font-bold text-gray-800 leading-tight">DS SCANNER</h2>
                    <p class="text-xs text-gray-500">ID Card to A4 PDF Converter</p>
                </div>
            </div>
            <span id="engineBadge" class="text-xs font-semibold bg-amber-100 text-amber-700 px-2 py-1 rounded-md">Engine Loading...</span>
        </div>

        <div class="bg-white p-5 rounded-2xl shadow-sm border border-gray-100 mb-6 no-print space-y-4">
            <div>
                <label class="block text-sm font-bold text-gray-700 mb-1">১. সামনের অংশ (Front Side)</label>
                <div id="frontUploadBox" class="border-2 border-dashed border-blue-200 hover:border-blue-400 bg-blue-50/30 rounded-xl py-3 px-4 flex items-center justify-center space-x-3 cursor-pointer transition" onclick="document.getElementById('frontInput').click()">
                    <span id="frontIcon" class="text-xl">📸</span>
                    <img id="frontThumb" class="w-[50px] h-[32px] object-cover rounded border border-gray-300 hidden" src="">
                    <span id="frontTextBtn" class="text-xs font-semibold text-blue-600">ক্লিক করে ছবি আপলোড করুন</span>
                </div>
                <input type="file" id="frontInput" accept="image/*" class="hidden" />
                <div id="frontActionGroup" class="hidden mt-2 flex space-x-2">
                    <button class="flex-1 bg-gray-100 hover:bg-gray-200 text-gray-700 text-xs font-bold py-1.5 rounded-lg transition" onclick="document.getElementById('frontInput').click()">🔄 পরিবর্তন</button>
                    <button class="flex-1 bg-blue-600 hover:bg-blue-700 text-white text-xs font-bold py-1.5 rounded-lg shadow-sm transition" onclick="openCvModal('front')">📐 ৪ কোণা সোজা করুন</button>
                </div>
            </div>
            
            <div>
                <label class="block text-sm font-bold text-gray-700 mb-1">২. পিছনের অংশ (Back Side)</label>
                <div id="backUploadBox" class="border-2 border-dashed border-blue-200 hover:border-blue-400 bg-blue-50/30 rounded-xl py-3 px-4 flex items-center justify-center space-x-3 cursor-pointer transition" onclick="document.getElementById('backInput').click()">
                    <span id="backIcon" class="text-xl">📸</span>
                    <img id="backThumb" class="w-[50px] h-[32px] object-cover rounded border border-gray-300 hidden" src="">
                    <span id="backTextBtn" class="text-xs font-semibold text-blue-600">ক্লিক করে ছবি আপলোড করুন</span>
                </div>
                <input type="file" id="backInput" accept="image/*" class="hidden" />
                <div id="backActionGroup" class="hidden mt-2 flex space-x-2">
                    <button class="flex-1 bg-gray-100 hover:bg-gray-200 text-gray-700 text-xs font-bold py-1.5 rounded-lg transition" onclick="document.getElementById('backInput').click()">🔄 পরিবর্তন</button>
                    <button class="flex-1 bg-blue-600 hover:bg-blue-700 text-white text-xs font-bold py-1.5 rounded-lg shadow-sm transition" onclick="openCvModal('back')">📐 ৪ কোণা সোজা করুন</button>
                </div>
            </div>

            <div class="pt-2 border-t border-gray-100 space-y-2">
                <button id="printBtn" class="w-full bg-green-600 hover:bg-green-700 text-white font-bold py-3 px-4 rounded-xl shadow-md transition disabled:opacity-40" disabled onclick="window.print()">
                    🖨️ PDF সেভ / প্রিন্ট করুন
                </button>
                <button id="resetBtn" class="w-full bg-gray-100 hover:bg-gray-200 text-gray-600 font-semibold py-1.5 px-4 rounded-xl text-xs transition" onclick="resetApp()">
                    🔄 সম্পূর্ণ রিসেট
                </button>
            </div>
        </div>

        <div class="w-full overflow-x-auto border border-gray-200 rounded-2xl shadow-sm bg-white mb-12 flex justify-start p-2 no-print">
            <div id="printArea" class="a4-preview flex flex-col items-center justify-start">
                <div class="mb-10 flex flex-col items-center mt-12">
                    <p id="frontText" class="text-gray-300 text-xs italic mb-2">[সামনের অংশ এখানে লোড হবে]</p>
                    <img id="frontView" class="id-card-img hidden shadow-sm" src="" onclick="openCvModal('front')">
                </div>
                <div class="flex flex-col items-center">
                    <p id="backText" class="text-gray-300 text-xs italic mb-2">[পিছনের অংশ এখানে লোড হবে]</p>
                    <img id="backView" class="id-card-img hidden shadow-sm" src="" onclick="openCvModal('back')">
                </div>
            </div>
        </div>
    </div>


    <div id="opencvModal" class="fixed inset-0 bg-black/85 flex flex-col items-center justify-center p-4 z-50 hidden">
        <div class="bg-white rounded-2xl p-5 w-full max-w-lg flex flex-col shadow-2xl">
            <div class="flex justify-between items-center mb-3">
                <h3 class="text-base font-bold text-gray-800">📐 ৪ কোণা পার্সপেক্টিভ এডজাস্টমেন্ট</h3>
                <span class="text-xs bg-green-50 text-green-700 px-2 py-0.5 rounded font-medium">Local OpenCV Engine</span>
            </div>
            
            <div class="bg-gray-950 rounded-xl overflow-hidden flex items-center justify-center border border-gray-800 p-2">
                <div id="canvasContainer">
                    <canvas id="srcCanvas"></canvas>
                    <svg id="polySvg" class="absolute inset-0 w-full h-full pointer-events-none">
                        <polygon id="svgPoly" points="" style="fill:rgba(37, 99, 235, 0.15); stroke:#2563eb; stroke-width:2.5; fill-rule:nonzero;" />
                    </svg>
                    <div id="p0" class="drag-point"></div>
                    <div id="p1" class="drag-point"></div>
                    <div id="p2" class="drag-point"></div>
                    <div id="p3" class="drag-point"></div>
                </div>
            </div>

            <div class="mt-4 bg-gray-50 p-3 rounded-xl border border-gray-100 space-y-2">
                <div class="flex justify-center space-x-3">
                    <button type="button" class="bg-white hover:bg-gray-100 border border-gray-300 text-gray-700 font-bold py-1.5 px-4 rounded-xl text-xs flex items-center space-x-1 shadow-sm transition active:scale-95" onclick="rotate90Canvas(-90)">
                        <span>⟲</span> <span>Rotate Left</span>
                    </button>
                    <button type="button" class="bg-white hover:bg-gray-100 border border-gray-300 text-gray-700 font-bold py-1.5 px-4 rounded-xl text-xs flex items-center space-x-1 shadow-sm transition active:scale-95" onclick="rotate90Canvas(90)">
                        <span>⟳</span> <span>Rotate Right</span>
                    </button>
                </div>
                <p class="text-[10px] text-gray-400 text-center">💡 চারটি গোল বিন্দু টেনে নিখুঁតভাবে আইডি কার্ডের ৪টি কণার সাথে মিলিয়ে নিন।</p>
            </div>

            <div class="mt-5 flex justify-end space-x-2">
                <button class="bg-gray-200 hover:bg-gray-300 text-gray-700 font-bold py-2.5 px-5 rounded-xl text-sm transition" onclick="closeCvModal()">বাতিল</button>
                <button class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2.5 px-6 rounded-xl text-sm shadow-md transition" onclick="processPerspective()">সম্পন্ন করুন</button>
            </div>
        </div>
    </div>

    <script>
        let isOpenCvReady = false;
        let currentTarget = '';
        let originalImages = { front: '', back: '' };
        
        let imgElement = new Image();
        let pts = []; 
        let activePoint = null;
        let startX, startY;
        let currentRotation = 0; // ছবি কত ডিগ্রি ঘোরেল আছে তা ট্র্যাক রাখার জন্য

        // OpenCV লোকাল ফাইল সফলভাবে লোড হলে এই ফাংশনটি রান হবে
        function onOpenCvReady() {
            isOpenCvReady = true;
            document.getElementById('loaderIcon').innerText = "✅";
            document.getElementById('loaderText').innerHTML = "ইঞ্জিন পুরোপুরি রেডি! প্রবেশ করতে টাচ করুন।";
            
            const badge = document.getElementById('engineBadge');
            badge.innerText = "OpenCV Local Ready";
            badge.className = "text-xs font-semibold bg-green-100 text-green-700 px-2 py-1 rounded-md";
        }

        function onOpenCvError() {
            document.getElementById('loaderText').innerHTML = "❌ opencv.js ফাইলটি খুঁজে পাওয়া যায়নি! ফাইলটি index.html এর পাশে রাখুন।";
        }

        function hideSplash() {
            const splash = document.getElementById('splashScreen');
            const workspace = document.getElementById('mainWorkspace');
            if(!splash.classList.contains('pointer-events-none')) {
                splash.classList.add('opacity-0', 'pointer-events-none');
                workspace.classList.remove('hidden');
                setTimeout(() => {
                    workspace.classList.remove('opacity-0');
                    splash.style.display = 'none';
                }, 500);
            }
        }

        document.getElementById('frontInput').addEventListener('change', (e) => loadImage(e, 'front'));
        document.getElementById('backInput').addEventListener('change', (e) => loadImage(e, 'back'));

        function loadImage(event, target) {
            const file = event.target.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = function(e) {
                    originalImages[target] = e.target.result;
                    document.getElementById(`${target}UploadBox`).classList.add('bg-green-50/20', 'border-green-300');
                    document.getElementById(`${target}Icon`).classList.add('hidden');
                    
                    const thumb = document.getElementById(`${target}Thumb`);
                    thumb.src = e.target.result;
                    thumb.classList.remove('hidden');
                    
                    document.getElementById(`${target}TextBtn`).innerText = "✅ আপলোড সম্পন্ন";
                    document.getElementById(`${target}ActionGroup`).classList.remove('hidden');
                    
                    openCvModal(target);
                };
                reader.readAsDataURL(file);
            }
        }

        function openCvModal(target) {
            if (!originalImages[target]) return;
            currentTarget = target;
            currentRotation = 0; // প্রতিবার নতুন করে ওপেন করলে রোটেশন জিরো হবে

            const modal = document.getElementById('opencvModal');
            modal.classList.remove('hidden');

            imgElement.src = originalImages[target];
            imgElement.onload = function() {
                drawCanvasWithRotation();
            }
        }

        // ছবি ঘোরানোসহ ক্যানভাস ড্র করার নতুন ফাংশন
        function drawCanvasWithRotation() {
            const canvas = document.getElementById('srcCanvas');
            const ctx = canvas.getContext('2d');
            
            let maxW = Math.min(window.innerWidth - 60, 420);
            
            // ৯০ বা ২৭০ ডিগ্রি ঘুরলে উইডথ আর হাইট অদলবদল হবে
            let is90 = (currentRotation % 180 !== 0);
            let origW = is90 ? imgElement.height : imgElement.width;
            let origH = is90 ? imgElement.width : imgElement.height;
            
            let scale = maxW / origW;
            if(origH * scale > window.innerHeight * 0.45) {
                scale = (window.innerHeight * 0.45) / origH;
            }
            
            canvas.width = origW * scale;
            canvas.height = origH * scale;
            
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.save();
            
            // ক্যানভাসের মাঝখানে ট্রান্সলেট করে ঘোরানো
            ctx.translate(canvas.width / 2, canvas.height / 2);
            ctx.rotate((currentRotation * Math.PI) / 180);
            
            // ছবি ড্র করা
            ctx.drawImage(imgElement, -imgElement.width * scale / 2, -imgElement.height * scale / 2, imgElement.width * scale, imgElement.height * scale);
            ctx.restore();

            // ডিফল্ট ৪টি পয়েন্ট রিসেট করা
            let w = canvas.width;
            let h = canvas.height;
            pts = [
                {x: w * 0.15, y: h * 0.15}, 
                {x: w * 0.85, y: h * 0.15}, 
                {x: w * 0.85, y: h * 0.85}, 
                {x: w * 0.15, y: h * 0.85}  
            ];

            updatePointsUI();
        }

        function updatePointsUI() {
            pts.forEach((p, i) => {
                const el = document.getElementById('p' + i);
                el.style.left = p.x + 'px';
                el.style.top = p.y + 'px';
            });

            const poly = document.getElementById('svgPoly');
            poly.setAttribute('points', `${pts[0].x},${pts[0].y} ${pts[1].x},${pts[1].y} ${pts[2].x},${pts[2].y} ${pts[3].x},${pts[3].y}`);
        }

        // টাচ এবং মাউস ড্রাগ লজিক
        document.querySelectorAll('.drag-point').forEach(el => {
            el.addEventListener('mousedown', startDrag);
            el.addEventListener('touchstart', startDrag, {passive: false});
        });

        function startDrag(e) {
            e.preventDefault();
            activePoint = e.target.id;
            const clientX = e.type.includes('touch') ? e.touches[0].clientX : e.clientX;
            const clientY = e.type.includes('touch') ? e.touches[0].clientY : e.clientY;
            
            const rect = document.getElementById('canvasContainer').getBoundingClientRect();
            startX = clientX - rect.left;
            startY = clientY - rect.top;

            document.addEventListener('mousemove', dragMove);
            document.addEventListener('touchmove', dragMove, {passive: false});
            document.addEventListener('mouseup', endDrag);
            document.addEventListener('touchend', endDrag);
        }

        function dragMove(e) {
            if (!activePoint) return;
            e.preventDefault();
            const clientX = e.type.includes('touch') ? e.touches[0].clientX : e.clientX;
            const clientY = e.type.includes('touch') ? e.touches[0].clientY : e.clientY;
            
            const container = document.getElementById('canvasContainer');
            const rect = container.getBoundingClientRect();
            
            let x = clientX - rect.left;
            let y = clientY - rect.top;

            x = Math.max(0, Math.min(x, container.clientWidth));
            y = Math.max(0, Math.min(y, container.clientHeight));

            let idx = parseInt(activePoint.replace('p', ''));
            pts[idx].x = x;
            pts[idx].y = y;

            updatePointsUI();
        }

        function endDrag() {
            activePoint = null;
            document.removeEventListener('mousemove', dragMove);
            document.removeEventListener('touchmove', dragMove);
        }

        // ছবি ৯০ ডিগ্রি ডানে বা বামে ঘোরানোর বাটন লজিক
        function rotate90Canvas(degree) {
            currentRotation = (currentRotation + degree) % 360;
            if (currentRotation < 0) currentRotation += 360;
            drawCanvasWithRotation();
        }

        // =========================================================
        // 🔮 ৪ কোণা সোজা করার OpenCV কোড
        // =========================================================
        function processPerspective() {
            if (!isOpenCvReady) {
                alert("OpenCV ইঞ্জিন এখনো পুরোপুরি রেডি হয়নি!");
                return;
            }

            // ১. ক্যানভাস থেকে বর্তমান ডিসপ্লে হওয়া ছবি ম্যাট্রিক্সে নেওয়া
            let src = cv.imread('srcCanvas');
            let dst = new cv.Mat();
            
            // আইডি কার্ডের এ৪ পেজ রেশিও সাইজ (৮৫:৫৫)
            let targetW = 1012;
            let targetH = 638;
            let dsize = new cv.Size(targetW, targetH);

            // ২. ৪টি পয়েন্টের কোঅর্ডিনেট নেওয়া
            let srcTri = cv.matFromArray(4, 1, cv.CV_32FC2, [
                pts[0].x, pts[0].y, 
                pts[1].x, pts[1].y, 
                pts[2].x, pts[2].y, 
                pts[3].x, pts[3].y  
            ]);

            // ৩. টার্গেট সোজা ফ্রেম পজিশন
            let dstTri = cv.matFromArray(4, 1, cv.CV_32FC2, [
                0, 0,
                targetW, 0,
                targetW, targetH,
                0, targetH
            ]);

            // ৪. OpenCV ওয়ার্প পার্সপেক্টিভ রান করা
            let M = cv.getPerspectiveTransform(srcTri, dstTri);
            cv.warpPerspective(src, dst, M, dsize, cv.INTER_LINEAR, cv.BORDER_CONSTANT, new cv.Scalar());

            // ৫. রেজাল্ট প্রিভিউতে পাঠানো
            let resultCanvas = document.createElement('canvas');
            cv.imshow(resultCanvas, dst);
            const finalDataURL = resultCanvas.toDataURL('image/jpeg', 0.95);

            const viewImg = document.getElementById(`${currentTarget}View`);
            const textPlaceholder = document.getElementById(`${currentTarget}Text`);

            viewImg.src = finalDataURL;
            viewImg.classList.remove('hidden');
            textPlaceholder.classList.add('hidden');
            
            document.getElementById(`${currentTarget}Thumb`).src = finalDataURL;

            // মেমোরি ফ্রি করা
            src.delete(); dst.delete(); M.delete(); srcTri.delete(); dstTri.delete();

            closeCvModal();
            checkReadyState();
        }

        function closeCvModal() {
            document.getElementById('opencvModal').classList.add('hidden');
        }

        function checkReadyState() {
            const frontSrc = document.getElementById('frontView').src;
            const backSrc = document.getElementById('backView').src;
            const printBtn = document.getElementById('printBtn');
            
            if (frontSrc && backSrc && !document.getElementById('frontView').classList.contains('hidden') && !document.getElementById('backView').classList.contains('hidden')) {
                printBtn.disabled = false;
            } else {
                printBtn.disabled = true;
            }
        }

        function resetApp() {
            location.reload(); 
        }
    </script>
</body>
</html>
