# ReactJS use WASM created with C & Emscripten


# Motivation


# Installation
## Software Prep on Mac M1 Sonoma

### Build llvm
https://github.com/llvm/llvm-project/tags - download rc2 #17
emscripten cribs that it needs v17 of LLVM, direct llvm clone gives version 18, mac default has ver 16
```
cd llvm-project-llvmorg-17.0.0-rc2
mkdir build
cd build
cmake -DLLVM_ENABLE_PROJECTS=clang -DLLVM_ENABLE_PROJECTS=lld -DCMAKE_BUILD_TYPE=Release -G "Unix Makefiles" ../llvm
make
sudo make install
```

### Build Binareyen
```
git clone https://github.com/WebAssembly/binaryen
git submodule init
git submodule update
cmake . && make
```
### Install Emscripten
```
apt-get install python3
apt-get install wget
apt-get install xz-utils
apt-get install libzip2
apt-get install bzip2
apt-get install python3
apt-get install git
apt-get install binutils
apt-get install make
git clone https://github.com/emscripten-core/emsdk.git
cd ./emsdk
git pull
./emsdk install latest
./emsdk activate latest
source ./emsdk_env.sh
```

### Change Emscripten variables
```
./emcc --generate-config
vim emsdk/upstream/emscripten/.emscripten
#change LLVM_ROOT and BINAREYEN_ROOT
LLVM_ROOT = '/Users/amythical/AD/mycode/wasm/llvm-project-llvmorg-17.0.0-rc2/build/bin' # directory
BINARYEN_ROOT = '/usr/local' # directory
```

### Modify path in ~.zshrc
```
export PATH="/Users/amythical/AD/mycode/wasm/emsdk:/Users/amythical/AD/mycode/wasm/emsdk/upstream/emscripten:/usr/local/bin:$PATH"
source "/Users/amythical/AD/mycode/wasm/emsdk/emsdk_env.sh"
```

### X264 installation and compile with emscripten
```
wget https://download.videolan.org/pub/videolan/x264/snapshots/x264-snapshot-20170226-2245-stable.tar.bz2
tar xvfj x264-snapshot-20170226-2245-stable.tar.bz2
cd x264-snapshot-20170226-2245-stable
 emconfigure ./configure \
  --prefix=/opt/ffmpeg \
  --host=i686-gnu \
  --enable-static \
  --disable-cli \
  --disable-asm \
  --extra-cflags="-s USE_PTHREADS=1"
emmake make
sudo emmake make install
####### The paths where includes/libs/binaries are installed by make install for x264 
# install -d /opt/ffmpeg/include
# install -d /opt/ffmpeg/lib
# install -d /opt/ffmpeg/lib/pkgconfig
# install -m 644 ./x264.h /opt/ffmpeg/include
# install -m 644 x264_config.h /opt/ffmpeg/include
# install -m 644 x264.pc /opt/ffmpeg/lib/pkgconfig
# install -m 644 libx264.a /opt/ffmpeg/lib
# /Users/amythical/AD/mycode/wasm/emsdk/upstream/emscripten/emranlib /opt/ffmpeg/lib/libx264.a
#######

#for pkg-config to find x264 - a x264.pc is in /opt/ffmpeg/lib/pkgconfig - run pkg-config --variable pc_path pkg-config and find the directories pkg-config looks into, copy x264.pc to one of the directories
cp /opt/ffmpeg/lib/pkgconfig/x264.pc /opt/homebrew/share/pkgconfig 
```


### FFmpeg install and compile with Emscripten

1. Wget ffmpeg gz
```
wget http://ffmpeg.org/releases/ffmpeg-${FFMPEG_VERSION}.tar.gz && \
tar zxf ffmpeg-${FFMPEG_VERSION}.tar.gz && rm ffmpeg-${FFMPEG_VERSION}.tar.gz
```

2. Configure ffmpeg
```
#ffmpeg configure script cant find the C compiler LLVM we compiled on MAC OS Venture/Sonoma, so modify it 
#cc value on LLVM compiled box is llvm-as
#change 1
PKG_CONFIG_LIBDIR="/opt/ffmpeg/lib/pkgconfig"
export PKG_CONFIG_LIBDIR

#change 2
probe_cc(){ 
    pfx=$1
    _cc=$2
    first=$3

    unset _type _ident _cc_c _cc_e _cc_o _flags _cflags
    unset _ld_o _ldflags _ld_lib _ld_path
    unset _depflags _DEPCMD _DEPFLAGS
    _flags_filter=echo

    if $_cc --version 2>&1 | grep -q '^GNU assembler'; then
        true # no-op to avoid reading stdin in following checks
    elif $_cc --version 2>&1 | grep "LLVM (http\:\/\/llvm\.org\/)"; then #ffmpeg/llvm/mac
        true # no-op to avoid reading stdin in following checks
 ```
 
* Build ffmpeg
```
##build-ffmpeg.sh
FFMPEG_VERSION=6.0
PREFIX=/opt/ffmpeg
CFLAGS="-s USE_PTHREADS=1 -O3 -I${PREFIX}/include:${PREFIX}/lib/pkgconfig:/Users/amythical/AD/mycode/wasm/llvm-project-llvmorg-17.0.0-rc2/build/lib/clang/17/include"
LDFLAGS="$CFLAGS -L${PREFIX}/lib -s INITIAL_MEMORY=33554432"
MAKEFLAGS="-j4"
cd ffmpeg-${FFMPEG_VERSION}
  emconfigure ./configure \
  --prefix=${PREFIX} \
  --target-os=none \
  --arch=x86_32 \
  --enable-cross-compile \
  --disable-debug \
  --disable-x86asm \
  --disable-inline-asm \
  --disable-stripping \
  --disable-programs \
  --disable-doc \
  --disable-all \
  --enable-avcodec \
  --enable-avformat \
  --enable-avfilter \
  --enable-avdevice \
  --enable-avutil \
  --enable-swresample \
  --enable-postproc \
  --enable-swscale \
  --enable-filters \
  --enable-protocol=file \
  --enable-decoder=h264,aac,pcm_s16le \
  --enable-demuxer=mov,matroska \
  --enable-muxer=mp4 \
  --enable-gpl \
 --enable-libx264 \
  --extra-cflags="$CFLAGS" \
  --extra-cxxflags="$CFLAGS" \
  --extra-ldflags="$LDFLAGS" \
  --nm="llvm-nm -g" \
  --ar=emar \
  --as=llvm-as \
  --ranlib=llvm-ranlib \
  --cc=emcc \
  --cxx=em++ \
  --objcc=emcc \
  --dep-cc=emcc
echo "configure done ... make starting"
emmake make -j4
echo "make done ... make install starting"
emmake make install
echo "all done ..."
##############
```

# Code pipeline
```
ReactJS ->> Helper: await Helper.load()
Helper ->> Worker: initialise worker and send init message
Worker->> Load JS + WASM
ReactJS ->> const val1 =  await Helper.fun1();
Helper ->> sends a message to Worker
Worker ->> executes function in wasm 
Worker ->> sends the return value via a message to Helper
Helper -->> returns the value from Worker to ReactJS using a Promise
```

# Hello world 
We start with a basic 'hello world' program called motd or message of the day, familiar to Linux users.

## React JS
```
# Javascript code
const  tCCVideoToolkitHelper = new  CCVideoToolkitHelper();
await  tCCVideoToolkitHelper.load();
const  tTrimmedVideoData = await  tCCVideoToolkitHelper.trimVideo(pTrimOptions);
```
## Helper
```
// old

// import WebWorkerEnabler from '../../utils/webWorkerEnabler';

// import { Buffer } from 'buffer';

// import ffmpegWorker from '../WebWorkers/ffmpeg-all-codecs';

// import WebworkerClient from '../FFMPEGWorker/FFMPEGWebWorkerClient'; // https://www.npmjs.com/package/ffmpeg-webworker-fix

import  options  from  '../../../fontConfig';

import  *  as  CanvasUtils  from  '../../../utils/canvas/canvasutils';

// const fs = require('fs');

// const { createFFmpeg, fetchFile } = require('@ffmpeg/ffmpeg');

/*

import * as TimeHelper from '../../utils/timeHelper';

import * as MediaFileHelper from '../../utils/MediaFileHelper';

import * as MediaInfoHelper from '../../utils/MediaInfoHelper';

import * as CCCache from '../../utils/cccache';

*/

import  WebWorkerEnabler  from  '../../../utils/webWorkerEnabler';

import  CCVideoToolkitWorker  from  '../utils/CCVideoToolkitWorker';

  

export  default  class  CCVideoToolkitHelper {

constructor(pOptions = {}) {

this.loadingStatus = 0; // 0 - Not loaded, 1 - loading, 2 loaded

this.debug = true;//false;//pOptions.debug ? pOptions.debug : false;

  

this.worker = null;

this.workerResolver = null;

  

this.receiveMessageFromWorker = this.receiveMessageFromWorker.bind(this);

this.load = this.load.bind(this);

  

// Create Worker

this.worker = new  WebWorkerEnabler(

CCVideoToolkitWorker,

).getWorker();

this.worker.onmessage = this.receiveMessageFromWorker;

}

  

receiveMessageFromWorker(pMessage) {

if (pMessage.data.cmd === 'workerloaded') {

this.workerResolver();

}

else  if (pMessage.data.cmd === 'trimmedvideo') {

this.workerResolver(pMessage.data.data);

}

}

  

createResolver(){

let  tPromise = new  Promise(r=>this.workerResolver=r);

return  tPromise;

}

  

load(){

const  tPromise = this.createResolver();

this.worker.postMessage({"cmd":"init","data":{}},[]);

return  tPromise;

}

  

trimVideo(pOptions){

const  tPromise = this.createResolver();

this.worker.postMessage({"cmd":"trimvideo",data:pOptions},[]);

return  tPromise;

}

  

async  extractAudioForVideo(pOptions) {

  

return  null;

}

  
  

}
```
## Worker
```
export  default  function  CCVideoToolkitWorker() {

const  JS_DOMAIN = `http://localhost:3002`;

  

let  myModule = null;

onmessage = e  => {

const  tCmd = e.data.cmd;

const  tPayloadJSON = e.data.data;

  

if (tCmd === 'init') {

self.importScripts(

`${JS_DOMAIN}/video/ccvideotoolkit/trim.js`,

);

new  createModule().then(pMyModule  => {

myModule = pMyModule;

self.postMessage({

cmd:  'workerloaded',

data: {},

});

});

  
  

}

else  if (tCmd === 'trimvideo') {

trimVideo(tPayloadJSON).then((pReturnJSON) => {

self.postMessage({

cmd:  'trimmedvideo',

data: { ur:  null, buffer:  pReturnJSON.arrayBuffer.buffer, outputFormat:  pReturnJSON.outputFormat },

}, [pReturnJSON.arrayBuffer.buffer]);

});

}

};

  

