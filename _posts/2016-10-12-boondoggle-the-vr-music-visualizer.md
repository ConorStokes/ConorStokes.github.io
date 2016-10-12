---
layout: post
title: "Boondoggle, the VR Music Visualizer"
description: ""
category: 
tags: []
---
{% include JB/setup %}
When I got my Oculus in June, one of the things that struck me was that the experiences that I enjoyed most were those where you could relax and just enjoy being somewhere else. It reminded me a lot of one of my favourite ways of relaxing; putting on headphones and losing myself in some music. I'd also been looking at [shadertoy](https://www.shadertoy.com/) and many cool demos coming through on my twitter feed and wanted to have a bit of a try of some of the neat ray marching techniques.

So the basic idea that followed from this was to create a music visualization framework that could capture audio currently playing on the desktop (as to work with any music player that wasn't running a DRM stream), do a bit of signal processing on it and then output pretty ray marched visuals on the HMD. 

Here's a capture of a visualizer effect running in a Window:

<iframe width="560" height="315" src="https://www.youtube.com/embed/nSGt-b3IKPM" frameborder="0" allowfullscreen></iframe>

<br/>
And here's one in the HMD rendering view: 

<iframe width="560" height="315" src="https://www.youtube.com/embed/RNpDMeUymUY" frameborder="0" allowfullscreen></iframe>
<br/>

This post basically goes through how I went about that, as well as the ins and outs of one visualizer effect I'm pretty happy with. I should be clear though, this is a work in progress and a spare time project that has often been left for weeks at a time (and it's in a definite state of flux). The current source is on [github](https://github.com/ConorStokes/boondoggle) and will be updated sporadically as I experiment. There is also a [binary release](https://github.com/ConorStokes/boondoggle/releases/tag/blog-version). There is a lot to be covered here, so in some sections I will be glossing over things (feel free to email me, leave a comment or contact me on twitter if there is any detail you would like clarified).

## Basic Architecture ##
The visualizer framework has two main components; the visualizer run-time itself, and a compiler that takes a JSON file that references a bundle a bunch of HLSL shaders and textures and turns them into visualizer effects package used by the run-time. The compiler turn around time is pretty quick, so you can see effects changes quite quickly (iteration time isn't quite shadertoy good, but still pretty reasonable). The targeted platforms are Windows, D3D 11, and the Oculus SDK (if you don't have the Oculus runtime installed or a HMD connected, it will render into a window, but the target is definitely a HMD; the future plan is to add OpenVR SDK support too).

I used Branimir Karadzic's [GENie](https://github.com/bkaradzic/GENie) to generate Visual Studio projects (currently the only target I support), because it was a bit easier than building and maintaining the projects by hand. Apart from the Oculus SDK and platform dependencies, all external code has been copied into the repository and is compiled directly as part of the build. Microsoft's DDS loader from [DirectXTex](https://github.com/Microsoft/DirectXTex) is used to load textures, [KISS FFT](https://github.com/itdaniher/kissfft) is used for signal processing and JSON is parsed with Neil Henning's [json.h](https://github.com/sheredom/json.h). 

Originally I was going to capture and process the audio on a separate thread, but given how light the CPU requirements for the audio capturing/processing are and that we aren't spending much CPU time on anything else, everything ended up done on a single thread, with the audio being polled every frame.

## Handling the Audio ##
Capturing audio from the desktop on Windows is interesting. The capture itself is done using WASAPI with a loop-back capture client running on the default device. This pretty much allows us to capture whatever is rendering on a device that isn't exclusive or DRM'd (and we make sure the data is coerced to floating point if it isn't already).

One of the caveats to this is that capture doesn't work when nothing is playing on the device, so we also create a non-exclusive render device and play silence on it at the same time.

Every frame we poll to A) play silence on the device if there is room in the playback buffer and B) capture any sound data packets currently to be played to the device by copying them into a circular buffer per channel (we do this as we only copy out the first two channels of the audio in case the device is actually surround sound, mono is handled by duplication). This works very well as long as you maintain a high enough frame-rate to capture every packet coming out of the device; the case where a packet is missed is handled by starting from scratch in the circular buffer.

Audio processing is tied to the first power-of-two number of samples that allow us to capture our minimum frequency for processing (50hz). The circular buffer is twice this size and instead of using the head of the buffer to do the audio processing every frame, we use either the low or high halves of the circular buffer whenever they are full.

When we capture one of those roughly-50hz shaped blocks, we do a number of processing steps to get the data ready to pass off to the shaders:

 - Calculate the [RMS](https://en.wikipedia.org/wiki/Root_mean_square) on all the samples in the block per channel to get an estimate of how loud that channel is. Also a logarithmic version of this cut off at a noise floor to get a more human friendly measure. These values are smoothed using an exponential moving average to make them more visually appealing in the shaders.
 - Run a Windowing function ([Hanning](https://en.wikipedia.org/wiki/Hann_function), but this can be changed to any other Generalized Hamming Window relatively easily) prior to the FFT to take out any spurious frequency data that would come from using a finite/non-wrapping FFT window. Note, we don't do overlapping FFT windows as this works fine for visualization purposes.
 - Run a real FFT per channel using KISS FFT. 
 - Divide the frequency bins into 16 buckets based on a modified version of the formula [here](http://dlbeer.co.nz/articles/fftvis.html) and work out the maximum magnitude of any frequency bin in the bucket, convert it to a log scale, cut off at the noise floor and smooth it using an exponential moving average with previous values. This works a lot better in the shader than putting the whole FFT into a texture and dealing with it.
 - Copy the audio channel data and the amplitude of each frequency bin in the FFT into a small texture. Note, this uses a fairly slow path for copying, but the texture is tiny so it has very little impact.

## Oculus Integration ##
I was very impressed with the Oculus SDK and just how simple it was to integrate. The SDK allows you to create texture swap chains that you can render to in D3D and then pass off to a compositor for display (as well as getting back a mirror texture to display in a Window) and this is very simple and integrates well.

In terms of geometry, you get the tan of all the half angles for an [off-axis frustum](http://paulbourke.net/papers/HET409_2004/het409.pdf) per eye, as well as the poses per eye as a quaternion orientation and position. Because we're ray-marching, we aren't going to use a traditional projection matrix, but instead some values that will make it easy to create a ray direction from screen coordinates:

    // Create rotation adjusted for handedness etc
    XMVECTOR rotation = XMVectorSet( -eyePose.Orientation.x, 
                                     -eyePose.Orientation.y, 
                                     eyePose.Orientation.z, 
                                     eyePose.Orientation.w );
                                     
    // Upper left corner of the view frustum at z of 1. Note, z positive, left handed. 
    XMVECTOR rayScreenUpperLeft = XMVector3Rotate( XMVectorSet( -fov.LeftTan, 
                                                                fov.UpTan, 
                                                                1.0f, 
                                                                0.0f ), 

    // Right direction scaled to width of the frustum at z of 1.
    XMVECTOR rayScreenRight = XMVector3Rotate( XMVectorSet( fov.LeftTan + fov.RightTan,
                                                            0.0f, 
                                                            0.0f, 
                                                            0.0f ), 
                                               rotation );
                                               
    // Down direction scaled to height of the frustum at z of 1.
    XMVECTOR rayScreenDown = XMVector3Rotate( XMVectorSet( 0.0f, 
                                                           -( fov.DownTan + fov.UpTan ),
                                                           0.0f, 
                                                           0.0f ), 
                                              rotation );

In the shader, we'll use the eye position we get from the Oculus SDK as the ray origin and use these values to create a ray direction. Oculus use meters as their unit, so I decided to stick with that.

Oculus also takes care of the GPU sync, time warp (if needed, when you begin your frame and get the HMD position, you also get a time value that you then feed in when you submit the frame), distortion and chromatic aberration correction.

## File Format ##
At the content build step, I package everything into a single file that gets memory mapped at run-time. I use the same relative addressing trick used by Per Vognsen's [GOB](https://gist.github.com/pervognsen/c25a039fcf8c256141ef0778a1b32a88), with a bit of C++ syntax sugar. The actual structures in the file are traversed in-place both at load time (to load the required resources into D3D) and per frame.

## Rendering ##
Per frame, we basically get an effect id. Given that effect id, we look-up the corresponding shader and set any corresponding textures/samplers (including potentially the sound texture); all of this information comes directly out of the memory mapped file and indexes corresponding arrays of shaders/textures. 

While I support off-screen procedural textures that can be generated either at load time or every frame, I haven't actually used this feature yet, so it's a not tested. Currently these use a specified size, but at some stage I plan to make these correspond to the HMD resolution.

Each package has a single vertex quad shader that is expected to generate a full-screen triangle without any input vertex or index data (it creates the coordinates directly from the ids). We never load any mesh data at all into index or vertex buffers for rendering, as all geometry will be ray-marched directly in the pixel shader.

Another thing on the to-do list is to support variable resolution rendering to avoid missed frames and allow super-sampling on higher-end hardware. I currently have the minimum required card for the Rift (a R9 290), so the effects I have so far are targeted to hit 90hz with some small margin of safety at the Rift's default render resolution on that hardware (I've also tested outside VR on a nvidia GTX 970M).

## The Visualizer Effect in the Video ##
The visualizer effect in the video is derived from my first test-bed effect (in the source code), but was my first attempt at focusing on creating something aesthetically pleasing as opposed to just playing around (my name for the effect is "glowing blobby bars", by the way). It's based on the very simple frequency-bars visualization you see in pretty much every music player.

To get a grounding for this, I recommend reading [Inigo Quilez article on ray marching with distance functions](http://iquilezles.org/www/articles/distfunctions/distfunctions.htm) and looking at [Mercury's SDF library](http://mercury.sexy/hg_sdf/).

Each bar's height comes from one of the 16 frequency buckets we created in the audio step, with the bars to either side coming from the respective stereo channel. 

To ray march this, we create a signed distance function based on a rounded box and we use a modulus operation to repeat it, which creates a grid (where we expect that if a point is within a grid box, it is closest to the contents of the grid box than other those in other grid boxes) . We can also get the current grid coordinates by doing an integer division, which is what we do to select the frequency bucket to get the bar's height and material (and also use as the material id). 

There is a slight caveat to this in that sometimes the height difference between two bars is such that a bar in another grid slot may actually be closer than the bar in the current grid slot. To avoid this, we simply space the bars conservatively relative to their maximum possible height difference. The height is actually tied to the cubed log-amplitude of the frequency bucket to make the movement a bit more dramatic.

To create the blobby effect, we actually do two things. Firstly, we twist the bars using a 2D rotation based on the current y coordinate, which adds a bit more visually interesting detail, especially on tall bars. We also use a displacement function based on overlapping sine waves of different frequencies and phases tied to the current position and time, with an amplitude tied to the different frequency bins. Here's the code for the displacement below:

	float scale = 0.25f / NoiseFloorDbSPL;

	// Choose the left stereo channel
    if ( from.x < 0 )
    {
        lowFrequency = 1 - ( SoundFrequencyBuckets[ 0 ].x + 
                             SoundFrequencyBuckets[ 1 ].x + 
                             SoundFrequencyBuckets[ 2 ].x + 
                             SoundFrequencyBuckets[ 3 ].x ) * scale;
        midLowFrequency = 1 - ( SoundFrequencyBuckets[ 4 ].x + 
                                SoundFrequencyBuckets[ 5 ].x + 
                                SoundFrequencyBuckets[ 6 ].x + 
                                SoundFrequencyBuckets[ 7 ].x ) * scale;
        midHighFrequency = 1 - ( SoundFrequencyBuckets[ 8 ].x + 
                                 SoundFrequencyBuckets[ 9 ].x + 
                                 SoundFrequencyBuckets[ 10 ].x + 
                                 SoundFrequencyBuckets[ 11 ].x ) * scale;
        highFrequency = 1 - ( SoundFrequencyBuckets[ 12 ].x + 
                              SoundFrequencyBuckets[ 13 ].y + 
                              SoundFrequencyBuckets[ 14 ].x + 
                              SoundFrequencyBuckets[ 15 ].x ) * scale;
    }
    else // right stereo channel
    {
        lowFrequency = 1 - ( SoundFrequencyBuckets[ 0 ].y + 
                             SoundFrequencyBuckets[ 1 ].y + 
                             SoundFrequencyBuckets[ 2 ].y + 
                             SoundFrequencyBuckets[ 3 ].y ) * scale;
        midLowFrequency = 1 - ( SoundFrequencyBuckets[ 4 ].y + 
                                SoundFrequencyBuckets[ 5 ].y + 
                                SoundFrequencyBuckets[ 6 ].y + 
                                SoundFrequencyBuckets[ 7 ].y ) * scale;
        midHighFrequency = 1 - ( SoundFrequencyBuckets[ 8 ].y + 
                                 SoundFrequencyBuckets[ 9 ].y + 
                                 SoundFrequencyBuckets[ 10 ].y + 
                                 SoundFrequencyBuckets[ 11 ].y ) * scale;
        highFrequency = 1 - ( SoundFrequencyBuckets[ 12 ].y + 
                              SoundFrequencyBuckets[ 13 ].y + 
                              SoundFrequencyBuckets[ 14 ].y + 
                              SoundFrequencyBuckets[ 15 ].y ) * scale;
    }
    
    float distortion = lowFrequency * lowFrequency * 0.8 * 
                            sin( 1.24 * from.x + Time ) *             
                            sin( 1.25 * from.y + Time * 0.9f ) * 
                            sin( 1.26 * from.z + Time * 1.1f ) +
                       midLowFrequency * midLowFrequency * midLowFrequency * 0.4 * 
                           sin( 3.0 * from.x + Time * 1.5 ) * 
                           sin( 3.1 * from.y + Time * 1.3f ) * 
                           sin( 3.2 * from.z + -Time * 1.6f ) +
                       pow( midHighFrequency, 4.0 ) * 0.5 *
                           sin( 5.71 * from.x + Time * 2.5 ) * 
                           sin( 5.72 * from.y + -Time * 2.3f ) * 
                           sin( 5.73 * from.z + Time * 2.6f ) +
                       pow( highFrequency, 5.0 ) * 0.7 * 
                           sin( 9.21 * from.x + -Time * 4.5 ) * 
                           sin( 9.22 * from.y + Time * 4.3f ) * 
                           sin( 9.23 * from.z + Time * 4.6f ) * 
                       ( from.x < 0 ? -1 : 1 );

There are a few issues with this displacement; first it's not a proper distance bound so sphere tracing can sometimes fail here. Edward Kmett pointed out to me that you can make it work by reducing the distance you move along the ray relative to the maximum derivative of the displacement function, but in practice that ends up too expensive in this case, so we live with the odd artifact where sometimes a ray will teleport through a thin blob.

Due to this problem with the distance bound, we can't use the relaxed sphere tracing in the [enhanced sphere tracing](http://lgdv.cs.fau.de/get/2234) paper, but we do make our minimum distance cut-off relative to the radius of the pixel along the ray. Because the distance function is light enough here, we don't actually limit the maximum number of iterations we use in the sphere tracing (the pixel radius cut-off actually puts a reasonable bound on this anyway with our relatively short view distance of 150 meters).

Also, the displacement function is relatively heavy, but luckily we're only computing it (and the entire distance function) for only the bar within the current grid area.

Another issue is that with enough distance from the origin, or a large enough time, things will start to break down numerically. In practice, this hasn't been a problem, even with the effect running for over an hour. We also reset the time every time the effect changes (currently changing is manual, but I plan to add a timer and transitions).

Lighting is from a directional light which shifts slowly with time. There is a fairly standard GGX BRDF that I use for shading, but in this scene the actual colour of the bars is mostly ambient (to try and give the impression that the bars are emissive/glowing). The material colours/physically based rendering attributes come from constant arrays indexed by the material index of the closest object returned by the distance function. 

For the fog, the first trick we use is from Inigo's [better fog](http://iquilezles.org/www/articles/fog/fog.htm), where we factor the light into the fog (I don't use the height in the fog here). The colour of the fog is tied to the sound RMS loudness we calculated earlier, as well as a time factor:

    float3 soundColor = float3( 0.43 + 0.3 * cos( Time * 0.1 ), 
                                0.45 + 0.3 * sin( 2.0 * M_PI * SoundRMS.x ), 
                                0.47 + 0.3 * cos( 2.0 * M_PI * SoundRMS.y ) );

This gives us a nice movement through a variety of colour ranges as the loudness of the music changes. Because of the exponential moving average smoothing we apply during the audio processing, the changes aren't too aggressive (but be aware, this is is still relatively rapid flashing light).

The glow on the bars isn't a post process and is calculated from the distance function while ray marching. To do this, we take the size of the movement along the previous ray and the ambient colour of the closest bar and sum like this every ray step:

        glow += previousDistance * Ambient[ ( uint )distance.y ] / ( 1 + distance.x * distance.x ); 

At the end we divide by the maximum ray distance and multiply by a constant factor to put the glow into a reasonable range. Then we add it on top of the final colour at the end. The glow gives the impression of the bars being emissive, but also enhances the fog effect quite a bit.

One optimisation we do to the ray marching is to terminate rays early if they go beyond the maximum or minimum height of the bars (plus some slop distance for displacement). Past this range the glow contribution also tends to be quite small, so it hasn't been a problem with the marching calculations for the glow.

## Shameless Plug ##
If you like anything here and need some work done, I'm available for freelance work via my business [Burning Candle Software](http://burningcandle.io/)! 