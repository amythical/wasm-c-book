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
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwMjk3OTAwNTYsLTk1NzMzMTMwOSwxNT
Y0ODkyOTYzLC00ODQwMzUxNDgsLTEwNjI0NzIyMTMsLTMxNzg2
NTY1LDIwODkwODQxMDMsLTk5NDgxNzc5NywtMjEwMTA0Mjc0NC
wtMTcwODU4OTY5NiwtMTQwNzc5NzEzNSw5NDkyMTExNTQsNTkz
ODAyNDgyLC0xNjMwNDEzMDUyXX0=
-->