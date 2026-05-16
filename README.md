<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>60fps Local Web Capture</title>
    <style>
        * {
            box-sizing: border-box;
        }
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #000;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
        }
        #v {
            width: 100vw;
            height: 100vh;
            object-fit: contain;
            display: block;
        }
        #hud {
            position: fixed;
            top: 15px;
            left: 15px;
            padding: 10px 20px;
            background: rgba(0, 0, 0, 0.85);
            color: #fff;
            font-size: 14px;
            font-weight: 500;
            border-radius: 6px;
            pointer-events: none;
            z-index: 999999;
            box-shadow: 0 4px 12px rgba(0,0,0,0.5);
            border: 1px solid rgba(255,255,255,0.1);
            transition: color 0.2s ease;
        }
        /* Hidden button for touch/click backup if keyboard isn't present */
        #click-trigger {
            position: fixed;
            bottom: 15px;
            right: 15px;
            padding: 10px 15px;
            background: rgba(255, 255, 255, 0.2);
            color: #fff;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            z-index: 999999;
            font-size: 12px;
        }
        #click-trigger:hover {
            background: rgba(255, 255, 255, 0.3);
        }
    </style>
</head>
<body>

    <video id="v" autoplay playsinline></video>
    <div id="hud">⏳ Requesting Capture Card Permissions...</div>
    <button id="click-trigger">Toggle Record</button>

    <script>
        const videoEl = document.getElementById('v');
        const hud = document.getElementById('hud');
        const clickTrigger = document.getElementById('click-trigger');
        
        let mediaRecorder;
        let recordedChunks = [];
        let isRecording = false;
        let stream = null;

        async function initCapture() {
            try {
                // Request raw 1080p60 video and uncompressed audio
                stream = await navigator.mediaDevices.getUserMedia({
                    video: { 
                        width: 1920, 
                        height: 1080, 
                        frameRate: { ideal: 60, exact: 60 } 
                    },
                    audio: { 
                        echoCancellation: false, 
                        noiseSuppression: false, 
                        autoGainControl: false 
                    }
                });

                videoEl.srcObject = stream;
                hud.innerText = "🔴 Press [SPACEBAR] to Start Recording";
                hud.style.color = '#fff';

            } catch (err) {
                hud.innerText = "❌ Error: " + err.name;
                hud.style.color = '#ff3333';
                alert("Hardware Constraint/Permission Error: " + err.message);
            }
        }

        function toggleRecording() {
            if (!stream) return;

            if (!isRecording) {
                recordedChunks = [];
                let options = { mimeType: 'video/webm;codecs=vp9' };
                
                if (!MediaRecorder.isTypeSupported(options.mimeType)) {
                    options = { mimeType: 'video/webm;codecs=vp8' };
                }
                
                try {
                    mediaRecorder = new MediaRecorder(stream, options);
                } catch (e) {
                    // Fallback to default if container codecs throw errors
                    mediaRecorder = new MediaRecorder(stream);
                }
                
                mediaRecorder.ondataavailable = (event) => {
                    if (event.data && event.data.size > 0) {
                        recordedChunks.push(event.data);
                    }
                };

                mediaRecorder.onstop = () => {
                    hud.innerText = "⏳ Processing Video...";
                    hud.style.color = '#ffff00';
                    
                    const blob = new Blob(recordedChunks, { type: 'video/webm' });
                    const url = URL.createObjectURL(blob);
                    
                    const a = document.createElement('a');
                    a.href = url;
                    a.download = `gameplay_1080p60_${Date.now()}.webm`;
                    document.body.appendChild(a);
                    a.click();
                    
                    setTimeout(() => {
                        document.body.removeChild(a);
                        window.URL.revokeObjectURL(url);
                        hud.innerText = "🔴 Press [SPACEBAR] to Start Recording";
                        hud.style.color = '#fff';
                    }, 150);
                };

                // Force 100ms chunks to embed stable timestamps for 60fps pacing
                mediaRecorder.start(100);
                isRecording = true;
                hud.innerText = "⏺️ RECORDING LIVE (Press SPACEBAR to Stop)";
                hud.style.color = '#ff3333';
            } else {
                mediaRecorder.stop();
                isRecording = false;
            }
        }

        // Keyboard Shortcut Action
        window.onkeydown = (e) => {
            if (e.code === 'Space') {
                e.preventDefault();
                toggleRecording();
            }
        };

        // Click/Touch Backup Action
        clickTrigger.addEventListener('click', (e) => {
            e.blur(); // Remove focus so spacebar doesn't trigger it twice
            toggleRecording();
        });

        // Fire up the feed automatically on page load
        window.addEventListener('DOMContentLoaded', initCapture);
    </script>
</body>
</html>