function  trimVideo(pOptions) {

return  new  Promise((resolve, reject) => {

try {

const  tVideoUrl = pOptions.videoUrl || null;

let  tVideoUintData = null;

if (tVideoUrl) {

// tVideoUintData = await fetchFile(tVideoUrl); // Doesnt work for 2gb+ files

customfetchFile(tVideoUrl).then((pVideoUintData) => {

// Call module to allocate buffer and return a pointer

// myModule._motd();

// Write UINTData to pointer location

const  tUint8clampedData = new  Uint8ClampedArray(pVideoUintData);

const  tVideoDataBufferPtr = myModule._createVideoDataBuffer(tUint8clampedData.length);

myModule.writeArrayToMemory(tUint8clampedData, tVideoDataBufferPtr);

// Call WASM Function

  

const  tTrimInfoPtr = myModule._trimVideo();

// TrimInfo has bytes trimmed and pointer to mem location of trimmed data

const  tTrimFileSizeBytes = myModule.getValue(tTrimInfoPtr,'i32');

console.log("tTrimFileSizeBytes "+tTrimFileSizeBytes);

const  tTrimFileBufferPtr = myModule.getValue(tTrimInfoPtr + 4, '*');

const  tTrimmedVideoData = new  Uint8ClampedArray(

myModule.HEAPU8.subarray(tTrimFileBufferPtr, tTrimFileBufferPtr + tTrimFileSizeBytes),

);

// clone to free C memory and return the trimmed data to the React component

const  tVideoDataBufferClone = structuredClone(tTrimmedVideoData);

  

myModule._freeVideoDataBuffer();

return  resolve({

url:  null, // trimmedVideoUrl, - url isnt used anywhere

arrayBuffer:  tVideoDataBufferClone,// tMp4Blob,

outputFormat:  `mp4`,

});

})// Doesnt work for 2gb+ files

} else {

console.log('no vdeo url');

return  resolve({ url:  null, blob:  null });

}

  

} catch (ex) {

console.log("error in ccvideotoolkitworker:54 " + ex);

return  resolve({ url:  null, buffer:  null });

}

});

}

  

// https://dmitripavlutin.com/javascript-fetch-async-await/

function  customfetchFile(pFile) {

return  new  Promise((resolve, reject) => {

// console.log('FFmpeghelper2:2534 customfetch called');

  

if (pFile.indexOf('blob:')) {

; // console.log('CUSTOMFETCHFILE - BLOB TYPE');

}

// console.log('FFEMPEGHELEPR2 custom fetch starting');

const  controller = new  AbortController();

let  tTimeout = -1;

// alert(pFile.indexOf('blob:'));

if (pFile.indexOf('blob:') > -1) {

// alert("set timer");

tTimeout = setTimeout(() => {

controller.abort();

const  message = `CustomFetchFile 0x0 An error has occured fetch timeout`;

return  reject(new  Error(message));

}, 1000 * 30); // 30 sec timeout

}

const  res = null;

  

fetch(pFile, controller.signal)

.then(pRes  => {

if (!pRes.ok) {

if (tTimeout) clearTimeout(tTimeout);

const  message = `CustomfetchFile 0x1 An error has occured status: ${pRes.status

}`;

return  reject(new  Error(message));

// throw new Error(message);

}

  

if (tTimeout) {

// console.log('clear timeout');

clearTimeout(tTimeout);

}

  

tTimeout = setTimeout(() => {

// console.log('arraybuffer timeout');

const  message = `customFetchFile: 0x2 An error has occured: creatingArrayBuffer timedout`;

return  reject(new  Error(message));

}, 1000 * 30);

  

pRes.arrayBuffer().then(pData  => {

if (tTimeout) {

clearTimeout(tTimeout);

}

const  tUintData = new  Uint8Array(pData);

return  resolve(tUintData);

});

})

.catch(ex2  => {

// console.log(`catch ${ex2}`);

if (tTimeout) clearTimeout(tTimeout);

const  message = `customFetchFile: 0x3 An error has occured 3: ${JSON.stringify(

ex2,

)}`;

return  reject(new  Error(message));

// throw new Error(message);

});

});

}

} // end main function
```
## C
```
#include <stdio.h>

#include <emscripten.h>

  

EMSCRIPTEN_KEEPALIVE

int motd() {

printf("MOTD: Chuck Norris doesn't need a watch, he decides what time it is \n");

fflush(stdin);

return 1;

}

```
## Compile with emscripten
```
emcc motd.c -o motd.js -s WASM=1 -s ENVIRONMENT=web  -s MODULARIZE=1 -s EXPORTED_FUNCTIONS="['motd']" -s MODULARIZE=1 -s WASM=1 -s USE_ES6_IMPORT_META=0  -s "EXPORT_NAME='createModule'"

  

produces motd.js and motd.wasm
```
## Deploying in express
* Copy motd.js to the public/video/mywasm/ directory
* Copy motd.wasm to public/video/mywasm/ directory
* Edit motd.js, find wasmBinary and change the path of motd.wasm to http://localhost:3000/video/mywasm/motd.wasm
// This path should change to any webserver / CDN path that you need.

## Running motd using wasm
When we run the React url, in the developer console we should see a nice little message from Chuck Norris
```
MOTD: Chuck Norris doesn't need a watch, he decides what time it is
```
## Pass UINT Array to C from JS

## Wasm call libav to get Video info

## Return a structure from C to JS

## Wasm Trim a video using libav
```
#include <emscripten.h>
#include <stdio.h>
#include <stdarg.h>
#include <stdlib.h>
#include <string.h>
#include <inttypes.h>
#include <libavformat/avformat.h>
#include <libavutil/avutil.h>
#include <libavformat/avio.h>
#include <libavcodec/packet.h>
#include <libavutil/log.h>
#include <libavutil/timestamp.h>
#include <libavutil/rational.h>

uint8_t *buffer1Ptr = NULL;
int buffer1Size = 0;
uint8_t *buffer2Ptr = NULL;
int buffer2Size = 0;

typedef struct trimInfo {
	int bufferSize;
	uint8_t *buffer;
} trimInfo;

trimInfo *trimInfoPtr;
static void logging(char *pMsg)
{
	printf("%s\n",pMsg);
	// av_log(NULL, AV_LOG_DEBUG, "%s\n", pMsg);
}

uint8_t *EMSCRIPTEN_KEEPALIVE createVideoDataBuffer(int pBufferSize)
{
	printf("In C createVideoDataBuffer ... allocating space\n");
	buffer1Size = pBufferSize;
	buffer1Ptr = malloc(pBufferSize * sizeof(uint8_t));
	return buffer1Ptr;
}

void EMSCRIPTEN_KEEPALIVE freeVideoDataBuffer()
{
	printf("In C destroyBufferImage ... freeing space\n");
	if(buffer1Ptr)
		free(buffer1Ptr);
	if(buffer2Ptr)
		free(buffer2Ptr);
}

trimInfo* EMSCRIPTEN_KEEPALIVE trimVideo()
{
	float tStartSeconds = 1.0;
	float tEndSeconds = 9.0;
	int tOperationResult;
	AVIOContext *tAvInputIOContext = NULL;
	AVIOContext *tAvOutputIOContext = NULL;
	AVFormatContext *tAvInputFormatContext = NULL;
	AVFormatContext *tAvOutputFormatContext = NULL;
	AVPacket *tAvPacket = NULL;
	int tWriteFlag = 0;

	tAvInputIOContext = avio_alloc_context(buffer1Ptr, 	buffer1Size, // internal Buffer and its size
tWriteFlag,  // bWriteable (1=true,0=false)
NULL,  // user data ; will be passed to our callback functions
NULL,  // read function
NULL,  // Write callback function (not used in this example)
NULL); //seek function

    tAvInputFormatContext = avformat_alloc_context();
    tAvInputFormatContext->pb = tAvInputIOContext;
    tOperationResult = avformat_open_input(&tAvInputFormatContext, "", 0, 0);
    if (tOperationResult < 0)
    {
        logging("Failed to open the input file");
        return NULL;
    }
    // we dont get duration without this

    tOperationResult = avformat_find_stream_info(tAvInputFormatContext, 0);

    if (tOperationResult < 0)
    {
        logging("Failed to retrieve the input stream information.");
    }

    avio_open_dyn_buf(&tAvOutputIOContext);
    avformat_alloc_output_context2(&tAvOutputFormatContext, NULL, "mp4", NULL);
    tAvOutputFormatContext->pb = tAvOutputIOContext;

    if (!tAvOutputFormatContext)
    {
        tOperationResult = AVERROR_UNKNOWN;
        logging("Failed to create the output context.");
    }

    int tStreamIndex = 0;
    int tStreamMapping[tAvInputFormatContext->nb_streams];
    int tStreamRescaledStartSeconds[tAvInputFormatContext->nb_streams];
    int tStreamRescaledEndSeconds[tAvInputFormatContext->nb_streams];

    // Copy streams from the input file to the output file.
    for (int i = 0; i < tAvInputFormatContext->nb_streams; i++)
    {
        AVStream *tOutStream;
        AVStream *tInStream = tAvInputFormatContext->streams[i];

    // Calculate scaled start time and end time
    tStreamRescaledStartSeconds[i] = av_rescale_q(tStartSeconds * AV_TIME_BASE, AV_TIME_BASE_Q, tInStream->time_base);
    tStreamRescaledEndSeconds[i] = av_rescale_q(tEndSeconds * AV_TIME_BASE, AV_TIME_BASE_Q, tInStream->time_base);

    if (tInStream->codecpar->codec_type != AVMEDIA_TYPE_AUDIO &&
    tInStream->codecpar->codec_type != AVMEDIA_TYPE_VIDEO &&
    tInStream->codecpar->codec_type != AVMEDIA_TYPE_SUBTITLE)
    {
    tStreamMapping[i] = -1;
    continue;
    }

    

    tStreamMapping[i] = tStreamIndex++;

    

    tOutStream = avformat_new_stream(tAvOutputFormatContext, NULL);

    if (!tOutStream)

    {

    tOperationResult = AVERROR_UNKNOWN;

    logging("Failed to allocate the output stream.");

    }

    

    tOperationResult = avcodec_parameters_copy(tOutStream->codecpar, tInStream->codecpar);

    if (tOperationResult < 0)

    {

    logging("Failed to copy codec parameters from input stream to output stream.");

    }

    tOutStream->codecpar->codec_tag = 0;

    } // for

    

    tOperationResult = avformat_write_header(tAvOutputFormatContext, NULL);

    if (tOperationResult < 0)

    {

    logging("Error occurred when opening output file.");

    }
    tOperationResult = avformat_seek_file(tAvInputFormatContext, -1, INT64_MIN, tStartSeconds * AV_TIME_BASE,

    tStartSeconds * AV_TIME_BASE, 0);

    if (tOperationResult < 0)

    {

    logging("Failed to seek the input file to the targeted start position.");

    }

  

  

    tAvPacket = av_packet_alloc();

    if (!tAvPacket)

    {

    logging("Failed to allocate AVPacket.");

    return NULL;

    }
    while (1)
    {

        tOperationResult = av_read_frame(tAvInputFormatContext, tAvPacket);

        if (tOperationResult < 0)
            break;

        // Skip packets from unknown streams and packets after the end cut position.
        if (tAvPacket->stream_index > tAvInputFormatContext->nb_streams || tStreamMapping[tAvPacket->stream_index] < 0)
        {
            av_packet_unref(tAvPacket);
            continue;
        }

        if (tAvPacket->pts > tStreamRescaledEndSeconds[tAvPacket->stream_index])
        {
            av_packet_unref(tAvPacket);
            break;
        }

        tAvPacket->stream_index = tStreamMapping[tAvPacket->stream_index];
        // logPacket(tAvInputFormatContext, tAvPacket, "in");

        // Shift the packet to its new position by subtracting the rescaled start seconds.
        tAvPacket->pts -= tStreamRescaledStartSeconds[tAvPacket->stream_index];
        tAvPacket->dts -= tStreamRescaledStartSeconds[tAvPacket->stream_index];
        av_packet_rescale_ts(tAvPacket, tAvInputFormatContext->streams[tAvPacket->stream_index]->time_base,
        tAvOutputFormatContext->streams[tAvPacket->stream_index]->time_base);
        tAvPacket->pos = -1;
        // logPacket(tAvOutputFormatContext, tAvPacket, "out");

        tOperationResult = av_interleaved_write_frame(tAvOutputFormatContext, tAvPacket);
        if (tOperationResult < 0)
        {
            logging("Failed to mux the packet.");
        }
    } // while
	av_write_trailer(tAvOutputFormatContext);
	av_packet_free(&tAvPacket);
	avformat_close_input(&tAvInputFormatContext);
	  
	buffer2Size = avio_close_dyn_buf(tAvOutputIOContext,&buffer2Ptr);

	// printf("dyn close returned %d\n",buffer2Size);
	trimInfoPtr = malloc(1 * sizeof(trimInfo));
	trimInfoPtr->bufferSize = buffer2Size;
	trimInfoPtr->buffer = buffer2Ptr;

	/*
// causes a crash as we havent allocated this using avio_io_open and instead used dyn_buf
if (tAvOutputFormatContext && !(tAvOutputFormatContext->oformat->flags & AVFMT_NOFILE)){
		avio_closep(&tAvOutputFormatContext->pb);
	}*/

	avformat_free_context(tAvOutputFormatContext);
	return trimInfoPtr;
}
```

## Make file
```
trim.js: trim.c

