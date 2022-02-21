# RubikDetector <img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/icons/rubikdetector_icon_small.png" width="60px" />  

[ ![Download](https://api.bintray.com/packages/cjurjiu/cjurjiu-opensource/rubikdetector-android/images/download.svg) ](https://bintray.com/cjurjiu/cjurjiu-opensource/rubikdetector-android/_latestVersion) [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://tldrlegal.com/license/apache-license-2.0-(apache-2.0))

Image Processing library capable of detecing a Rubik's Cube in an input frame. Developed for mobile, but 100% portable.

Drawing found facelets as rectangles | Drawing found facelets as circles
:--: | :--: 
<img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/gifs/2.gif"/> | <img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/gifs/1.gif"/>

The Android library also has a connector to the popular [FotoApparat](https://github.com/Fotoapparat/Fotoapparat) camera library. See more [here](https://github.com/cjurjiu/RubikDetector-Android).

The information available here relates to the Android library version of RubikDetector. For details on the C++ code, and on the desktop demo, see the README available in the respective folders.

## Table of contents
  * [Folder Structure](#folder-structure)
  * [RubikDetector Usage](#rubikdetector-usage)
  * [Showing things on screen](#showing-things-on-screen)
  * [Fotoapparat Connector](#fotoapparat-connector-)
  * [Memory layout](#memory-layout) 
  * [RubikDetector as a learning tool/Debug mode](#rubikdetector-as-a-learning-tooldebug-mode)
  * [How to build](#how-to-build)
  * [Known issues](#known-issues)
  * [Misc](#misc) 
  * [Binaries](#binaries)
    * [RubikDetector](#rubikdetector-binaries)
    * [Fotoapparat Connector](#fotoapparat-connector-binaries)

## Folder Structure:

<pre>
.
├── RubikDetectorCore         ->     C++ image processing library (the core)
├── RubikDetectorAndroid      ->     Android Library, FotoApparat Connector, Demo Project
│   ├── app                   ->     Android demo project
│   ├── rubikdetector         ->     RubikDetector Android library
│   └── fotoapparatconnector  ->     Fotoapparat Connector for the RubikDetector Android library
|
├── RubikDetectorLinux        ->     Desktop CMake project, demo for usage on UNIX systems
├── LICENSE
└── README.md
</pre>

## RubikDetector Usage

Using the RubikDetector library directly gives you, the library client, great control over the implementation. However, setting everything up correctly can be somewhat tedios.

Using RubikDetector via the Fotoapparat Connector makes things significantly easier, but comes at a significant performance cost. ¯\\\_(ツ)\_/¯

To use the RubikDetector library directly, first you need to create your RubikDetector instance:

```java
RubikDetector rubikDetector = new RubikDetector.Builder()
                // the input image will have a resolution of 1080x1920
                .inputFrameSize(1080, 1920)
                // the input image data will be stored in the YUV NV21 format
                .inputFrameFormat(RubikDetector.ImageFormat.YUV_NV21)
                // draw found facelets as colored rectangles
                .drawConfig(DrawConfig.Rectangles())
                // builds the RubikDetector for the given params.
                .build();
```                

If you want to specify the image format as an `android.graphics.ImageFormat`, you can use the `RubikDetectorUtils` class to convert it to the equivalent `RubikDetector.ImageFormat`:

```java
@RubikDetector.ImageFormat int detectorImageFormat = RubikDetectorUtils.convertAndroidImageFormat(android.graphics.ImageFormat.NV21);
```                 
The detector expects a `byte[]` of a specific size (see [memory layout](#memory-layout)), which will contain the input frame. The input frame data is expected to be placed at the beggining of the `byte[]`.

To perform the detection, just call:

```java
// getImageData() should return a the byte[] which will contain the image data, and which 
// will have the size expected by the detector
byte[] imageData = getImageData();
// searches for the cube in the data present in the "imageData" byte[]
RubikFacelet[][] facelets = rubikDetector.findCube(imageData);
if(facelets != null) {
    // we found the cube in the frame!
} else {
    // the cube was not found
}
```

The `byte[]` which contains the image, and which will be passed to `RubikDetector#findCube(...)`, needs to have a size at least equal to the total memory required by the RubikDetector. This total memory value is computed based on the dimensions and format of the input frame ( 1080 x 1920, YUV NV21, in this example), and will be larger than what is required to store stictly the input frame in memory. 

To ensure the `byte[]` you pass to the `RubikDetector` has the adquate capacity, perform the following:

```java
//get the image data, store it in a byte[]
byte[] image = getImageFromCamera()

// allocate a ByteBuffer of the capacity required by the RubikDetector
ByteBuffer imageDataBuffer = ByteBuffer.allocateDirect(rubikDetector.getRequiredMemory());

// copy the bytes of the image in the created buffer
imageDataBuffer.put(image, 0, image.length);

// call findCube passing the backing array of the ByteBuffer as a parameter to the RubikDetector
final RubikFacelet[][] result = rubikDetector.findCube(imageDataBuffer.array());
```

This would work just fine if you need to find the cube in a single image. However, this approach is not ideal if you want to perform the detection on a live video stream, as it would allocate a new (huge) buffer each frame, and would then copy the current frame to the newly allocated buffer.

Needless to say, both the allocation, and the copy are expensive operations which would seriously affect the framerate, and would cause considerable memory churn.

An implementation better fitted for live processing would imply the following:
  * preallocate buffers of the size required by the RubikDetector, before camera/video frames start coming in
  * ensure the camera or video frames are written into the preallocated buffers. 
  * send the preallocated buffers to the detector for processing.
  * once processing has been performed, reuse each preallocated buffer.
  
In code, this means the following (Android Camera1 API):

```java
android.hardware.Camera camera = Camera.open(MAIN_CAMERA_ID);

//add 3 buffers to the camera. 3 is just an arbitrary value...pick whatever suits your code best
camera.addCallbackBuffer(ByteBuffer.allocateDirect(rubikDetector.getRequiredMemory()).array());
camera.addCallbackBuffer(ByteBuffer.allocateDirect(rubikDetector.getRequiredMemory()).array());
camera.addCallbackBuffer(ByteBuffer.allocateDirect(rubikDetector.getRequiredMemory()).array());

// set camera params, configure other options
...
// start the camera preview
camera.startPreview();
```

Then in `onPreviewFrame(...)`:
```java
@Override
public void onPreviewFrame(byte[] data, Camera camera) {
    //the byte[] data parameter is one of the buffers preallocated earlier, and has the
    //size required by the RubikDetector
    
    //additionally now it also contains the preview frame! we can just call  findCube(...) on it
    final RubikFacelet[][] result = rubikDetector.findCube(data);
    
    if(facelets != null) {
        // we found the cube in the frame!
    } else {
        // the cube was not found
    }
    
    //finally, add the byte array back as a buffer, so it can be reused.
    camera.addCallbackBuffer(data);
}
```

This technique is used in the Android demo app, in the LiveProcessingActivity. Camera2 sample is also in the works.

## Showing things on screen

### Getting the frame being processed

When searching for facelets within the input frame, the detector creates an RGBA copy of the input frame. This copy is regarded as being the "output frame", and its image data will be stored in the RGBA format, regardless of the data format of the input frame.

It's called the "output frame" because this is a read-to-display, already converted to RGBA, version of the input data. 

This output frame will be written back into the same `byte[]` which also stores the input frame. The output frame is always created, even if the cube is not found in the correspunding input frame.

To prevent the ouput frame overriding the input frame, the output frame will be written at a certain offset in the `byte[]`.

To get the ouput frame as a bitmap:

```java
// get a byte[] with sufficient capacity, which also contains the input frame 
byte[] data = getByteArrayWithInputFrame()
        
// search for the Rubik's Cube in the input frame. regardless whether the cube
// is found or not, this creates the ouput frame.
RubikFacelet[][] facelets = rubikDetector.findCube(data);
        
// allocate a buffer of suficient capacity to store the output frame.
ByteBuffer drawingBuffer = ByteBuffer.allocate(rubikDetector.getResultFrameByteCount());
//copy the output frame data into the buffer
drawingBuffer.put(data, rubikDetector.getResultFrameBufferOffset(), rubikDetector.getResultFrameByteCount());
drawingBuffer.rewind();
        
// create a bitmap for the output frame
Bitmap drawingBitmap = Bitmap.createBitmap(frameWidth, frameHeight, Bitmap.Config.ARGB_8888);
// write the output frame data to the bitmap
drawingBitmap.copyPixelsFromBuffer(drawingBuffer);
```
`BitmapFactory.decoteByteArray(...)` can also be used as an one line alternative. Most likely however, in practice, the bitmap + buffer initialization and data ssignment will take place in different places. 

This make the above code slightly better for demonstration purposes, although longer.

### Drawing the facelets

The detector is capable of drawing the found facelets, as seem below:

<img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/gifs/2.gif"/>

To configure how the frames are drawn, you can supply a `DrawConfig` instance to the builder of the RubikProcessor, or to `RubikDetector#setDrawConfig(...)`. 

Drawing is performed in C++, right after a succesful detection. The result of the drawing will be available in the output frame described at the previous section.

To disable drawing, use `DrawConfig.DoNotDraw()`. If `DrawConfig.DoNotDraw()` is used, but at some point you still want to draw the found facelets, then `RubikDetectorUtils.drawFaceletsAsCircles(...)` & `RubikDetectorUtils.drawFaceletsAsRectangles(...)` can be used.

These methods expect, among others, a canvas to which to draw the facelets. When using these methods, drawing will be performed using the standard Android Canvas drawing API.

Regardless whether drawing is performed in C++ or Java, the original image frame on which the detection ocurred, is left untouched. 

## Fotoapparat Connector <img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/icons/fotoapparat_connector_icon.png" width="40px"/>

Using the RubikDetector via the Fotoapparat Connector makes the setup significantly simpler, but comes at a performance cost.

| Detection with the FotoApparat Connector | Detection with RubikDetector directly, no FotoApparat Connector|
| :--: | :--: |
| <img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/gifs/3.gif"/> | <img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/gifs/1.gif" /> |


To use the connector first you need to place a `RubikDetectorResultView` in your layout file (let's call it `activity_foto_apparat.xml`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:rubik="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.catalinjurjiu.rubikdetectorfotoapparatconnector.view.RubikDetectorResultView
        android:id="@+id/rubik_results_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        rubik:faceletsDrawMode="filledCircles"
        rubik:strokeWidth="3dp"/>
</FrameLayout>
```
The `RubikDetectorResultView` is just a custom view which contains an `io.fotoapparat.view.CameraView`, and knows how to render the facelets over the camera preview.

Then, in your activity:

```java
public class FotoApparatActivity extends Activity {

    private static final String TAG = FotoApparatActivity.class.getSimpleName();
    private Fotoapparat fotoapparat;
    private RubikDetector rubikDetector;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_foto_apparat);

        rubikDetector = new RubikDetector.Builder()
                .drawConfig(DrawConfig.FilledCircles())
                .debuggable(true)
                .build();
                
        RubikDetectorResultView rubikDetectorResultView = findViewById(R.id.rubik_results_view);

        fotoapparat = FotoApparatConnector.configure(Fotoapparat.with(this.getBaseContext()), rubikDetector, rubikDetectorResultView)
                .build();
    }
    ...
}
```
Then, in `onStart()`,`onStop()` and `onDestroy()`:

```java
@Override
protected void onStart() {
    super.onStart();
    fotoapparat.start();
}

@Override
protected void onStop() {
    super.onStop();
    fotoapparat.stop();
}

@Override
protected void onDestroy() {
    super.onDestroy();
    rubikDetector.releaseResources();
}
```

And you're done!

The `FotoapparatConnector` class is just a utility that configures the `RubikDetector` and `FotoApparat` to talk with each other. It achieves this through with the following classes:
  - `RubikDetectorFrameProcessor` - a subclass of `io.fotoapparat.preview.FrameProcessor` which internally uses a `RubikDetector` instance to find the cube in the camera frames.
  - `RubikDetectorSizeSelectorWrapper` - a subclass of `io.fotoapparat.parameter.selector.SelectorFunction` which allows you to specify the desired resolution at which the processing will take place, and which will also notify the `RubikDetectorFrameProcessor` of the selected resolution.
  - `RubikDetectorResultView` - a custom view which knows how to draw the detection result.
  
The 3 classes mentioned above can be extended to tweak their behavior, and they can be used outside of the `FotoapparatConnector`, if some custom binding logic is needed.

### Additional notes
<b>TL;DR:</b>

When performing the setup with the `FotoapparatConnector` class, the image size and `DrawConfig` you set on your `RubikDetector` instance will be overwritten or ignored. 

To control drawing, use `RubikDetectorResultView` XML attributes, and setters (see the class's javadoc). To control the resolution at which processing takes place, use a `Fotoapparat` `SelectorFunction<Collection<Size>, Size>`.

<b>Long version</b>

The image size will be overwritten by the `RubikDetectorFrameProcessor` class, when it will be notified by the size selector of the selected frame size. 

The `DrawConfig` you set on the `RubikDetector` also has no effect when using the `FotoApparat` library. When using a `DrawConfig`, you're telling the `RubikDetector` how to draw the found facelets in the native code. The results of this drawing will be visible in the "output frame" (see [memory layout](#memory-layout)) created by the `RubikDetector`. 

However, the `io.fotoapparat.view.CameraView` used internally by `RubikDetectorResultView` does not know how to access this "output frame". Because of this, if any drawing ocurrs in the native code, it is ignored by the class rendering the preview.

The solution here is to draw the found facelets in Java, using an overlay view. This is performed by the `RubikDetectorResultView`, which has no access to the `DrawConfig` set on the `RubikDetector`. Instead, the `RubikDetectorResultView`'s XML attributes and setter methods can be used to control how the facelets need to be drawn.

## Memory Layout

When calling `RubikDetector#findCube(...)`, a byte array which contains the input frame is required as a parameter. 

However, the byte array needs to have the capacity returned by `RubikDetector#getRequiredMemory()`. This is necessary because the `byte[]` parameter passed to `RubikDetector#findCube(...)` acts as an `inout` parameter, and will also contain the output frame, after `RubikDetector#findCube(...)` returns.


| <p align="center"> <img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/memory_model.png" align="center" width="800px"/> </p> |
| :---: |
| Contents of the `byte[]` after `findCube(...)` returns. |

To know where to write the input frame in the `byte[]`, use:
  - `RubikDetector#getInputFrameBufferOffset()` -> returns the index in the `byte[]` at which the input frame bytes need to start. 
  - `RubikDetector#getInputFrameByteCount()` -> returns the size of the input frame, in bytes.
  
To know at what offset the output frame is written, after `findCube(...)` has returned, use:
  - `RubikDetector#getResultFrameBufferOffset()` -> returns the index in the `byte[]` at which the output frame bytes start.
  - `RubikDetector#getResultFrameByteCount()` -> returns the size of the output frame, in bytes.
  
`RubikDetector#getRequiredMemory()` returns the total required size of the inout `byte[]`. Passing a `byte[]` of a smaller capacity to `findCube(...)` will result in a crash.

The values returned by the `get...` methods mentioned here are recomputed after every call to `RubikDetector#updateImageProperties(...)`.
    
## RubikDetector as a learning tool/Debug mode

RubikDetector is considered to be in equal measure a learning tool, and a practical library - despite the problem it tries to solve is quite niche.

Because of this, first, it offers a debuggable mode. When enabled, the `RubikDetector` prints to the standard output additional logs which offer insight into its inner workings.

Additionally, when a path to a location with write access is provided to `RubikDetector`, through `RubikDetector.Builder#.imageSavePath(<enter path here)`, <b>and</b> debuggable mode is <b>enabled</b>, then additional internal processing frames will be saved at that location to be inspected later.

Internal frame showing edge detection | Internal frame showing the found rectangles 
:--: | :--: 
<img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/pic_12_detected_edges.jpg"/> | <img src="https://github.com/cjurjiu/RubikDetector-Android/blob/master/images/pic_12_filtered_rectangles.jpg"/>

When debugging is disabled no images will be saved on disk, even if a valid path is provided. When images will be saved on disk, expect to have abysmal performance (below 10 fps), since writing to persistent storage is very slow.

Saving those images has no actual practical purpose, but it offers insight for the curious on the various stages of processing.

Improvement and suggestions on what to additionally save are welcomed.

## How to build

  * [Android library](https://github.com/cjurjiu/RubikDetector-Android/tree/master/RubikDetectorAndroid)
  * [Desktop sample/library](https://github.com/cjurjiu/RubikDetector-Android/tree/master/RubikDetectorLinux)

## Known issues

RubikDetector has a few known issues. You can see them [here](https://github.com/cjurjiu/RubikDetector-Android/issues).

The most annoying issue would be the [one related to color detection](https://github.com/cjurjiu/RubikDetector-Android/issues/2). More specifically, in low light conditions there is a tendency to mistake green facelets for blue facelets, and sometimes orange facelets for red facelets.

Any help would be appreciated.

## Misc

### Support

The image processing algorithm is written in C++ and is fully portable to other platforms (incl. iOS). 

Currently, samples exist for desktop (linux, OSX) and Android. 

The Android library also has a connector to the popular FotoApparat camera library.

### Current state

Stable release v1.0.0.

### Performance

~45-~55 (depending on device) average FPS at 1080p, while processing frames, for RubikDetector. When using the Fotoapparat connector, the framerate is lower.

## Binaries

### RubikDetector Binaries
Binaries and dependency information for Maven, Ivy, Gradle and others can be found on [jcenter](https://bintray.com/cjurjiu/cjurjiu-opensource/rubikdetector-android).

Example for Gradle:

```groovy
compile 'com.catalinjurjiu:rubikdetector-android:1.0.0'
```

and for Maven:

```xml
<dependency>
  <groupId>com.catalinjurjiu</groupId>
  <artifactId>rubikdetector-android</artifactId>
  <version>1.0.0</version>
  <type>pom</type>
</dependency>
```
and for Ivy:

```xml
<dependency org='com.catalinjurjiu' name='rubikdetector-android' rev='1.0.0'>
  <artifact name='rubikdetector-android' ext='pom' ></artifact>
</dependency>
```
### Fotoapparat Connector Binaries

Complete details available on [jcenter](https://bintray.com/cjurjiu/cjurjiu-opensource/rubikdetector-fotoapparat-connector).

Example for Gradle:

```groovy
compile 'com.catalinjurjiu:rubikdetector-fotoapparat-connector:1.0.0'
```

and for Maven:

```xml
<dependency>
  <groupId>com.catalinjurjiu</groupId>
  <artifactId>rubikdetector-fotoapparat-connector</artifactId>
  <version>1.0.0</version>
  <type>pom</type>
</dependency>
```
and for Ivy:

```xml
<dependency org='com.catalinjurjiu' name='rubikdetector-fotoapparat-connector' rev='1.0.0'>
  <artifact name='rubikdetector-fotoapparat-connector' ext='pom' ></artifact>
</dependency>
```
