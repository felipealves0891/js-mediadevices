<template>
	<div id="app">
		<div class="suport" v-show="!suportSpeechRecognition.status">Transcrição de voz para audio não suportado!</div>
		<nav>
			<button class="btn transcription" @click="startTranscrever" :class="{'start-audio': suportSpeechRecognition.start}" title='Transcrever um áudio'> 🎤 </button>
			<button class="btn record-audio" @click="startAudio" :class="{'start-audio': recorder}" title='Enviar um áudio'> 🎤 </button>
			<button class="btn start-video" @click="startCamera" title='Câmera'>Câmera</button>
			<button class="btn stop-video" @click="stopCamera" title='Stop'>Parar</button>
			<button class="btn take-picture" @click="photograph" title='Tirar uma foto'> 📷 </button>
			<button class="btn record-video" @click="recordVideo" title='Gravar vídeo'> ⏺ </button>	
		</nav>
		<div id="transcription"></div>
		<video src="" id="videoFeed" muted autoplay></video>
		<video src="" id="videoRecord" muted autoplay></video>
		<audio src="" id="audioRecord" controls="true" muted autoplay></audio>
		<canvas id="picture-canvas"></canvas>

	</div>
</template>

<script>
export default {
	name: 'App',
	data: function () {
		return {
			videoRecorder: null,
			audioRecorder: null,
			recorder: false,
			suportSpeechRecognition: {
				start: false
			}
		}	
	},
	methods: {
		startCamera: function() {
			navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' }, audio: true })
					.then((stream) => {
						document.getElementById('videoFeed').srcObject = stream
					})
		},
		stopCamera: function() {
			document.getElementById('videoFeed')
					.srcObject
					.getVideoTracks()
					.forEach(track => track.stop())
		},
		photograph: function () {
			// coletamos os elementos que precisamos referenciar
			const canvas = document.getElementById('picture-canvas')
			const context = canvas.getContext('2d')
			const video = document.getElementById('videoFeed')
			// o canvas terá o mesmo tamanho do vídeo
			canvas.width = video.offsetWidth
			canvas.height = video.offsetHeight
			// e então, desenhamos o que houver no vídeo, no canvas
			context.drawImage(video, 0, 0, canvas.width, canvas.height)

			// olha que barbada, o canvas tem um método toBlob!
			canvas.toBlob(function(blob){
				const url = URL.createObjectURL(blob)
				console.log(url);
			}, 'image/jpeg', 0.95)

			this.stopCamera()
		},
		recordVideo: function () {
			let chunks = []
			const videoFeed = document.getElementById('videoFeed')
			// caso não estejamos gravando, começaremos
			if (!this.videoRecorder) {
				// vamos usar o mesmo stream que já está ativo em nosso vídeo
				const stream = videoFeed.srcObject

				this.videoRecorder = new MediaRecorder(stream)
				this.videoRecorder.start(3000)

				// sempre que um novo chunk estiver pronto, ou
				// quando a gravação for finalizada
				this.videoRecorder.ondataavailable = event => {
					// nós simplesmente armazenaremos o novo chunk
					chunks.push(event.data)
				}
		
				// e, finalmente, quando a gravação é finalizada
				this.videoRecorder.onstop = event => {
					// nós montaremos um blob a partir de nossos chunks
					// nesse caso, no formato de vídeo/mp4
					let blob = new Blob(chunks, { 'type' : 'video/mp4' })
					const url = URL.createObjectURL(blob)
					document.getElementById('videoRecord').src= url;
					// e podemos usar o nosso blob, aqui, à vontade
					console.log(event)
					console.log(blob)
				}

			} else {
				// se o vídeo estava sendo gravado, quer dizer que o usuário
				// quer finalizar a gravação
				this.videoRecorder.stop()
				// e podemos também finalizar a câmera
				this.stopCamera()
			}	
		},
		startAudio: function () {

			if(!this.audioRecorder) {
				
				navigator.mediaDevices
						.getUserMedia({ audio: true })
						.then(this.recordAudio);

			} else {

				this.audioRecorder.stop()
				this.recorder = false
				this.audioRecorder = null

			}

		},
		recordAudio: function (stream) {
			let chunks = []
			this.recorder = true
			this.suportSpeechRecognition.recognizer.start();

			this.audioRecorder = new MediaRecorder(stream);
			this.audioRecorder.start(3000)
			
			this.audioRecorder.ondataavailable = (event) => {
				chunks.push(event.data)
			}

			this.audioRecorder.onstop = () => {
				let blob = new Blob(chunks, { type : 'audio/ogg; code=opus' })
				const url = URL.createObjectURL(blob)
				document.getElementById('audioRecord').src= url;
			}
		},
		startTranscrever: function() {
			if(this.suportSpeechRecognition.start == false){
				
				try{
					this.suportSpeechRecognition.recognizer.start()
					this.suportSpeechRecognition.start = true
				}catch(ex){
					this.suportSpeechRecognition.start = false
					this.suportSpeechRecognition.recognizer.stop()
					alert("error: " + ex.message);
				}

			} else {
				this.suportSpeechRecognition.start = false
				this.suportSpeechRecognition.recognizer.stop()
			}
			
			
		}
	},
	created: function () {
		window.SpeechRecognition = window.SpeechRecognition ||
									window.webkitSpeechRecognition ||
									null;

		let recognition = {}

		if(window.SpeechRecognition === null){
			recognition.status = false
		} else {	
			recognition.status = true
			recognition.recognizer = new window.SpeechRecognition();
			recognition.recognizer.lang = "pt-BR"
			recognition.recognizer.continuous = true
			recognition.recognizer.onresult = function(event){
				let transcription =  document.getElementById("transcription");
                transcription.textContent = "";
                for (var i = event.resultIndex; i < event.results.length; i++) {
                    if(event.results[i].isFinal){
                        transcription.textContent = event.results[i][0].transcript+' (Taxa de acerto [0/1] : ' + event.results[i][0].confidence + ')';
                    }else{
                        transcription.textContent += event.results[i][0].transcript;
                    }
                }
            }
		}

		this.suportSpeechRecognition = recognition

	}
}
</script>

<style>
	nav {
		text-align: center;
	}
	.btn {
		margin: 5px;
		background-color: none;
	}
	.picture-canvas {
		display: none;
	}
	.start-audio{
		background-color: red;
	}
	.suport{
		text-align: center;
	}
</style>