emcc -I/opt/ffmpeg/include -L/opt/ffmpeg/lib/ -lavcodec -lavformat -lavfilter -lavdevice -lswresample -lpostproc -lswscale -lavutil -lm $^ -o $@ -s WASM=1 -s ENVIRONMENT=web  -s MODULARIZE=1  -s MODULARIZE=1 -s WASM=1 -s USE_ES6_IMPORT_META=0 -s "EXPORTED_RUNTIME_METHODS='writeArrayToMemory','getValue'"  -s "EXPORT_NAME='createModule'" -s ALLOW_MEMORY_GROWTH  -s TOTAL_MEMORY=65536000
```
For lager files we need the -s ALLOW_MEMORY_GROWTH option or we get a runtime memory error
After updating to ffmpeg-7.0, got a *initial memory too small* error, to fix this assign 1Gb (in multiples of 64KB i.e. 64*500*2*1024 as TOTAL_MEMORY)

## Read UINT Array (Returned from C) in JS
```
/* fetch(videoUrl) returns an ArrayBuffer - convert it to a UintArray and assigned to  pVideoUintData */

const  tUint8clampedData = new  Uint8ClampedArray(pVideoUintData);
const  tVideoDataBufferPtr = myModule._createVideoDataBuffer(tUint8clampedData.length);
myModule.writeArrayToMemory(tUint8clampedData, tVideoDataBufferPtr);
// Call WASM Function
const  tTrimInfoPtr = myModule._trimVideo();
// TrimInfo has bytes trimmed and pointer to mem location of trimmed data
const  tTrimFileSizeBytes = myModule.getValue(tTrimInfoPtr,'i32');
console.log("tTrimFileSizeBytes "+tTrimFileSizeBytes);
const  tTrimFileBufferPtr = myModule.getValue(tTrimInfoPtr + 4, '*');
const  tTrimmedVideoData = new  Uint8ClampedArray(
myModule.HEAPU8.subarray(tTrimFileBufferPtr, tTrimFileBufferPtr + tTrimFileSizeBytes),
);

// clone to free C memory and return the trimmed data to the React component
const  tVideoDataBufferClone = structuredClone(tTrimmedVideoData);

myModule._freeVideoDataBuffer();
// return tVideoDataBufferClone;
```

https://www-nxrte-com.translate.goog/jishu/44499.html?_x_tr_sl=zh-CN&_x_tr_tl=en&_x_tr_hl=en&_x_tr_pto=sc

https://javascript.plainenglish.io/slimming-down-ffmpeg-for-a-web-app-compiling-a-custom-version-20a06d36ece1

https://zhuanlan.zhihu.com/p/74287959

https://jeromewu.github.io/build-ffmpeg-webassembly-version-part-4-v0.2/

https://javascript.plainenglish.io/slimming-down-ffmpeg-for-a-web-app-compiling-a-custom-version-20a06d36ece1


Problems
------------
## Frame_size was not respected for a non-last frame"
FFmpeg error for audio transcoding, "frame_size (%d) was not respected for a non-last frame"
Fix - check for samples read less than decoders frame size, do not send samples to encoder until this number is reached.
Ideally add samples to a buffer and send them when they are equal to the encoders frame size.
```
In audioEncoder()
 if(input_frame && input_frame->nb_samples < outputAudioCodecContext->frame_size){
	return -1;
}// dont send samples to the encoder
```
## Memory not enough 
	fix  - add emcc option in Makefile for js/wasm `TOTAL_MEMORY=1000MB`
## Cannot find libx264 while encoding in browser
	
Recompile ffmpeg 

```
FFMPEG_VERSION=7.0
PREFIX=/opt/ffmpeg
#CFLAGS="-s USE_PTHREADS=1 -O3 -I${PREFIX}/include:${PREFIX}/lib/pkgconfig:/Users/amythical/AD/mycode/wasm/llvm-project-llvmorg-17.0.0-rc2/build/lib/clang/17/include"
CFLAGS="-s USE_PTHREADS=1 -O3 -I${PREFIX}/include"
LDFLAGS="$CFLAGS -L${PREFIX}/lib -s INITIAL_MEMORY=33554432"
MAKEFLAGS="-j4"
cd ffmpeg-${FFMPEG_VERSION}
  emconfigure ./configure \
  --prefix=${PREFIX} \
  --target-os=none \
  --arch=x86_32 \
  --enable-cross-compile \
  --disable-debug \
  --disable-x86asm \
  --disable-inline-asm \
  --disable-stripping \
  --disable-programs \
  --disable-doc \
  --disable-all \
  --enable-avcodec \
  --enable-avformat \
  --enable-avfilter \
  --enable-avdevice \
  --enable-avutil \
  --enable-swresample \
  --enable-postproc \
  --enable-swscale \
  --enable-filters \
  --enable-protocol=file \
  --enable-decoder=h264,aac,pcm_s16le \
  --enable-demuxer=mov,matroska \
  --enable-muxer=mp4 \
  --enable-encoder=libx264,aac \
  --enable-gpl \
 --enable-libx264 \
  --extra-cflags="$CFLAGS" \
  --extra-cxxflags="$CFLAGS" \
  --extra-ldflags="$LDFLAGS" \
--nm="/Users/amythical/AD/mycode/wasm/emsdk/upstream/bin/llvm-nm -g" \
  --ar=emar \
  --as=llvm-as \
  --ranlib=llvm-ranlib \
  --cc=emcc \
  --cxx=em++ \
  --objcc=emcc \
  --dep-cc=emcc
echo "configure done ... make starting"
emmake make -j4
echo "make done ... make install starting"
emmake make install
echo "all done ..."
```

**what was missing ? **
```
 --enable-encoder=libx264,aac 
 ```
*Found this from [ffmpeg.js build](https://github.com/BrianJFeldman/ffmpeg.js.wasm/blob/master/Makefile#L289)
Makefile 
``` $^ refers to source i.e trim.c and $@ refers to targer i.e. trim.js
CC = emcc
LIBS = -lavcodec -lavutil -lavformat -lx264
INCLUDESPATH = -I/opt/ffmpeg/include
LIBPATH = -L/opt/ffmpeg/lib
LDFLAGS = -s "EXPORT_NAME='createModule'" -s TOTAL_MEMORY=100MB

both.js: both.o
        emcc  -I/opt/ffmpeg/include/ -L/opt/ffmpeg/lib/ -lavcodec -lavformat -lavutil both.c /opt/ffmpeg/lib/libx264.a -o both.js -s WASM=1 -s ENVIRONMENT=web  -s MODULARIZE=1  -s WASM=1 -msimd128 -s SIMD=1  -s USE_ES6_IMPORT_META=0 -s "EXPORTED_RUNTIME_METHODS='writeArrayToMemory','getValue'"  -s "EXPORT_NAME='createModule'" -s ALLOW_MEMORY_GROWTH -s LLD_REPORT_UNDEFINED -s TOTAL_MEMORY=1024MB -O3

#       ${CC} $^ ${LIBPATH} -o $@ ${LIBS} ${LDFLAGS}

both.o: both.c
        ${CC} ${INCLUDESPATH} -c $^

clean:
        rm *.o
        rm *.js
        rm *.wasm
```

## Not much of a speed difference using ffmpeg.wasm

### Compile with SIMD!
https://v8.dev/features/simd
`./configure --enable-simd` so that it passes `-msse`, `-msse2`, `-mavx` or `-mfpu=neon` to the compiler and calls corresponding intrinsics. Then, additionally pass `-msimd128` to enable WebAssembly SIMD too either by using `CFLAGS=-msimd128 make …` / `CXXFLAGS="-msimd128 make …` or by modifying the build config directly when targeting Wasm.

```
pass -msmd128 flag to emcc
```

https://jeromewu.github.io/improving-performance-using-webassembly-simd-intrinsics/


# THREADING
https://web.dev/articles/webassembly-threads
Always wondered what ffmpeg.worker.js file is
So here is how I made one

```
  

#include  <emscripten.h>

#include  <stdio.h>

#include  <unistd.h>

#include  <pthread.h>

  
  

uint8_t *gbuffer1Ptr = NULL;

int  gbuffer1Size = 0;

  

uint8_t *gbuffer2Ptr = NULL;

int  gbuffer2Size = 0;

  

typedef  struct  trimInfo {

int  bufferSize;

uint8_t *buffer;

} trimInfo;

  

uint8_t *EMSCRIPTEN_KEEPALIVE  createVideoDataBuffer(int  pBufferSize)

{

printf("In C createVideoDataBuffer ... allocating space\n");

gbuffer1Size = pBufferSize;

gbuffer1Ptr = malloc(pBufferSize * sizeof(uint8_t));

return gbuffer1Ptr;

}

  

void EMSCRIPTEN_KEEPALIVE freeVideoDataBuffer()

{

printf("In C destroyBufferImage ... freeing space\n");

if(gbuffer1Ptr)

free(gbuffer1Ptr);

if(gbuffer2Ptr)

free(gbuffer2Ptr);

// free(fileinfo_ptr->format_name);

// free(fileinfo_ptr);

}

  
  
  

void *thread_callback(void *arg)

{

printf("Inside the thread Sleeping 5 : %d\n", *(int *)arg);

sleep(5);

printf("Inside the thread woke up: %d\n", *(int *)arg);

return  NULL;

}

  
  
  

trimInfo* EMSCRIPTEN_KEEPALIVE trimVideo()

{

printf("Before the thread\n");

  

pthread_t thread_id;

int arg = 42;

pthread_create(&thread_id, NULL, thread_callback, &arg);

  

pthread_join(thread_id, NULL);

  

puts("After the thread\n");

  

return  NULL;

}
```

