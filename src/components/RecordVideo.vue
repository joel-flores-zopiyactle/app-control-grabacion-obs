<template>
    <div class="container-fluid mt-5">

        <div class="row">
            <!-- Panel de la grabación -->
            <div class="col-5">
                <!-- Lista de camaras activadas Camaras -->
                <h5>Camaras</h5>
                <div class="mb-2 d-flex flex-nowrap">
                    <button class="btn btn-sm btn-outline-primary me-1 mb-1" v-for="video in videoSourcesSelect" :key="video.deviceId" @click="changeSceneAndCamera(video)" >
                        Camara - {{ video.label }}
                    </button>
                </div>  

                <div>
                   <p> Escena actual activo: <b> {{ activeSceneCurrent }}</b> </p>
                   <p> Grabación: <b>{{tiempo}}</b> </p>
                </div>

                <!-- Video stream -->
                <video class="w-100 shadow border overflow-hidden bg-dark" autoplay muted id="video-record" width="70%" height="50%"></video>

                <!-- Control de grabación -->
                <div class="mt-2 w-100 d-flex justify-content-center">
                    <button type="button" class="btn btn-outline-dark me-2" @click="startRecord">
                        Play
                    </button>

                    <button type="button" class="btn btn-outline-dark me-2" @click="pauseRecord">
                        Pause
                    </button>

                    <button type="button" class="btn btn-outline-dark me-2" @click="resumenRecord">
                        Resumen
                    </button>

                    <button type="button" class="btn btn-outline-dark" @click="stopRecord">
                        Stop
                    </button>
                </div>
            </div>

            <!-- Mas configuraciones -->
            <div class="col-6">

            </div>
        </div>
    </div>
</template>

<script>
// OBS 
const OBSWebSocket = require('obs-websocket-js');
const obs = new OBSWebSocket();
const obs2 = new OBSWebSocket(); // Hace una conexion a una maquina externa mediante una direccion IP

