<template>
    <div>
        <div id="instructions">
        <p><strong>Thanks for helping train my face-touching app!</strong>
        To make my "don't touch your face" app work better, I need more training data of people sitting at their webcams. That's where you come in!</p>
        </p>
            <p><strong>Capturing Data</strong></p>
            <ol>
                <li>Select what type of video footage you want to capture (ideally do all three!)</li>
                <li>Get into position!</li>
                <li>Press the spacebar to start capturing video footage</li>
            </ol>
            <p><strong>Sorting and Uploading Data</strong></p>
            <p>After you've recorded video for all three states (you're touching your face, you're not touching your face, you're not in-frame at all), you'll see what photos are going to be sent to the server.</p>
            <p>Please delete the ones that are a bit ambiguous, then click "Submit Training Data" to upload those photos.</p>
            <p><a href="https://twitter.com/lazerwalker">I</a> am the only person who will be able to view these photos, and I will delete them as soon as the model has been sufficiently trained.</p>
            <p><strong>Thanks so much!!!</strong></p>
        </div>
        <div id="radioselection">
            <span :key="item+index" v-for="(item, index) in labels">
                <input type="radio" :id="item" name="sign" :value="item" 
                    :checked="selectedSign == item"
                    v-model="selectedSign">
                <label :for="item">{{item}}</label>
                &nbsp;
            </span>
            <div v-if="interval != null">Click Spacebar to Stop!</div>
        </div>
        <div id="images">
            <video @click="predict()" id="video" width="320" height="240" autoplay></video>
            <canvas id="rendered" width="224" height="224"></canvas>
            <canvas id="canvas" width="320" height="240"></canvas>
            <div id="output">
                <div id="flavor" v-if="modelmeta != null">Type: {{modelmeta.Flavor}}</div>
                <div id="exported" v-if="modelmeta != null">Exported: {{modelmeta.ExportedDate}}</div>
                <div id="current">{{guess}}</div>
                <div id="plist">
                    <ul>
                        <li :key="idx" v-for="(pitem, idx) in probabilities">{{pitem.label}}: {{pitem.probability.toFixed(2)}}%</li>
                    </ul>
                </div>
            </div>
        </div>
        <div id="listOPics" v-if="list.length > 0">
            <div>Click on an image to remove (or <button type="button" v-on:click="clearImages()">Clear All</button>)</div>
            <div>(Also maybe clean out any images that might be ambiguous if possible)</div>
            <div id="warning" v-if="list.length >= 64">64 Limit Reached - either submit or remove some images</div>
            <ul class="imagelist" :key="index" v-for="(item, index) in list">
                <li class="imgitem" @click="removeImage(index)">
                    <div>{{item.type}}</div>
                    <img height="120" width="160" :src="item.image" />
                </li>
            </ul>
            <div class="btn">
                <button type="button" v-if="list.length > 0" v-on:click="submitImages()" v-show="!processing">Submit Training Data</button>
            </div>
        </div>
        <div id="notifications" v-show="processing" v-on:click="processing=!processing"> 
            {{message}}
        </div>
    </div>
</template>