makefile
```

Produces
wo

Note: so PROXY_TO_PTHREAD needs a main method always

## Deployment issues

### embed corp error
```
// thread1.js change wasmBinaryFile= to
wasmBinaryFile="http://localhost:3002/threadxx/thread1.wasm"
// pthreadMainJs to 
var pthreadMainJs=locateFile("http://localhost:3002/threadxx/thread1.worker.js")
```
```
// server.js
app.get("/threadxx/:filename", (req, res) => {

const  filename = req.params.filename;

// response headers
res.set('Cross-Origin-Opener-Policy', 'same-origin');
res.set('Cross-Origin-Resource-Policy', 'cross-origin');
res.set('Cross-Origin-Embedder-Policy', 'require-corp'); 

res.sendFile(`/video/ccvideotoolkit/${filename}`, { root:  "public" });

});
```

### createObjectUrl error
edit thread1.js
```
worker.postMessage({"cmd":"load","handlers":handlers,"urlOrBlob":Module["mainScriptUrlOrBlob"]||_scriptDir
```
replace with
```
worker.postMessage({"cmd":"load","handlers":handlers,"urlOrBlob":Module["mainScriptUrlOrBlob"]||(self.location.origin + "/video/ccvideotoolkit/thread1.js")
```


working glue file
```
  

var  createModule = (() => {

var  _scriptDir = typeof  document !== 'undefined' && document.currentScript ? document.currentScript.src : undefined;

return (

function(moduleArg = {}) {

  

function GROWABLE_HEAP_I8(){if(wasmMemory.buffer!=HEAP8.buffer){updateMemoryViews()}return HEAP8}function GROWABLE_HEAP_U8(){if(wasmMemory.buffer!=HEAP8.buffer){updateMemoryViews()}return HEAPU8}function GROWABLE_HEAP_I16(){if(wasmMemory.buffer!=HEAP8.buffer){updateMemoryViews()}return HEAP16}function GROWABLE_HEAP_I32(){if(wasmMemory.buffer!=HEAP8.buffer){updateMemoryViews()}return HEAP32}function GROWABLE_HEAP_U32(){if(wasmMemory.buffer!=HEAP8.buffer){updateMemoryViews()}return HEAPU32}function GROWABLE_HEAP_F32(){if(wasmMemory.buffer!=HEAP8.buffer){updateMemoryViews()}return HEAPF32}function GROWABLE_HEAP_F64(){if(wasmMemory.buffer!=HEAP8.buffer){updateMemoryViews()}return HEAPF64}var Module=moduleArg;var readyPromiseResolve,readyPromiseReject;Module["ready"]=new Promise((resolve,reject)=>{readyPromiseResolve=resolve;readyPromiseReject=reject});["_main","__emscripten_thread_init","__emscripten_thread_exit","__emscripten_thread_crashed","__emscripten_thread_mailbox_await","__emscripten_tls_init","_pthread_self","checkMailbox","establishStackSpace","invokeEntryPoint","PThread","_createVideoDataBuffer","_freeVideoDataBuffer","_trimVideo","___indirect_function_table","_fflush","__emscripten_check_mailbox","onRuntimeInitialized"].forEach(prop=>{if(!Object.getOwnPropertyDescriptor(Module["ready"],prop)){Object.defineProperty(Module["ready"],prop,{get:()=>abort("You are getting "+prop+" on the Promise object, instead of the instance. Use .then() to get called back with the instance, see the MODULARIZE docs in src/settings.js"),set:()=>abort("You are setting "+prop+" on the Promise object, instead of the instance. Use .then() to get called back with the instance, see the MODULARIZE docs in src/settings.js")})}});var moduleOverrides=Object.assign({},Module);var arguments_=[];var thisProgram="./this.program";var quit_=(status,toThrow)=>{throw toThrow};var ENVIRONMENT_IS_WEB=false;var ENVIRONMENT_IS_WORKER=true;var ENVIRONMENT_IS_NODE=false;var ENVIRONMENT_IS_SHELL=false;if(Module["ENVIRONMENT"]){throw new Error("Module.ENVIRONMENT has been deprecated. To force the environment, use the ENVIRONMENT compile-time option (for example, -sENVIRONMENT=web or -sENVIRONMENT=node)")}var ENVIRONMENT_IS_PTHREAD=Module["ENVIRONMENT_IS_PTHREAD"]||false;var scriptDirectory="";function locateFile(path){if(Module["locateFile"]){return Module["locateFile"](path,scriptDirectory)}return scriptDirectory+path}var read_,readAsync,readBinary,setWindowTitle;if(ENVIRONMENT_IS_SHELL){if(typeof process=="object"&&typeof require==="function"||typeof window=="object"||typeof importScripts=="function")throw new Error("not compiled for this environment (did you build to HTML and try to run it not on the web, or set ENVIRONMENT to something - like node - and run it someplace else - like on the web?)");if(typeof read!="undefined"){read_=read}readBinary=f=>{if(typeof readbuffer=="function"){return new Uint8Array(readbuffer(f))}let data=read(f,"binary");assert(typeof data=="object");return data};readAsync=(f,onload,onerror)=>{setTimeout(()=>onload(readBinary(f)))};if(typeof clearTimeout=="undefined"){globalThis.clearTimeout=id=>{}}if(typeof setTimeout=="undefined"){globalThis.setTimeout=f=>typeof f=="function"?f():abort()}if(typeof scriptArgs!="undefined"){arguments_=scriptArgs}else if(typeof arguments!="undefined"){arguments_=arguments}if(typeof quit=="function"){quit_=(status,toThrow)=>{setTimeout(()=>{if(!(toThrow instanceof ExitStatus)){let toLog=toThrow;if(toThrow&&typeof toThrow=="object"&&toThrow.stack){toLog=[toThrow,toThrow.stack]}err(`exiting due to exception: ${toLog}`)}quit(status)});throw toThrow}}if(typeof print!="undefined"){if(typeof console=="undefined")console={};console.log=print;console.warn=console.error=typeof printErr!="undefined"?printErr:print}}else if(ENVIRONMENT_IS_WEB||ENVIRONMENT_IS_WORKER){if(ENVIRONMENT_IS_WORKER){scriptDirectory=self.location.href}else if(typeof document!="undefined"&&document.currentScript){scriptDirectory=document.currentScript.src}if(_scriptDir){scriptDirectory=_scriptDir}if(scriptDirectory.indexOf("blob:")!==0){scriptDirectory=scriptDirectory.substr(0,scriptDirectory.replace(/[?#].*/,"").lastIndexOf("/")+1)}else{scriptDirectory=""}if(!(typeof window=="object"||typeof importScripts=="function"))throw new Error("not compiled for this environment (did you build to HTML and try to run it not on the web, or set ENVIRONMENT to something - like node - and run it someplace else - like on the web?)");{read_=url=>{var xhr=new XMLHttpRequest;xhr.open("GET",url,false);xhr.send(null);return xhr.responseText};if(ENVIRONMENT_IS_WORKER){readBinary=url=>{var xhr=new XMLHttpRequest;xhr.open("GET",url,false);xhr.responseType="arraybuffer";xhr.send(null);return new Uint8Array(xhr.response)}}readAsync=(url,onload,onerror)=>{var xhr=new XMLHttpRequest;xhr.open("GET",url,true);xhr.responseType="arraybuffer";xhr.onload=()=>{if(xhr.status==200||xhr.status==0&&xhr.response){onload(xhr.response);return}onerror()};xhr.onerror=onerror;xhr.send(null)}}setWindowTitle=title=>document.title=title}else{throw new Error("environment detection error")}var out=Module["print"]||console.log.bind(console);var err=Module["printErr"]||console.error.bind(console);Object.assign(Module,moduleOverrides);moduleOverrides=null;checkIncomingModuleAPI();if(Module["arguments"])arguments_=Module["arguments"];legacyModuleProp("arguments","arguments_");if(Module["thisProgram"])thisProgram=Module["thisProgram"];legacyModuleProp("thisProgram","thisProgram");if(Module["quit"])quit_=Module["quit"];legacyModuleProp("quit","quit_");assert(typeof Module["memoryInitializerPrefixURL"]=="undefined","Module.memoryInitializerPrefixURL option was removed, use Module.locateFile instead");assert(typeof Module["pthreadMainPrefixURL"]=="undefined","Module.pthreadMainPrefixURL option was removed, use Module.locateFile instead");assert(typeof Module["cdInitializerPrefixURL"]=="undefined","Module.cdInitializerPrefixURL option was removed, use Module.locateFile instead");assert(typeof Module["filePackagePrefixURL"]=="undefined","Module.filePackagePrefixURL option was removed, use Module.locateFile instead");assert(typeof Module["read"]=="undefined","Module.read option was removed (modify read_ in JS)");assert(typeof Module["readAsync"]=="undefined","Module.readAsync option was removed (modify readAsync in JS)");assert(typeof Module["readBinary"]=="undefined","Module.readBinary option was removed (modify readBinary in JS)");assert(typeof Module["setWindowTitle"]=="undefined","Module.setWindowTitle option was removed (modify setWindowTitle in JS)");assert(typeof Module["TOTAL_MEMORY"]=="undefined","Module.TOTAL_MEMORY has been renamed Module.INITIAL_MEMORY");legacyModuleProp("asm","wasmExports");legacyModuleProp("read","read_");legacyModuleProp("readAsync","readAsync");legacyModuleProp("readBinary","readBinary");legacyModuleProp("setWindowTitle","setWindowTitle");assert(ENVIRONMENT_IS_WEB||ENVIRONMENT_IS_WORKER||ENVIRONMENT_IS_NODE,"Pthreads do not work in this environment yet (need Web Workers, or an alternative to them)");assert(!ENVIRONMENT_IS_WEB,"web environment detected but not enabled at build time. Add 'web' to `-sENVIRONMENT` to enable.");assert(!ENVIRONMENT_IS_NODE,"node environment detected but not enabled at build time. Add 'node' to `-sENVIRONMENT` to enable.");assert(!ENVIRONMENT_IS_SHELL,"shell environment detected but not enabled at build time. Add 'shell' to `-sENVIRONMENT` to enable.");var wasmBinary;if(Module["wasmBinary"])wasmBinary=Module["wasmBinary"];legacyModuleProp("wasmBinary","wasmBinary");var noExitRuntime=Module["noExitRuntime"]||true;legacyModuleProp("noExitRuntime","noExitRuntime");if(typeof WebAssembly!="object"){abort("no native wasm support detected")}var wasmMemory;var wasmExports;var wasmModule;var ABORT=false;var EXITSTATUS;function assert(condition,text){if(!condition){abort("Assertion failed"+(text?": "+text:""))}}var HEAP8,HEAPU8,HEAP16,HEAPU16,HEAP32,HEAPU32,HEAPF32,HEAPF64;function updateMemoryViews(){var b=wasmMemory.buffer;Module["HEAP8"]=HEAP8=new Int8Array(b);Module["HEAP16"]=HEAP16=new Int16Array(b);Module["HEAP32"]=HEAP32=new Int32Array(b);Module["HEAPU8"]=HEAPU8=new Uint8Array(b);Module["HEAPU16"]=HEAPU16=new Uint16Array(b);Module["HEAPU32"]=HEAPU32=new Uint32Array(b);Module["HEAPF32"]=HEAPF32=new Float32Array(b);Module["HEAPF64"]=HEAPF64=new Float64Array(b)}assert(!Module["STACK_SIZE"],"STACK_SIZE can no longer be set at runtime. Use -sSTACK_SIZE at link time");assert(typeof Int32Array!="undefined"&&typeof Float64Array!=="undefined"&&Int32Array.prototype.subarray!=undefined&&Int32Array.prototype.set!=undefined,"JS engine does not provide full typed array support");var INITIAL_MEMORY=Module["INITIAL_MEMORY"]||16777216;legacyModuleProp("INITIAL_MEMORY","INITIAL_MEMORY");assert(INITIAL_MEMORY>=65536,"INITIAL_MEMORY should be larger than STACK_SIZE, was "+INITIAL_MEMORY+"! (STACK_SIZE="+65536+")");if(ENVIRONMENT_IS_PTHREAD){wasmMemory=Module["wasmMemory"]}else{if(Module["wasmMemory"]){wasmMemory=Module["wasmMemory"]}else{wasmMemory=new WebAssembly.Memory({"initial":INITIAL_MEMORY/65536,"maximum":1073741824/65536,"shared":true});if(!(wasmMemory.buffer instanceof SharedArrayBuffer)){err("requested a shared WebAssembly.Memory but the returned buffer is not a SharedArrayBuffer, indicating that while the browser has SharedArrayBuffer it does not have WebAssembly threads support - you may need to set a flag");if(ENVIRONMENT_IS_NODE){err("(on node you may need: --experimental-wasm-threads --experimental-wasm-bulk-memory and/or recent version)")}throw Error("bad memory")}}}updateMemoryViews();INITIAL_MEMORY=wasmMemory.buffer.byteLength;assert(INITIAL_MEMORY%65536===0);var wasmTable;function writeStackCookie(){var max=_emscripten_stack_get_end();assert((max&3)==0);if(max==0){max+=4}GROWABLE_HEAP_U32()[max>>2]=34821223;GROWABLE_HEAP_U32()[max+4>>2]=2310721022;GROWABLE_HEAP_U32()[0>>2]=1668509029}function checkStackCookie(){if(ABORT)return;var max=_emscripten_stack_get_end();if(max==0){max+=4}var cookie1=GROWABLE_HEAP_U32()[max>>2];var cookie2=GROWABLE_HEAP_U32()[max+4>>2];if(cookie1!=34821223||cookie2!=2310721022){abort(`Stack overflow! Stack cookie has been overwritten at ${ptrToString(max)}, expected hex dwords 0x89BACDFE and 0x2135467, but received ${ptrToString(cookie2)} ${ptrToString(cookie1)}`)}if(GROWABLE_HEAP_U32()[0>>2]!=1668509029){abort("Runtime error: The application has corrupted its heap memory area (address zero)!")}}(function(){var h16=new Int16Array(1);var h8=new Int8Array(h16.buffer);h16[0]=25459;if(h8[0]!==115||h8[1]!==99)throw"Runtime error: expected the system to be little-endian! (Run with -sSUPPORT_BIG_ENDIAN to bypass)"})();var __ATPRERUN__=[];var __ATINIT__=[];var __ATPOSTRUN__=[];var runtimeInitialized=false;var runtimeKeepaliveCounter=0;function keepRuntimeAlive(){return noExitRuntime||runtimeKeepaliveCounter>0}function preRun(){assert(!ENVIRONMENT_IS_PTHREAD);if(Module["preRun"]){if(typeof Module["preRun"]=="function")Module["preRun"]=[Module["preRun"]];while(Module["preRun"].length){addOnPreRun(Module["preRun"].shift())}}callRuntimeCallbacks(__ATPRERUN__)}function initRuntime(){assert(!runtimeInitialized);runtimeInitialized=true;if(ENVIRONMENT_IS_PTHREAD)return;checkStackCookie();callRuntimeCallbacks(__ATINIT__)}function postRun(){checkStackCookie();if(ENVIRONMENT_IS_PTHREAD)return;if(Module["postRun"]){if(typeof Module["postRun"]=="function")Module["postRun"]=[Module["postRun"]];while(Module["postRun"].length){addOnPostRun(Module["postRun"].shift())}}callRuntimeCallbacks(__ATPOSTRUN__)}function addOnPreRun(cb){__ATPRERUN__.unshift(cb)}function addOnInit(cb){__ATINIT__.unshift(cb)}function addOnPostRun(cb){__ATPOSTRUN__.unshift(cb)}assert(Math.imul,"This browser does not support Math.imul(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill");assert(Math.fround,"This browser does not support Math.fround(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill");assert(Math.clz32,"This browser does not support Math.clz32(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill");assert(Math.trunc,"This browser does not support Math.trunc(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill");var runDependencies=0;var runDependencyWatcher=null;var dependenciesFulfilled=null;var runDependencyTracking={};function addRunDependency(id){runDependencies++;if(Module["monitorRunDependencies"]){Module["monitorRunDependencies"](runDependencies)}if(id){assert(!runDependencyTracking[id]);runDependencyTracking[id]=1;if(runDependencyWatcher===null&&typeof setInterval!="undefined"){runDependencyWatcher=setInterval(()=>{if(ABORT){clearInterval(runDependencyWatcher);runDependencyWatcher=null;return}var shown=false;for(var dep in runDependencyTracking){if(!shown){shown=true;err("still waiting on run dependencies:")}err("dependency: "+dep)}if(shown){err("(end of list)")}},1e4)}}else{err("warning: run dependency added without ID")}}function removeRunDependency(id){runDependencies--;if(Module["monitorRunDependencies"]){Module["monitorRunDependencies"](runDependencies)}if(id){assert(runDependencyTracking[id]);delete runDependencyTracking[id]}else{err("warning: run dependency removed without ID")}if(runDependencies==0){if(runDependencyWatcher!==null){clearInterval(runDependencyWatcher);runDependencyWatcher=null}if(dependenciesFulfilled){var callback=dependenciesFulfilled;dependenciesFulfilled=null;callback()}}}function abort(what){if(Module["onAbort"]){Module["onAbort"](what)}what="Aborted("+what+")";err(what);ABORT=true;EXITSTATUS=1;var e=new WebAssembly.RuntimeError(what);readyPromiseReject(e);throw e}var FS={error(){abort("Filesystem support (FS) was not included. The problem is that you are using files from JS, but files were not used from C/C++, so filesystem support was not auto-included. You can force-include filesystem support with -sFORCE_FILESYSTEM")},init(){FS.error()},createDataFile(){FS.error()},createPreloadedFile(){FS.error()},createLazyFile(){FS.error()},open(){FS.error()},mkdev(){FS.error()},registerDevice(){FS.error()},analyzePath(){FS.error()},ErrnoError(){FS.error()}};Module["FS_createDataFile"]=FS.createDataFile;Module["FS_createPreloadedFile"]=FS.createPreloadedFile;var dataURIPrefix="data:application/octet-stream;base64,";function isDataURI(filename){return filename.startsWith(dataURIPrefix)}function isFileURI(filename){return filename.startsWith("file://")}function createExportWrapper(name){return function(){assert(runtimeInitialized,`native function \`${name}\` called before runtime initialization`);var f=wasmExports[name];assert(f,`exported native function \`${name}\` not found`);return f.apply(null,arguments)}}var wasmBinaryFile;wasmBinaryFile="http://localhost:3002/threadxx/thread1.wasm";if(!isDataURI(wasmBinaryFile)){wasmBinaryFile=locateFile(wasmBinaryFile)}function getBinarySync(file){if(file==wasmBinaryFile&&wasmBinary){return new Uint8Array(wasmBinary)}if(readBinary){return readBinary(file)}throw"both async and sync fetching of the wasm failed"}function getBinaryPromise(binaryFile){if(!wasmBinary&&(ENVIRONMENT_IS_WEB||ENVIRONMENT_IS_WORKER)){if(typeof fetch=="function"){return fetch(binaryFile,{credentials:"same-origin"}).then(response=>{if(!response["ok"]){throw"failed to load wasm binary file at '"+binaryFile+"'"}return response["arrayBuffer"]()}).catch(()=>getBinarySync(binaryFile))}}return Promise.resolve().then(()=>getBinarySync(binaryFile))}function instantiateArrayBuffer(binaryFile,imports,receiver){return getBinaryPromise(binaryFile).then(binary=>WebAssembly.instantiate(binary,imports)).then(instance=>instance).then(receiver,reason=>{err("failed to asynchronously prepare wasm: "+reason);if(isFileURI(wasmBinaryFile)){err("warning: Loading from a file URI ("+wasmBinaryFile+") is not supported in most browsers. See https://emscripten.org/docs/getting_started/FAQ.html#how-do-i-run-a-local-webserver-for-testing-why-does-my-program-stall-in-downloading-or-preparing")}abort(reason)})}function instantiateAsync(binary,binaryFile,imports,callback){if(!binary&&typeof WebAssembly.instantiateStreaming=="function"&&!isDataURI(binaryFile)&&typeof fetch=="function"){return fetch(binaryFile,{credentials:"same-origin"}).then(response=>{var result=WebAssembly.instantiateStreaming(response,imports);return result.then(callback,function(reason){err("wasm streaming compile failed: "+reason);err("falling back to ArrayBuffer instantiation");return instantiateArrayBuffer(binaryFile,imports,callback)})})}return instantiateArrayBuffer(binaryFile,imports,callback)}function createWasm(){var info={"env":wasmImports,"wasi_snapshot_preview1":wasmImports};function receiveInstance(instance,module){var exports=instance.exports;wasmExports=exports;registerTLSInit(wasmExports["_emscripten_tls_init"]);wasmTable=wasmExports["__indirect_function_table"];assert(wasmTable,"table not found in wasm exports");addOnInit(wasmExports["__wasm_call_ctors"]);wasmModule=module;removeRunDependency("wasm-instantiate");return exports}addRunDependency("wasm-instantiate");var trueModule=Module;function receiveInstantiationResult(result){assert(Module===trueModule,"the Module object should not be replaced during async compilation - perhaps the order of HTML elements is wrong?");trueModule=null;receiveInstance(result["instance"],result["module"])}if(Module["instantiateWasm"]){try{return Module["instantiateWasm"](info,receiveInstance)}catch(e){err("Module.instantiateWasm callback failed with error: "+e);readyPromiseReject(e)}}instantiateAsync(wasmBinary,wasmBinaryFile,info,receiveInstantiationResult).catch(readyPromiseReject);return{}}function legacyModuleProp(prop,newName,incomming=true){if(!Object.getOwnPropertyDescriptor(Module,prop)){Object.defineProperty(Module,prop,{configurable:true,get(){let extra=incomming?" (the initial value can be provided on Module, but after startup the value is only looked for on a local variable of that name)":"";abort(`\`Module.${prop}\` has been replaced by \`${newName}\``+extra)}})}}function ignoredModuleProp(prop){if(Object.getOwnPropertyDescriptor(Module,prop)){abort(`\`Module.${prop}\` was supplied but \`${prop}\` not included in INCOMING_MODULE_JS_API`)}}function isExportedByForceFilesystem(name){return name==="FS_createPath"||name==="FS_createDataFile"||name==="FS_createPreloadedFile"||name==="FS_unlink"||name==="addRunDependency"||name==="FS_createLazyFile"||name==="FS_createDevice"||name==="removeRunDependency"}function missingGlobal(sym,msg){if(typeof globalThis!=="undefined"){Object.defineProperty(globalThis,sym,{configurable:true,get(){warnOnce("`"+sym+"` is not longer defined by emscripten. "+msg);return undefined}})}}missingGlobal("buffer","Please use HEAP8.buffer or wasmMemory.buffer");function missingLibrarySymbol(sym){if(typeof globalThis!=="undefined"&&!Object.getOwnPropertyDescriptor(globalThis,sym)){Object.defineProperty(globalThis,sym,{configurable:true,get(){var msg="`"+sym+"` is a library symbol and not included by default; add it to your library.js __deps or to DEFAULT_LIBRARY_FUNCS_TO_INCLUDE on the command line";var librarySymbol=sym;if(!librarySymbol.startsWith("_")){librarySymbol="$"+sym}msg+=" (e.g. -sDEFAULT_LIBRARY_FUNCS_TO_INCLUDE='"+librarySymbol+"')";if(isExportedByForceFilesystem(sym)){msg+=". Alternatively, forcing filesystem support (-sFORCE_FILESYSTEM) can export this for you"}warnOnce(msg);return undefined}})}unexportedRuntimeSymbol(sym)}function unexportedRuntimeSymbol(sym){if(!Object.getOwnPropertyDescriptor(Module,sym)){Object.defineProperty(Module,sym,{configurable:true,get(){var msg="'"+sym+"' was not exported. add it to EXPORTED_RUNTIME_METHODS (see the Emscripten FAQ)";if(isExportedByForceFilesystem(sym)){msg+=". Alternatively, forcing filesystem support (-sFORCE_FILESYSTEM) can export this for you"}abort(msg)}})}}function dbg(text){console.warn.apply(console,arguments)}function ExitStatus(status){this.name="ExitStatus";this.message=`Program terminated with exit(${status})`;this.status=status}var terminateWorker=function(worker){worker.terminate();worker.onmessage=e=>{var cmd=e["data"]["cmd"];err('received "'+cmd+'" command from terminated worker: '+worker.workerID)}};function killThread(pthread_ptr){assert(!ENVIRONMENT_IS_PTHREAD,"Internal Error! killThread() can only ever be called from main application thread!");assert(pthread_ptr,"Internal Error! Null pthread_ptr in killThread!");var worker=PThread.pthreads[pthread_ptr];delete PThread.pthreads[pthread_ptr];terminateWorker(worker);__emscripten_thread_free_data(pthread_ptr);PThread.runningWorkers.splice(PThread.runningWorkers.indexOf(worker),1);worker.pthread_ptr=0}function cancelThread(pthread_ptr){assert(!ENVIRONMENT_IS_PTHREAD,"Internal Error! cancelThread() can only ever be called from main application thread!");assert(pthread_ptr,"Internal Error! Null pthread_ptr in cancelThread!");var worker=PThread.pthreads[pthread_ptr];worker.postMessage({"cmd":"cancel"})}function cleanupThread(pthread_ptr){assert(!ENVIRONMENT_IS_PTHREAD,"Internal Error! cleanupThread() can only ever be called from main application thread!");assert(pthread_ptr,"Internal Error! Null pthread_ptr in cleanupThread!");var worker=PThread.pthreads[pthread_ptr];assert(worker);PThread.returnWorkerToPool(worker)}function spawnThread(threadParams){assert(!ENVIRONMENT_IS_PTHREAD,"Internal Error! spawnThread() can only ever be called from main application thread!");assert(threadParams.pthread_ptr,"Internal error, no pthread ptr!");var worker=PThread.getNewWorker();if(!worker){return 6}assert(!worker.pthread_ptr,"Internal error!");PThread.runningWorkers.push(worker);PThread.pthreads[threadParams.pthread_ptr]=worker;worker.pthread_ptr=threadParams.pthread_ptr;var msg={"cmd":"run","start_routine":threadParams.startRoutine,"arg":threadParams.arg,"pthread_ptr":threadParams.pthread_ptr};worker.postMessage(msg,threadParams.transferList);return 0}var UTF8Decoder=typeof TextDecoder!="undefined"?new TextDecoder("utf8"):undefined;var UTF8ArrayToString=(heapOrArray,idx,maxBytesToRead)=>{var endIdx=idx+maxBytesToRead;var endPtr=idx;while(heapOrArray[endPtr]&&!(endPtr>=endIdx))++endPtr;if(endPtr-idx>16&&heapOrArray.buffer&&UTF8Decoder){return UTF8Decoder.decode(heapOrArray.buffer instanceof SharedArrayBuffer?heapOrArray.slice(idx,endPtr):heapOrArray.subarray(idx,endPtr))}var str="";while(idx<endPtr){var u0=heapOrArray[idx++];if(!(u0&128)){str+=String.fromCharCode(u0);continue}var u1=heapOrArray[idx++]&63;if((u0&224)==192){str+=String.fromCharCode((u0&31)<<6|u1);continue}var u2=heapOrArray[idx++]&63;if((u0&240)==224){u0=(u0&15)<<12|u1<<6|u2}else{if((u0&248)!=240)warnOnce("Invalid UTF-8 leading byte "+ptrToString(u0)+" encountered when deserializing a UTF-8 string in wasm memory to a JS string!");u0=(u0&7)<<18|u1<<12|u2<<6|heapOrArray[idx++]&63}if(u0<65536){str+=String.fromCharCode(u0)}else{var ch=u0-65536;str+=String.fromCharCode(55296|ch>>10,56320|ch&1023)}}return str};var UTF8ToString=(ptr,maxBytesToRead)=>{assert(typeof ptr=="number");return ptr?UTF8ArrayToString(GROWABLE_HEAP_U8(),ptr,maxBytesToRead):""};var SYSCALLS={varargs:undefined,get(){assert(SYSCALLS.varargs!=undefined);SYSCALLS.varargs+=4;var ret=GROWABLE_HEAP_I32()[SYSCALLS.varargs-4>>2];return ret},getStr(ptr){var ret=UTF8ToString(ptr);return ret}};function _proc_exit(code){if(ENVIRONMENT_IS_PTHREAD)return proxyToMainThread(1,1,code);EXITSTATUS=code;if(!keepRuntimeAlive()){PThread.terminateAllThreads();if(Module["onExit"])Module["onExit"](code);ABORT=true}quit_(code,new ExitStatus(code))}var exitJS=(status,implicit)=>{EXITSTATUS=status;checkUnflushedContent();if(ENVIRONMENT_IS_PTHREAD){assert(!implicit);exitOnMainThread(status);throw"unwind"}if(keepRuntimeAlive()&&!implicit){var msg=`program exited (with status: ${status}), but keepRuntimeAlive() is set (counter=${runtimeKeepaliveCounter}) due to an async operation, so halting execution but not exiting the runtime or preventing further async execution (you can use emscripten_force_exit, if you want to force a true shutdown)`;readyPromiseReject(msg);err(msg)}_proc_exit(status)};var _exit=exitJS;var ptrToString=ptr=>{assert(typeof ptr==="number");ptr>>>=0;return"0x"+ptr.toString(16).padStart(8,"0")};var handleException=e=>{if(e instanceof ExitStatus||e=="unwind"){return EXITSTATUS}checkStackCookie();if(e instanceof WebAssembly.RuntimeError){if(_emscripten_stack_get_current()<=0){err("Stack overflow detected. You can try increasing -sSTACK_SIZE (currently set to 65536)")}}quit_(1,e)};var PThread={unusedWorkers:[],runningWorkers:[],tlsInitFunctions:[],pthreads:{},nextWorkerID:1,debugInit:function(){function pthreadLogPrefix(){var t=0;if(runtimeInitialized&&typeof _pthread_self!="undefined"){t=_pthread_self()}return"w:"+(Module["workerID"]||0)+",t:"+ptrToString(t)+": "}var origDbg=dbg;dbg=message=>origDbg(pthreadLogPrefix()+message)},init:function(){PThread.debugInit();if(ENVIRONMENT_IS_PTHREAD){PThread.initWorker()}else{PThread.initMainThread()}},initMainThread:function(){var pthreadPoolSize=1;while(pthreadPoolSize--){PThread.allocateUnusedWorker()}addOnPreRun(()=>{addRunDependency("loading-workers");PThread.loadWasmModuleToAllWorkers(()=>removeRunDependency("loading-workers"))})},initWorker:function(){noExitRuntime=false},setExitStatus:function(status){EXITSTATUS=status},terminateAllThreads__deps:["$terminateWorker"],terminateAllThreads:function(){assert(!ENVIRONMENT_IS_PTHREAD,"Internal Error! terminateAllThreads() can only ever be called from main application thread!");for(var worker of PThread.runningWorkers){terminateWorker(worker)}for(var worker of PThread.unusedWorkers){terminateWorker(worker)}PThread.unusedWorkers=[];PThread.runningWorkers=[];PThread.pthreads=[]},returnWorkerToPool:function(worker){var pthread_ptr=worker.pthread_ptr;delete PThread.pthreads[pthread_ptr];PThread.unusedWorkers.push(worker);PThread.runningWorkers.splice(PThread.runningWorkers.indexOf(worker),1);worker.pthread_ptr=0;__emscripten_thread_free_data(pthread_ptr)},receiveObjectTransfer:function(data){},threadInitTLS:function(){PThread.tlsInitFunctions.forEach(f=>f())},loadWasmModuleToWorker:worker=>new Promise(onFinishedLoading=>{worker.onmessage=e=>{var d=e["data"];var cmd=d["cmd"];if(d["targetThread"]&&d["targetThread"]!=_pthread_self()){var targetWorker=PThread.pthreads[d.targetThread];if(targetWorker){targetWorker.postMessage(d,d["transferList"])}else{err('Internal error! Worker sent a message "'+cmd+'" to target pthread '+d["targetThread"]+", but that thread no longer exists!")}return}if(cmd==="checkMailbox"){checkMailbox()}else if(cmd==="spawnThread"){spawnThread(d)}else if(cmd==="cleanupThread"){cleanupThread(d["thread"])}else if(cmd==="killThread"){killThread(d["thread"])}else if(cmd==="cancelThread"){cancelThread(d["thread"])}else if(cmd==="loaded"){worker.loaded=true;onFinishedLoading(worker)}else if(cmd==="alert"){alert("Thread "+d["threadId"]+": "+d["text"])}else if(d.target==="setimmediate"){worker.postMessage(d)}else if(cmd==="callHandler"){Module[d["handler"]](...d["args"])}else if(cmd){err("worker sent an unknown command "+cmd)}};worker.onerror=e=>{var message="worker sent an error!";if(worker.pthread_ptr){message="Pthread "+ptrToString(worker.pthread_ptr)+" sent an error!"}err(message+" "+e.filename+":"+e.lineno+": "+e.message);throw e};assert(wasmMemory instanceof WebAssembly.Memory,"WebAssembly memory should have been loaded by now!");assert(wasmModule instanceof WebAssembly.Module,"WebAssembly Module should have been loaded by now!");var handlers=[];var knownHandlers=["onExit","onAbort","print","printErr"];for(var handler of knownHandlers){if(Module.hasOwnProperty(handler)){handlers.push(handler)}}worker.workerID=PThread.nextWorkerID++;worker.postMessage({"cmd":"load","handlers":handlers,"urlOrBlob":Module["mainScriptUrlOrBlob"]||(self.location.origin + "/video/ccvideotoolkit/thread1.js"),"wasmMemory":wasmMemory,"wasmModule":wasmModule,"workerID":worker.workerID})}),loadWasmModuleToAllWorkers:function(onMaybeReady){if(ENVIRONMENT_IS_PTHREAD){return onMaybeReady()}let pthreadPoolReady=Promise.all(PThread.unusedWorkers.map(PThread.loadWasmModuleToWorker));pthreadPoolReady.then(onMaybeReady)},allocateUnusedWorker:function(){var worker;var pthreadMainJs=locateFile("http://localhost:3002/threadxx/thread1.worker.js");worker=new Worker(pthreadMainJs);PThread.unusedWorkers.push(worker)},getNewWorker:function(){if(PThread.unusedWorkers.length==0){err("Tried to spawn a new thread, but the thread pool is exhausted.\n"+"This might result in a deadlock unless some threads eventually exit or the code explicitly breaks out to the event loop.\n"+"If you want to increase the pool size, use setting `-sPTHREAD_POOL_SIZE=...`."+"\nIf you want to throw an explicit error instead of the risk of deadlocking in those cases, use setting `-sPTHREAD_POOL_SIZE_STRICT=2`.");PThread.allocateUnusedWorker();PThread.loadWasmModuleToWorker(PThread.unusedWorkers[0])}return PThread.unusedWorkers.pop()}};Module["PThread"]=PThread;var callRuntimeCallbacks=callbacks=>{while(callbacks.length>0){callbacks.shift()(Module)}};function establishStackSpace(){var pthread_ptr=_pthread_self();var stackHigh=GROWABLE_HEAP_I32()[pthread_ptr+52>>2];var stackSize=GROWABLE_HEAP_I32()[pthread_ptr+56>>2];var stackLow=stackHigh-stackSize;assert(stackHigh!=0);assert(stackLow!=0);assert(stackHigh>stackLow,"stackHigh must be higher then stackLow");_emscripten_stack_set_limits(stackHigh,stackLow);stackRestore(stackHigh);writeStackCookie()}Module["establishStackSpace"]=establishStackSpace;function exitOnMainThread(returnCode){if(ENVIRONMENT_IS_PTHREAD)return proxyToMainThread(2,0,returnCode);_exit(returnCode)}function getValue(ptr,type="i8"){if(type.endsWith("*"))type="*";switch(type){case"i1":return GROWABLE_HEAP_I8()[ptr>>0];case"i8":return GROWABLE_HEAP_I8()[ptr>>0];case"i16":return GROWABLE_HEAP_I16()[ptr>>1];case"i32":return GROWABLE_HEAP_I32()[ptr>>2];case"i64":abort("to do getValue(i64) use WASM_BIGINT");case"float":return GROWABLE_HEAP_F32()[ptr>>2];case"double":return GROWABLE_HEAP_F64()[ptr>>3];case"*":return GROWABLE_HEAP_U32()[ptr>>2];default:abort(`invalid type for getValue: ${type}`)}}var wasmTableMirror=[];var getWasmTableEntry=funcPtr=>{var func=wasmTableMirror[funcPtr];if(!func){if(funcPtr>=wasmTableMirror.length)wasmTableMirror.length=funcPtr+1;wasmTableMirror[funcPtr]=func=wasmTable.get(funcPtr)}assert(wasmTable.get(funcPtr)==func,"JavaScript-side Wasm function table mirror is out of date!");return func};function invokeEntryPoint(ptr,arg){var result=getWasmTableEntry(ptr)(arg);checkStackCookie();function finish(result){if(keepRuntimeAlive()){PThread.setExitStatus(result)}else{__emscripten_thread_exit(result)}}finish(result)}Module["invokeEntryPoint"]=invokeEntryPoint;function registerTLSInit(tlsInitFunc){PThread.tlsInitFunctions.push(tlsInitFunc)}var warnOnce=text=>{if(!warnOnce.shown)warnOnce.shown={};if(!warnOnce.shown[text]){warnOnce.shown[text]=1;err(text)}};var ___assert_fail=(condition,filename,line,func)=>{abort(`Assertion failed: ${UTF8ToString(condition)}, at: `+[filename?UTF8ToString(filename):"unknown filename",line,func?UTF8ToString(func):"unknown function"])};function ___emscripten_init_main_thread_js(tb){__emscripten_thread_init(tb,!ENVIRONMENT_IS_WORKER,1,!ENVIRONMENT_IS_WEB,65536,false);PThread.threadInitTLS()}function ___emscripten_thread_cleanup(thread){if(!ENVIRONMENT_IS_PTHREAD)cleanupThread(thread);else postMessage({"cmd":"cleanupThread","thread":thread})}function pthreadCreateProxied(pthread_ptr,attr,startRoutine,arg){if(ENVIRONMENT_IS_PTHREAD)return proxyToMainThread(3,1,pthread_ptr,attr,startRoutine,arg);return ___pthread_create_js(pthread_ptr,attr,startRoutine,arg)}function ___pthread_create_js(pthread_ptr,attr,startRoutine,arg){if(typeof SharedArrayBuffer=="undefined"){err("Current environment does not support SharedArrayBuffer, pthreads are not available!");return 6}var transferList=[];var error=0;if(ENVIRONMENT_IS_PTHREAD&&(transferList.length===0||error)){return pthreadCreateProxied(pthread_ptr,attr,startRoutine,arg)}if(error)return error;var threadParams={startRoutine:startRoutine,pthread_ptr:pthread_ptr,arg:arg,transferList:transferList};if(ENVIRONMENT_IS_PTHREAD){threadParams.cmd="spawnThread";postMessage(threadParams,transferList);return 0}return spawnThread(threadParams)}var nowIsMonotonic=true;var __emscripten_get_now_is_monotonic=()=>nowIsMonotonic;var maybeExit=()=>{if(!keepRuntimeAlive()){try{if(ENVIRONMENT_IS_PTHREAD)__emscripten_thread_exit(EXITSTATUS);else _exit(EXITSTATUS)}catch(e){handleException(e)}}};var callUserCallback=func=>{if(ABORT){err("user callback triggered after runtime exited or application aborted. Ignoring.");return}try{func();maybeExit()}catch(e){handleException(e)}};function __emscripten_thread_mailbox_await(pthread_ptr){if(typeof Atomics.waitAsync==="function"){var wait=Atomics.waitAsync(GROWABLE_HEAP_I32(),pthread_ptr>>2,pthread_ptr);assert(wait.async);wait.value.then(checkMailbox);var waitingAsync=pthread_ptr+128;Atomics.store(GROWABLE_HEAP_I32(),waitingAsync>>2,1)}}Module["__emscripten_thread_mailbox_await"]=__emscripten_thread_mailbox_await;var checkMailbox=function(){var pthread_ptr=_pthread_self();if(pthread_ptr){__emscripten_thread_mailbox_await(pthread_ptr);callUserCallback(()=>__emscripten_check_mailbox())}};Module["checkMailbox"]=checkMailbox;var __emscripten_notify_mailbox_postmessage=function(targetThreadId,currThreadId,mainThreadId){if(targetThreadId==currThreadId){setTimeout(()=>checkMailbox())}else if(ENVIRONMENT_IS_PTHREAD){postMessage({"targetThread":targetThreadId,"cmd":"checkMailbox"})}else{var worker=PThread.pthreads[targetThreadId];if(!worker){err("Cannot send message to thread with ID "+targetThreadId+", unknown thread ID!");return}worker.postMessage({"cmd":"checkMailbox"})}};function __emscripten_set_offscreencanvas_size(target,width,height){err("emscripten_set_offscreencanvas_size: Build with -sOFFSCREENCANVAS_SUPPORT=1 to enable transferring canvases to pthreads.");return-1}function __emscripten_thread_set_strongref(thread){}function _emscripten_check_blocking_allowed(){if(ENVIRONMENT_IS_WORKER)return;warnOnce("Blocking on the main thread is very dangerous, see https://emscripten.org/docs/porting/pthreads.html#blocking-on-the-main-browser-thread")}function _emscripten_date_now(){return Date.now()}var runtimeKeepalivePush=()=>{runtimeKeepaliveCounter+=1};var _emscripten_exit_with_live_runtime=()=>{runtimeKeepalivePush();throw"unwind"};var _emscripten_get_now;_emscripten_get_now=()=>performance.timeOrigin+performance.now();var withStackSave=f=>{var stack=stackSave();var ret=f();stackRestore(stack);return ret};function proxyToMainThread(index,sync){var numCallArgs=arguments.length-2;var outerArgs=arguments;var maxArgs=19;if(numCallArgs>maxArgs){throw"proxyToMainThread: Too many arguments "+numCallArgs+" to proxied function idx="+index+", maximum supported is "+maxArgs}return withStackSave(()=>{var serializedNumCallArgs=numCallArgs;var args=stackAlloc(serializedNumCallArgs*8);var b=args>>3;for(var i=0;i<numCallArgs;i++){var arg=outerArgs[2+i];GROWABLE_HEAP_F64()[b+i]=arg}return __emscripten_run_in_main_runtime_thread_js(index,serializedNumCallArgs,args,sync)})}var emscripten_receive_on_main_thread_js_callArgs=[];function _emscripten_receive_on_main_thread_js(index,callingThread,numCallArgs,args){PThread.currentProxiedOperationCallerThread=callingThread;emscripten_receive_on_main_thread_js_callArgs.length=numCallArgs;var b=args>>3;for(var i=0;i<numCallArgs;i++){emscripten_receive_on_main_thread_js_callArgs[i]=GROWABLE_HEAP_F64()[b+i]}var func=proxiedFunctionTable[index];assert(func.length==numCallArgs,"Call args mismatch in emscripten_receive_on_main_thread_js");return func.apply(null,emscripten_receive_on_main_thread_js_callArgs)}var getHeapMax=()=>1073741824;var growMemory=size=>{var b=wasmMemory.buffer;var pages=size-b.byteLength+65535>>>16;try{wasmMemory.grow(pages);updateMemoryViews();return 1}catch(e){err(`growMemory: Attempted to grow heap from ${b.byteLength} bytes to ${size} bytes, but got error: ${e}`)}};var _emscripten_resize_heap=requestedSize=>{var oldSize=GROWABLE_HEAP_U8().length;requestedSize>>>=0;if(requestedSize<=oldSize){return false}var maxHeapSize=getHeapMax();if(requestedSize>maxHeapSize){err(`Cannot enlarge memory, asked to go up to ${requestedSize} bytes, but the limit is ${maxHeapSize} bytes!`);return false}var alignUp=(x,multiple)=>x+(multiple-x%multiple)%multiple;for(var cutDown=1;cutDown<=4;cutDown*=2){var overGrownHeapSize=oldSize*(1+.2/cutDown);overGrownHeapSize=Math.min(overGrownHeapSize,requestedSize+100663296);var newSize=Math.min(maxHeapSize,alignUp(Math.max(requestedSize,overGrownHeapSize),65536));var replacement=growMemory(newSize);if(replacement){return true}}err(`Failed to grow the heap from ${oldSize} bytes to ${newSize} bytes, not enough memory!`);return false};var printCharBuffers=[null,[],[]];var printChar=(stream,curr)=>{var buffer=printCharBuffers[stream];assert(buffer);if(curr===0||curr===10){(stream===1?out:err)(UTF8ArrayToString(buffer,0));buffer.length=0}else{buffer.push(curr)}};var flush_NO_FILESYSTEM=()=>{_fflush(0);if(printCharBuffers[1].length)printChar(1,10);if(printCharBuffers[2].length)printChar(2,10)};function _fd_write(fd,iov,iovcnt,pnum){if(ENVIRONMENT_IS_PTHREAD)return proxyToMainThread(4,1,fd,iov,iovcnt,pnum);var num=0;for(var i=0;i<iovcnt;i++){var ptr=GROWABLE_HEAP_U32()[iov>>2];var len=GROWABLE_HEAP_U32()[iov+4>>2];iov+=8;for(var j=0;j<len;j++){printChar(fd,GROWABLE_HEAP_U8()[ptr+j])}num+=len}GROWABLE_HEAP_U32()[pnum>>2]=num;return 0}var writeArrayToMemory=(array,buffer)=>{assert(array.length>=0,"writeArrayToMemory array must have a length (should be an array or typed array)");GROWABLE_HEAP_I8().set(array,buffer)};PThread.init();var proxiedFunctionTable=[null,_proc_exit,exitOnMainThread,pthreadCreateProxied,_fd_write];function checkIncomingModuleAPI(){ignoredModuleProp("fetchSettings")}var wasmImports={__assert_fail:___assert_fail,__emscripten_init_main_thread_js:___emscripten_init_main_thread_js,__emscripten_thread_cleanup:___emscripten_thread_cleanup,__pthread_create_js:___pthread_create_js,_emscripten_get_now_is_monotonic:__emscripten_get_now_is_monotonic,_emscripten_notify_mailbox_postmessage:__emscripten_notify_mailbox_postmessage,_emscripten_set_offscreencanvas_size:__emscripten_set_offscreencanvas_size,_emscripten_thread_mailbox_await:__emscripten_thread_mailbox_await,_emscripten_thread_set_strongref:__emscripten_thread_set_strongref,emscripten_check_blocking_allowed:_emscripten_check_blocking_allowed,emscripten_date_now:_emscripten_date_now,emscripten_exit_with_live_runtime:_emscripten_exit_with_live_runtime,emscripten_get_now:_emscripten_get_now,emscripten_receive_on_main_thread_js:_emscripten_receive_on_main_thread_js,emscripten_resize_heap:_emscripten_resize_heap,exit:_exit,fd_write:_fd_write,memory:wasmMemory||Module["wasmMemory"]};var asm=createWasm();var ___wasm_call_ctors=createExportWrapper("__wasm_call_ctors");var _createVideoDataBuffer=Module["_createVideoDataBuffer"]=createExportWrapper("createVideoDataBuffer");var _freeVideoDataBuffer=Module["_freeVideoDataBuffer"]=createExportWrapper("freeVideoDataBuffer");var _trimVideo=Module["_trimVideo"]=createExportWrapper("trimVideo");var __emscripten_tls_init=Module["__emscripten_tls_init"]=createExportWrapper("_emscripten_tls_init");var _pthread_self=Module["_pthread_self"]=()=>(_pthread_self=Module["_pthread_self"]=wasmExports["pthread_self"])();var ___errno_location=createExportWrapper("__errno_location");var __emscripten_thread_init=Module["__emscripten_thread_init"]=createExportWrapper("_emscripten_thread_init");var __emscripten_thread_crashed=Module["__emscripten_thread_crashed"]=createExportWrapper("_emscripten_thread_crashed");var _fflush=Module["_fflush"]=createExportWrapper("fflush");var _emscripten_main_runtime_thread_id=createExportWrapper("emscripten_main_runtime_thread_id");var _emscripten_main_thread_process_queued_calls=createExportWrapper("emscripten_main_thread_process_queued_calls");var __emscripten_run_in_main_runtime_thread_js=createExportWrapper("_emscripten_run_in_main_runtime_thread_js");var _emscripten_dispatch_to_thread_=createExportWrapper("emscripten_dispatch_to_thread_");var __emscripten_thread_free_data=createExportWrapper("_emscripten_thread_free_data");var __emscripten_thread_exit=Module["__emscripten_thread_exit"]=createExportWrapper("_emscripten_thread_exit");var _emscripten_stack_get_base=()=>(_emscripten_stack_get_base=wasmExports["emscripten_stack_get_base"])();var _emscripten_stack_get_end=()=>(_emscripten_stack_get_end=wasmExports["emscripten_stack_get_end"])();var __emscripten_check_mailbox=Module["__emscripten_check_mailbox"]=createExportWrapper("_emscripten_check_mailbox");var _emscripten_stack_init=()=>(_emscripten_stack_init=wasmExports["emscripten_stack_init"])();var _emscripten_stack_set_limits=(a0,a1)=>(_emscripten_stack_set_limits=wasmExports["emscripten_stack_set_limits"])(a0,a1);var _emscripten_stack_get_free=()=>(_emscripten_stack_get_free=wasmExports["emscripten_stack_get_free"])();var stackSave=createExportWrapper("stackSave");var stackRestore=createExportWrapper("stackRestore");var stackAlloc=createExportWrapper("stackAlloc");var _emscripten_stack_get_current=()=>(_emscripten_stack_get_current=wasmExports["emscripten_stack_get_current"])();var dynCall_jiji=Module["dynCall_jiji"]=createExportWrapper("dynCall_jiji");Module["keepRuntimeAlive"]=keepRuntimeAlive;Module["wasmMemory"]=wasmMemory;Module["getValue"]=getValue;Module["writeArrayToMemory"]=writeArrayToMemory;Module["ExitStatus"]=ExitStatus;Module["PThread"]=PThread;var missingLibrarySymbols=["writeI53ToI64","writeI53ToI64Clamped","writeI53ToI64Signaling","writeI53ToU64Clamped","writeI53ToU64Signaling","readI53FromI64","readI53FromU64","convertI32PairToI53","convertI32PairToI53Checked","convertU32PairToI53","isLeapYear","ydayFromDate","arraySum","addDays","setErrNo","inetPton4","inetNtop4","inetPton6","inetNtop6","readSockaddr","writeSockaddr","getHostByName","initRandomFill","randomFill","getCallstack","emscriptenLog","convertPCtoSourceLocation","readEmAsmArgs","jstoi_q","jstoi_s","getExecutableName","listenOnce","autoResumeAudioContext","dynCallLegacy","getDynCaller","dynCall","runtimeKeepalivePop","safeSetTimeout","asmjsMangle","asyncLoad","alignMemory","mmapAlloc","handleAllocatorInit","HandleAllocator","getNativeTypeSize","STACK_SIZE","STACK_ALIGN","POINTER_SIZE","ASSERTIONS","getCFunc","ccall","cwrap","uleb128Encode","sigToWasmTypes","generateFuncType","convertJsFunctionToWasm","getEmptyTableSlot","updateTableMap","getFunctionAddress","addFunction","removeFunction","reallyNegative","unSign","strLen","reSign","formatString","stringToUTF8Array","stringToUTF8","lengthBytesUTF8","intArrayFromString","intArrayToString","AsciiToString","stringToAscii","UTF16ToString","stringToUTF16","lengthBytesUTF16","UTF32ToString","stringToUTF32","lengthBytesUTF32","stringToNewUTF8","stringToUTF8OnStack","registerKeyEventCallback","maybeCStringToJsString","findEventTarget","findCanvasEventTarget","getBoundingClientRect","fillMouseEventData","registerMouseEventCallback","registerWheelEventCallback","registerUiEventCallback","registerFocusEventCallback","fillDeviceOrientationEventData","registerDeviceOrientationEventCallback","fillDeviceMotionEventData","registerDeviceMotionEventCallback","screenOrientation","fillOrientationChangeEventData","registerOrientationChangeEventCallback","fillFullscreenChangeEventData","registerFullscreenChangeEventCallback","JSEvents_requestFullscreen","JSEvents_resizeCanvasForFullscreen","registerRestoreOldStyle","hideEverythingExceptGivenElement","restoreHiddenElements","setLetterbox","softFullscreenResizeWebGLRenderTarget","doRequestFullscreen","fillPointerlockChangeEventData","registerPointerlockChangeEventCallback","registerPointerlockErrorEventCallback","requestPointerLock","fillVisibilityChangeEventData","registerVisibilityChangeEventCallback","registerTouchEventCallback","fillGamepadEventData","registerGamepadEventCallback","registerBeforeUnloadEventCallback","fillBatteryEventData","battery","registerBatteryEventCallback","setCanvasElementSizeCallingThread","setCanvasElementSizeMainThread","setCanvasElementSize","getCanvasSizeCallingThread","getCanvasSizeMainThread","getCanvasElementSize","demangle","demangleAll","jsStackTrace","stackTrace","getEnvStrings","checkWasiClock","wasiRightsToMuslOFlags","wasiOFlagsToMuslOFlags","createDyncallWrapper","setImmediateWrapped","clearImmediateWrapped","polyfillSetImmediate","getPromise","makePromise","idsToPromises","makePromiseCallback","ExceptionInfo","findMatchingCatch","setMainLoop","getSocketFromFD","getSocketAddress","FS_createPreloadedFile","FS_modeStringToFlags","FS_getMode","FS_stdin_getChar","_setNetworkCallback","heapObjectForWebGLType","heapAccessShiftForWebGLHeap","webgl_enable_ANGLE_instanced_arrays","webgl_enable_OES_vertex_array_object","webgl_enable_WEBGL_draw_buffers","webgl_enable_WEBGL_multi_draw","emscriptenWebGLGet","computeUnpackAlignedImageSize","colorChannelsInGlTextureFormat","emscriptenWebGLGetTexPixelData","__glGenObject","emscriptenWebGLGetUniform","webglGetUniformLocation","webglPrepareUniformLocationsBeforeFirstUse","webglGetLeftBracePos","emscriptenWebGLGetVertexAttrib","__glGetActiveAttribOrUniform","writeGLArray","emscripten_webgl_destroy_context_before_on_calling_thread","registerWebGlEventCallback","runAndAbortIfError","SDL_unicode","SDL_ttfContext","SDL_audio","GLFW_Window","ALLOC_NORMAL","ALLOC_STACK","allocate","writeStringToMemory","writeAsciiToMemory"];missingLibrarySymbols.forEach(missingLibrarySymbol);var unexportedSymbols=["run","addOnPreRun","addOnInit","addOnPreMain","addOnExit","addOnPostRun","addRunDependency","removeRunDependency","FS_createFolder","FS_createPath","FS_createDataFile","FS_createLazyFile","FS_createLink","FS_createDevice","FS_unlink","out","err","callMain","abort","wasmTable","wasmExports","stackAlloc","stackSave","stackRestore","getTempRet0","setTempRet0","GROWABLE_HEAP_I8","GROWABLE_HEAP_U8","GROWABLE_HEAP_I16","GROWABLE_HEAP_U16","GROWABLE_HEAP_I32","GROWABLE_HEAP_U32","GROWABLE_HEAP_F32","GROWABLE_HEAP_F64","writeStackCookie","checkStackCookie","ptrToString","zeroMemory","exitJS","getHeapMax","growMemory","ENV","MONTH_DAYS_REGULAR","MONTH_DAYS_LEAP","MONTH_DAYS_REGULAR_CUMULATIVE","MONTH_DAYS_LEAP_CUMULATIVE","ERRNO_CODES","ERRNO_MESSAGES","DNS","Protocols","Sockets","timers","warnOnce","UNWIND_CACHE","readEmAsmArgsArray","handleException","runtimeKeepalivePush","callUserCallback","maybeExit","freeTableIndexes","functionsInTableMap","setValue","PATH","PATH_FS","UTF8Decoder","UTF8ArrayToString","UTF8ToString","UTF16Decoder","JSEvents","specialHTMLTargets","currentFullscreenStrategy","restoreOldWindowedStyle","flush_NO_FILESYSTEM","promiseMap","uncaughtExceptionCount","exceptionLast","exceptionCaught","Browser","wget","SYSCALLS","preloadPlugins","FS_stdin_getChar_buffer","FS","MEMFS","TTY","PIPEFS","SOCKFS","tempFixedLengthArray","miniTempWebGLFloatBuffers","miniTempWebGLIntBuffers","GL","emscripten_webgl_power_preferences","AL","GLUT","EGL","GLEW","IDBStore","SDL","SDL_gfx","GLFW","allocateUTF8","allocateUTF8OnStack","terminateWorker","killThread","cleanupThread","registerTLSInit","cancelThread","spawnThread","exitOnMainThread","proxyToMainThread","emscripten_receive_on_main_thread_js_callArgs","invokeEntryPoint","checkMailbox"];unexportedSymbols.forEach(unexportedRuntimeSymbol);var calledRun;dependenciesFulfilled=function runCaller(){if(!calledRun)run();if(!calledRun)dependenciesFulfilled=runCaller};function stackCheckInit(){assert(!ENVIRONMENT_IS_PTHREAD);_emscripten_stack_init();writeStackCookie()}function run(){if(runDependencies>0){return}if(!ENVIRONMENT_IS_PTHREAD)stackCheckInit();if(ENVIRONMENT_IS_PTHREAD){readyPromiseResolve(Module);initRuntime();startWorker(Module);return}preRun();if(runDependencies>0){return}function doRun(){if(calledRun)return;calledRun=true;Module["calledRun"]=true;if(ABORT)return;initRuntime();readyPromiseResolve(Module);if(Module["onRuntimeInitialized"])Module["onRuntimeInitialized"]();assert(!Module["_main"],'compiled without a main, but one is present. if you added it from JS, use Module["onRuntimeInitialized"]');postRun()}if(Module["setStatus"]){Module["setStatus"]("Running...");setTimeout(function(){setTimeout(function(){Module["setStatus"]("")},1);doRun()},1)}else{doRun()}checkStackCookie()}function checkUnflushedContent(){var oldOut=out;var oldErr=err;var has=false;out=err=x=>{has=true};try{flush_NO_FILESYSTEM()}catch(e){}out=oldOut;err=oldErr;if(has){warnOnce("stdio streams had content in them that was not flushed. you should set EXIT_RUNTIME to 1 (see the Emscripten FAQ), or make sure to emit a newline when you printf etc.");warnOnce("(this may also be due to not including full filesystem support - try building with -sFORCE_FILESYSTEM)")}}if(Module["preInit"]){if(typeof Module["preInit"]=="function")Module["preInit"]=[Module["preInit"]];while(Module["preInit"].length>0){Module["preInit"].pop()()}}run();

  
  

return  moduleArg.ready

}

  

);

})();

if (typeof  exports === 'object' && typeof  module === 'object')

module.exports = createModule;

else  if (typeof  define === 'function' && define['amd'])

define([], () =>  createModule);
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyNTE3OTMxMzksLTQ2OTUwMzk4MiwxND
Y2NjY0MzUyLC0xNzQyMDUyNDgyLC02MjA5NjkyNTcsLTE1ODY3
NjE1MDIsMTM4Mzc0NjUyNiwtMTAyOTc5MDA1NiwtOTU3MzMxMz
A5LDE1NjQ4OTI5NjMsLTQ4NDAzNTE0OCwtMTA2MjQ3MjIxMywt
MzE3ODY1NjUsMjA4OTA4NDEwMywtOTk0ODE3Nzk3LC0yMTAxMD
QyNzQ0LC0xNzA4NTg5Njk2LC0xNDA3Nzk3MTM1LDk0OTIxMTE1
NCw1OTM4MDI0ODJdfQ==
-->