export default {
    name: 'RecordVideo',
    data() {
        return {
            video: [], // video del stream de la WebCam
            scenes: [], // Escenas disponibles en OBS
            videoSourcesSelect: [],
            videoSourceId: '', // Id del dispositivo web
            audioSourcesSelect: [],
            activeSceneCurrent: 'HD60-S', // Muestra el Escena actual activo en OBS
            expedienteID: 0,
            durationVideo: '', // Duracion del video grabado desde OBS
            ubicationVideo: '', // Ubicaccion del video gurdado desde OBS
            // Crnometro
            tiempoRef : Date.now(),
            cronometrar : false,
            acumulado : 0,
            tiempo: '00:00:00.000',
        }
    },

    created() {
        this.connectOBS()
        this.connectOBSExterno()
    },

    methods: {

        // Obtener video de la WebCam
        async startVideoWebCam() {
            this.video =  document.getElementById('video-record')
            
            const constraints = {
                // {
                //     deviceId: audioSource ? {exact: audioSource} : undefined
                // }
                audio: true,
                video: {
                    deviceId: this.videoSourceId ? {exact: this.videoSourceId} : undefined
                }
            }

            navigator.getMedia = (  navigator.getUserMedia         ||
                                    navigator.webkitGetUserMedia   ||
                                    navigator.mozGetUserMedia      ||
                                    navigator.msGetUserMedia);

            try {

                const stream = await navigator.mediaDevices.getUserMedia(constraints)
                this.video.srcObject = stream

            } catch (error) {
                console.log(error);
            }
        },

        connectOBS() {

            obs.connect({ address: 'localhost:4444', password: ''}).then(() => {
                // console.log(`Success! We're connected & authenticated.`);
                return obs.send('GetSceneList');

            }).then(data => {
                // console.log(data.scenes);
                this.scenes = data.scenes
            })
            .catch(err => { // Promise convention dicates you have a catch on every chain.
                console.log(err);
                if(err.code === 'CONNECTION_ERROR') {
                    alert('OBS - No esta activado, pára empezar a grabar hay que activar OBS Studio!.')
                }
            });

           
        },

        connectOBSExterno() {
            // Si la direccion es cambiada hay que actualizar por la nueva direccion de la maquina externa
            obs2.connect({ address: '192.168.0.105:4445', password: ''}).then(() => { // TODO: Conexion por direccion IP
                // console.log(`Success! We're connected & authenticated.`);
                return obs.send('GetSceneList');

            }).then(data => { 
                console.log(data.scenes);
                //this.scenes = data.scenes
            })
            .catch(err => { // Promise convention dicates you have a catch on every chain.
                console.log(err);
                if(err.code === 'CONNECTION_ERROR') {
                    alert('OBS extreno - No esta activado.')
                }
            });
        },

        changeSceneHD60_S() {
            obs.send('SetCurrentScene', {'scene-name': 'HD60-S'}).then( data => {
                console.log(data);
            }).catch( err => console.log(err)) 
        },

        changeSceneHD60_Pro() {
            obs.send('SetCurrentScene', {'scene-name': 'HD60-PRO'}).then( data => {
                console.log(data);
            }).catch( err => console.log(err)) 
        },

        changeSceneAndCamera(video) {
            // console.log(this.videoSourceId);
            switch (video.label) {
                case 'HP Truevision HD (0bda:5776)': // TODO: cambiar por el bnombre de la capturadora
                    this.videoSourceId = video.deviceId
                     this.startVideoWebCam()
                    this.changeSceneHD60_S()
                    break;

                case 'GENERAL WEBCAM (1b3f:2247)': // TODO: cambiar por el bnombre de la capturadora
                    this.videoSourceId = video.deviceId
                    this.startVideoWebCam()
                    this.changeSceneHD60_Pro()
                    break;

                case 'OBS Virtual Camera':
                    this.videoSourceId = video.deviceId
                    this.startVideoWebCam()
                    // this.changeSceneHD60_S()
                    break;
            
                default:
                    // this.changeSceneHD60_S()
                    break;
            }
        },

        listMediaDevices() {
            if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
                console.log("enumerateDevices() not supported.");
                return;
            }

            navigator.mediaDevices.enumerateDevices().then((devices) => {
                // Iterar sobre toda la lista de dispositivos (InputDeviceInfo y MediaDeviceInfo)
                devices.forEach((device) => {
                    // Según el tipo de dispositivo multimedia
                    switch(device.kind){
                        // Agregar dispositivo a la lista de cámaras
                        case "videoinput":
                            this.videoSourcesSelect.push(device)
                            break;
                        // Agregar dispositivo a la lista de micrófonos
                        case "audioinput":
                            this.audioSourcesSelect.push(device)
                            break;
                    }
                });
            }).catch(function (e) {
                console.log(e.name + ": " + e.message);
            });
        },

        async startRecord() {
    
            try {

                if(!confirm('¿Esta seguro de empezara a grabar?')) return
                // Se manda el comando para empezar a grabar
                await obs.send('StartRecording')
                await obs2.send('StartRecording')
                this.video.play()   
                this.cronometrar = true
                 

            } catch (error) {

                if(error.status === 'error') {
                        alert('¿OBS no esta activado?. Para grabar hay que conectarse a OBS...')
                }
            }       
        },

        pauseRecord() {
            if(!confirm('¿Estas seguro de pausar la grabación?')) return
            obs.send('PauseRecording').catch(error => console.log(error))
            obs2.send('PauseRecording').catch(error => console.log(error))
            this.video.pause()
        },
        
        resumenRecord() {
            if(!confirm('¿Estas seguro de seguir grabando?')) return
            obs.send('ResumeRecording').catch(error => console.log(error))
            obs2.send('ResumeRecording').catch(error => console.log(error))
            this.video.play()
        },

        stopRecord() {
            if(!confirm('¿Estas seguro de finalizar la grabación?\nUna vez finalizada la grabación ya no podra grabar de nuevo.')) return
            obs.send('StopRecording')
            obs2.send('StopRecording')
            .catch(error => {
                console.log(error);
            })
            //Permite asignar el nombre del archivo
            obs.send('SetFilenameFormatting', {
                'filename-formatting': `audiencia_numero_${this.expedienteID}`
            }).catch(error => console.log(error))
            // OBS 2
            obs2.send('SetFilenameFormatting', {
                'filename-formatting': `audiencia_numero_${this.expedienteID}`
            }).catch(error => console.log(error))
            this.video.pause();   
            this.cronometrar = false
        },

        cronometro() {
            setInterval(() => {
                //let tiempo = document.getElementById("tiempo")
                if (this.cronometrar) {
                    this.acumulado += Date.now() - this.tiempoRef
                }
            
                this.tiempoRef = Date.now()
                this.tiempo = formatearMS(this.acumulado)
            }, 1000 / 60);

            function formatearMS(tiempo_ms) {
                let MS = tiempo_ms % 1000
                
                //Agregué la variable St para solucionar el problema de contar los minutos y horas.
                
                let St = Math.floor(((tiempo_ms - MS) / 1000))
                
                let S = St%60
                let M = Math.floor((St / 60) % 60)
                let H = Math.floor((St/60 / 60))
                Number.prototype.ceros = function (n) {
                    return (this + "").padStart(n, 0)
                }
                return H.ceros(2) + ":" + M.ceros(2) + ":" + S.ceros(2) + "." + MS.ceros(3)
            }
        }
    },

    mounted() {
        this.startVideoWebCam()
        this.listMediaDevices()
        this.cronometro()
        // Obtner escena actual activo
        obs.on('SwitchScenes', data => { // Espera en cambio de escnea
        console.log(data);
            this.activeSceneCurrent = data.sceneName
        });

        obs.on('RecordingStopping', data => {
               this.durationVideo   = data.recTimecode;
               this.ubicationVideo  = data.recordingFilename
        });
    }
}
</script>