<script>
    import axios from 'axios'
    import * as cvstfjs from 'customvision-tfjs'

    export default {
        name: 'Capture',
        data: function () {
            return {
                processing: false,
                message: '',
                video: null,
                canvas: null,
                selectedSign: 'none',
                list: [],
                lastresponse: null,
                interval: null,
                model: null,
                modelmeta: null,
                labels: ['touching face', 'not touching face', 'no face present'],
                modelLabels: [],
                probabilities: [],
                guess: 'none',
                vdim: {
                    'width': 0,
                    'height': 0
                },
                appSettings: ''
            }
        },
        mounted: async function () {
            // map spacebar key event
            document.onkeyup = this.key

            // get canvas context
            let canvas = document.getElementById('canvas')
            this.canvas = canvas.getContext('2d')

            // run and start video
            this.video = document.getElementById('video')
            if(navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
                let stream = await navigator.mediaDevices.getUserMedia({ video: true })
                let tracks = stream.getVideoTracks()
                if(tracks.length >= 1) {
                    let settings = tracks[0].getSettings()
                    this.vdim.width = settings.width
                    this.vdim.height = settings.height
                    this.video.srcObject = stream
                    this.video.play()
                }
            }

            // load appSettings
            let response = await axios.get('config.json')
            this.appSettings = response.data
            console.log(this.appSettings)

            // load model
            await this.loadModel()
            
        },
        methods: {
            loadModel: async function() {
                // load model
                this.model = new cvstfjs.ClassificationModel()
                try {
                    await this.model.loadModelAsync('model/model.json')

                    // load metadata and labels
                    let manifest = await axios.get('model/cvexport.manifest')
                    this.modelmeta = manifest.data
                    let l = await axios.get('model/labels.txt')
                    this.modelLabels = l.data.split('\n').map(e => {
                        return e.trim()
                    })
                    
                    // start prediction interval
                    setInterval(this.predict, 1000)
                } catch (error) {
                    this.guess = 'model error'
                    console.log(error)
                }
            },
            predict: async function () {
                // draw video image on canvas
                var pic = document.getElementById('rendered')
                var ctx = pic.getContext('2d')
                ctx.drawImage(this.video, 0, 0, 320, 240)

                // run prediction
                const prediction = await this.model.executeAsync(pic)
                let pred = prediction[0]

                // get label and populate probabilities
                this.guess = this.modelLabels[pred.indexOf(Math.max(...pred))]
                this.probabilities = pred.map((e, i) => { 
                    return { 'label': this.modelLabels[i], 'probability': e*100 }
                })
            },
            key: function (event) {
                if(event.keyCode == 32) {
                    if(this.interval != null)
                        this.stopCapture()
                    else
                        this.startCapture()
                }
            },
            startCapture: function () {
                this.stopCapture()
                setTimeout(this.stopCapture, 60010)
                this.interval = setInterval(this.addImage, 500)
                this.video.style.border = "thick solid #FF0000"
            },
            stopCapture: function () {
                if(this.interval != null) {
                    clearInterval(this.interval);
                    this.interval = null;
                    this.video.style.border = "solid 1px gray"
                }
            },
            addImage: function () {
                if(this.list.length <  64) {
                    this.canvas.drawImage(this.video, 0, 0, 320, 240)
                    let c = document.getElementById('canvas')
                    this.list.push({
                        type: this.selectedSign,
                        image: c.toDataURL()
                    })
                } else {
                    // reached max
                    this.stopCapture()
                }
            },
            removeImage: function (index) {
                this.list.splice(index, 1)
            },
            clearImages: function () {
                this.list = []
                this.processing = false
            },
            submitImages: async function () {
                this.processing = true
                this.message = 'sending data'
                // api endpoint
                try {
                    let url = this.appSettings.saveEndpoint
                    let max_submit = this.appSettings.maxSubmitCount

                    for(let i = 0; i < this.list.length; i+=max_submit) {
                        let response = await axios.post(url, { items: this.list.slice(i, i+max_submit) }, {
                            headers: { 'Content-Type': 'application/json' }
                        })
                        this.lastresponse = response['data']
                    }
                    
                    this.message = 'done!'
                    this.list = []
                    this.processing = false
                } catch(error) {
                    // uh oh - log error and reset
                    console.log(error)
                    this.processing = false
                }
            }
        }
    }
</script>

<style scoped>
    #instructions {
        width: 640px;
        margin: 0px auto;
        text-align: left;
    }
    video {
        border: solid 1px gray;
        transform: rotateY(180deg);
        -webkit-transform:rotateY(180deg); /* Safari and Chrome */
        -moz-transform:rotateY(180deg); /* Firefox */
        float: left;
    }

    #output {
        border: solid 1px gray;
        height: 240px;
        width: 320px;
        margin-left: 10px;
        float: left;
        clear: right;
        text-align: left;
        padding: 0px 4px;
    }

    #images{
        margin: 0px auto;
        width: 700px;
        /* border: solid 10px red;*/
    }

    #rendered {
        display: none;
    }

    #canvas {
        display: none;
    }
    #warning {
        color: red;
        font-size: 16px;
        font-weight: bold;
    }
    #listOPics {
        clear: both;
        padding: 25px 100px;
        text-align: center;
        margin: 0px;
    }
    #radioselection {
        margin-bottom: 10px;
    }

    #notifications {
        width:150px;
        height:30px;
        display:table-cell;
        text-align:center;
        background:rgb(255, 166, 0);
        border:1px solid #000;
        bottom: 10px;
        right: 10px;
        position: absolute;
        padding-top: 10px;
    }

    .imagelist {
        list-style-type: none;
        padding: 0px;
    }

    .imgitem {
        float: left;
        padding: 10px;
    }

    .btn {
        text-align: center;
        clear: both;
    }
    #plist ul {
        margin: 1px;
    }
    #plist li {
        /*border: solid 1px black;*/
        margin: 5px;
    }

    #current {
        text-align: center;
        margin: 3px;
        font-size: 30px;
        color: red;
        font-weight: bolder;
    }

    #flavor {
        margin-top: 5px;
        margin-bottom: 5px;
    }

    #exported {
        margin-bottom: 5px;
    }
    #console {
        margin: 20px;
    }
</style>