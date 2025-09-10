<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Animador de IA</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #1a202c;
            color: #e2e8f0;
        }
        .container {
            max-width: 800px;
        }
        .loading-spinner {
            border: 4px solid rgba(255, 255, 255, 0.3);
            border-top: 4px solid #3498db;
            border-radius: 50%;
            width: 40px;
            height: 40px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        .message-box {
            background-color: #2d3748;
            border: 1px solid #4a5568;
            padding: 1rem;
            border-radius: 0.5rem;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
            display: none;
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 1000;
        }
    </style>
</head>
<body class="bg-gray-900 text-gray-100 min-h-screen flex flex-col items-center p-4">

    <!-- Contenedor Principal -->
    <div class="container bg-gray-800 p-8 rounded-lg shadow-xl w-full">
        <h1 class="text-4xl font-bold text-center text-blue-400 mb-6">Animador de IA</h1>
        <p class="text-center text-gray-400 mb-8">
            Describe el personaje y la acción para generar una animación simple.
            <br>
            Ejemplos: "un gato feliz saltando", "un robot amigable saludando"
        </p>

        <!-- Formulario de Entrada -->
        <div class="flex flex-col md:flex-row gap-4 mb-6">
            <input type="text" id="promptInput" placeholder="Describe la animación..." class="flex-grow p-3 rounded-lg bg-gray-700 border-2 border-gray-600 focus:outline-none focus:border-blue-400">
            <input type="number" id="framesInput" value="3" min="1" max="10" class="p-3 rounded-lg bg-gray-700 border-2 border-gray-600 focus:outline-none focus:border-blue-400 w-24">
            <button id="generateBtn" class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-lg transition-colors duration-200">Generar</button>
        </div>

        <!-- Área de visualización de la animación -->
        <div id="animationContainer" class="flex flex-col items-center justify-center p-6 bg-gray-700 rounded-lg h-[400px] overflow-hidden relative">
            <div id="loading" class="hidden loading-spinner"></div>
            <img id="animationImg" class="w-full h-full object-contain" src="" alt="Animación generada por IA">
            <p id="placeholderText" class="text-gray-400 text-center">
                Tu animación aparecerá aquí.
            </p>
        </div>
    </div>

    <!-- Mensaje de error/información -->
    <div id="messageBox" class="message-box">
        <p id="messageText" class="text-center"></p>
        <button id="closeBtn" class="mt-4 w-full bg-blue-600 hover:bg-blue-700 text-white py-2 rounded-lg">Cerrar</button>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            // Variables de la UI
            const promptInput = document.getElementById('promptInput');
            const framesInput = document.getElementById('framesInput');
            const generateBtn = document.getElementById('generateBtn');
            const loading = document.getElementById('loading');
            const animationImg = document.getElementById('animationImg');
            const placeholderText = document.getElementById('placeholderText');
            const messageBox = document.getElementById('messageBox');
            const messageText = document.getElementById('messageText');
            const closeBtn = document.getElementById('closeBtn');

            // Estado de la animación
            let images = [];
            let currentFrame = 0;
            let animationInterval = null;
            const FRAME_RATE = 200; // Milisegundos por fotograma

            // Manejador del botón de generar
            generateBtn.addEventListener('click', async () => {
                const prompt = promptInput.value.trim();
                const numFrames = framesInput.value;

                if (!prompt) {
                    showMessage('Por favor, ingresa una descripción para la animación.');
                    return;
                }

                if (numFrames < 1 || numFrames > 10) {
                    showMessage('El número de fotogramas debe ser entre 1 y 10.');
                    return;
                }

                // Reiniciar el estado
                stopAnimation();
                images = [];
                currentFrame = 0;
                animationImg.src = "";
                placeholderText.classList.add('hidden');
                loading.classList.remove('hidden');

                const prompts = generatePrompts(prompt, numFrames);

                try {
                    images = await generateImages(prompts);
                    loading.classList.add('hidden');
                    if (images.length > 0) {
                        startAnimation();
                    } else {
                        placeholderText.classList.remove('hidden');
                        showMessage('No se pudieron generar las imágenes. Inténtalo de nuevo.');
                    }
                } catch (error) {
                    console.error('Error generando imágenes:', error);
                    loading.classList.add('hidden');
                    placeholderText.classList.remove('hidden');
                    showMessage('Ocurrió un error al generar la animación. Por favor, revisa la consola para más detalles.');
                }
            });

            // Función para generar los prompts incrementales
            function generatePrompts(basePrompt, num) {
                const prompts = [];
                for (let i = 0; i < num; i++) {
                    const progress = Math.round((i / (num - 1)) * 100);
                    // Añade un texto de progreso para guiar la IA
                    prompts.push(`${basePrompt}, frame ${i+1} de ${num}, with a slight change to show motion, progression is ${progress}% complete`);
                }
                return prompts;
            }

            // Función para llamar a la API de generación de imágenes
            async function generateImages(prompts) {
                const generatedImages = [];
                for (const prompt of prompts) {
                    let success = false;
                    let retries = 0;
                    const maxRetries = 3;
                    const baseDelay = 1000;

                    while (!success && retries < maxRetries) {
                        try {
                            const payload = {
                                instances: { prompt: prompt },
                                parameters: { "sampleCount": 1 }
                            };
                            const apiKey = ""; // Canvas proporcionará la clave
                            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-002:predict?key=${apiKey}`;
                            const response = await fetch(apiUrl, {
                                method: 'POST',
                                headers: { 'Content-Type': 'application/json' },
                                body: JSON.stringify(payload)
                            });

                            if (!response.ok) {
                                throw new Error(`Error HTTP: ${response.status}`);
                            }

                            const result = await response.json();
                            if (result.predictions && result.predictions.length > 0 && result.predictions[0].bytesBase64Encoded) {
                                const imageUrl = `data:image/png;base64,${result.predictions[0].bytesBase64Encoded}`;
                                generatedImages.push(imageUrl);
                                success = true;
                            } else {
                                throw new Error('Respuesta de la API inesperada.');
                            }
                        } catch (error) {
                            console.error(`Intento ${retries + 1} fallido para el prompt: "${prompt}". Error: ${error}`);
                            retries++;
                            if (retries < maxRetries) {
                                const delay = baseDelay * Math.pow(2, retries);
                                await new Promise(res => setTimeout(res, delay));
                            } else {
                                showMessage('Error al generar una de las imágenes. La animación podría estar incompleta.');
                                generatedImages.push(null); // Marcar como fallido
                            }
                        }
                    }
                }
                return generatedImages.filter(img => img !== null);
            }

            // Funciones de control de la animación
            function startAnimation() {
                if (images.length === 0) return;
                currentFrame = 0;
                animationImg.src = images[currentFrame];
                animationInterval = setInterval(() => {
                    currentFrame = (currentFrame + 1) % images.length;
                    animationImg.src = images[currentFrame];
                }, FRAME_RATE);
            }

            function stopAnimation() {
                if (animationInterval) {
                    clearInterval(animationInterval);
                    animationInterval = null;
                }
            }

            // Manejador de la caja de mensajes
            function showMessage(text) {
                messageText.textContent = text;
                messageBox.style.display = 'block';
            }

            closeBtn.addEventListener('click', () => {
                messageBox.style.display = 'none';
            });
        });
    </script>

</body>
</html>
