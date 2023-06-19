# Intel® Library for Video Super Resolution (Intel® Library for VSR) README
This project aims to provide a CPU based implementation of the RAISR (Rapid and Accurate Image Super Resolution) algorithm (https://arxiv.org/pdf/1606.01299.pdf) optimized to achieve beyond real-time performance for 2x upscaling on Intel® Xeon® platforms.  It can take a lower resolution image and upscale 2x (e.g 540p to 1080p) and provides better quality results than standard (bicubic) algorithms and a good performance vs quality trade-off as compared to DL-based algorithms like EDSR.  The Intel Library for VSR is provided as an FFmpeg plugin inside of a Docker container to help ease testing and deployment burdens.  This project is developed using C++ and takes advantage of Intel® Advanced Vector Extension 512 (Intel® AVX-512).

## How to build
Please see "How to build.md" to build via scripts or manually.

## Running the Intel Library for VSR 
One should be able to test with video files:
```
./ffmpeg -y -i /input_files/input.mp4 -vf raisr=threadcount=20 -pix_fmt yuv420p /output_files/out.yuv
```
Or folders of images:
```
./ffmpeg -y -start_number 000 -i '/input_files/img_%03d.png' -vf scale=out_range=full,raisr=threadcount=20 -start_number 000 '/output_files/img_%03d.png'
```
Because saving raw uncompressed (.yuv) video can take up a lot of disk space, one could consider using the lossless (-crf 0) setting in x264/x265 to reduce the output file size by a substantial amount.

**x264 lossless encoding**
```
./ffmpeg -y -i /input_files/input.mp4 -vf raisr=threadcount=20 -pix_fmt yuv420p -c:v libx264 -crf 0 /output_files/out.mp4
```
**x265 lossless encoding**
```
./ffmpeg -y -i /input_files/input.mp4 -vf raisr=threadcount=20 -pix_fmt yuv420p -c:v libx265 -crf 0 /output_files/out_hevc.mp4
```
## Evaluating the Quality of RAISR Super Resolution
Evaluating the quality of the RAISR can be done in different ways.
1. A source video or image can be upscaled by 2x using different filter configurations. We suggest trying these 3 command lines based upon preference:

**Sharpest output**
```
./ffmpeg -i /input_files/input.mp4 -vf "raisr=threadcount=20:passes=2:filterfolder=filters5" -pix_fmt yuv420p /output_files/out.yuv
```
**Fastest Performance ( second pass disabled )**
```
./ffmpeg -i /input_files/input.mp4 -vf "raisr=threadcount=20:filterfolder=filters1" -pix_fmt yuv420p /output_files/out.yuv
```
**Medium sharpness**
```
./ffmpeg -i /input_files/input.mp4 -vf "raisr=threadcount=20:passes=2:filterfolder=filters3" -pix_fmt yuv420p /output_files/out.yuv
```
2. A source video or image can be downscaled by 2x, then passed through the RAISR filter which upscales by 2x
```
./ffmpeg -y -i /input_files/input.mp4 -vf scale=iw/2:ih/2,raisr=threadcount=20 -pix_fmt yuv420p /output_files/out.yuv
```
At this point the source content is the same resolution as the output and the two can be compared to understand how well the super resolution is working.  RAISR can be compared against existing DL super resolution algorithms as well.  It is recommended to enable second pass in Intel Library for VSR to produce sharper images.  Please see the Advanced Usage section for guidance on enabling second pass as a feature.

**OpenCL acceleration**
```
./ffmpeg -y -i /input_files/input.mp4 -vf raisr=asm=opencl -pix_fmt yuv420p /output_files/out.yuv
```
or user can use filter "raisr_opencl" to build full gpu pipeline. \
[ffmpeg-qsv](https://trac.ffmpeg.org/wiki/Hardware/QuickSync) \
[ffmpeg-vaapi](https://trac.ffmpeg.org/wiki/Hardware/VAAPI)
```
ffmpeg -init_hw_device vaapi=va -init_hw_device qsv=qs@va -init_hw_device opencl=ocl@va -hwaccel qsv -c:v h264_qsv -i input.264 -vf "hwmap=derive_device=opencl,format=opencl,raisr_opencl,hwmap=derive_device=qsv:reverse=1:extra_hw_frames=16" -c:v hevc_qsv output.mp4
```
```
ffmpeg -init_hw_device vaapi=va -init_hw_device opencl=ocl@va -hwaccel vaapi -hwaccel_output_format vaapi -i input.264 -vf "hwmap=derive_device=opencl,format=opencl,raisr_opencl,hwmap=derive_device=vaapi:reverse=1:extra_hw_frames=16" -c:v hevc_vaapi output.mp4
```


## To see help on the RAISR filter
`./ffmpeg -h filter=raisr`

    raisr AVOptions:
      ratio             <int>        ..FV....... ratio (currently only ratio of 2 is supported) (from 2 to 2) (default 2)
      bits              <int>        ..FV....... bit depth (from 8 to 10) (default 8)
      range             <string>     ..FV....... color range of the input. If you are working with images, you 
                                                 may want to set range to full (video/full) (default video)
      threadcount       <int>        ..FV....... thread count (from 1 to 120) (default 1)
      filterfolder      <string>     ..FV....... absolute filter folder path (default "filters1")
      blending          <int>        ..FV....... CT blending mode (1: Randomness, 2: CountOfBitsChanged) (from 1 to 2) (default 2)
      passes            <int>        ..FV....... passes to run (1: one pass, 2: two pass) (from 1 to 2) (default 1)
      mode              <int>        ..FV....... mode for two pass (1: upscale in 1st pass, 2: upscale in 2nd
                                                 pass) (from 1 to 2) (default 1)
      asm               <string>     ..FV....... x86 asm type: (avx512 or avx2) (default "avx512")

## Advanced Usage ( through Exposed Parameters )
The FFmpeg plugin for Intel Library for VSR exposes a number of parameters that can be changed for advanced customization
### threadcount
Allowable values (1,120), default (20)

Changes the number of software threads used in the algorithm.  Values 1..120 will operate on segments of an image such that efficient threading can greatly increase the performance of the upscale.  The value itself is the number of threads allocated.
### filterfolder
Allowable values: (Any folder path containing the 4 required filter files: Qfactor_cohbin_2_8/10, Qfactor_strbin_2_8/10, filterbin_2_8/10, config), default (“filters1”)

Changing the way RAISR is trained (using different parameters and datasets) can alter the way RAISR's ML-based algorithms do upscale.  For the current release, 5 filters have been provided: filters1, filters2, filters3, filters4 and filters5.  Each filter has been generated with varying amounts of complexity and sharpness. For filters4 the 1st and 2nd pass filters both trained on sharpened HR as reference, for other filters only 2nd pass filters trained on shaprpened HR as reference
and you can find the training informantion in filternotes.txt of each filters folder.

If doing a single pass ( passes=1 ), it is suggested to use filters1 or filters4.  With passes=2 the 5 filters are described as:
| filter name | filter complexity | sharpness |
| ----------- | ----------------- | ----------- |
| filters1    | 864x2             | high |
| filters2    | 864x2             | low |
| filters3    | 864x2             | medium |
| filters4    | 864x2             | high |
| filters5    | 32000x2           | high |

Please see the examples under the "Evaluating the Quality" section above where we suggest 3 command lines based upon preference.
Note that for second pass to work, the filter folder must contain 3 additional files: Qfactor_cohbin_2_8/10_2, Qfactor_strbin_2_8/10_2, filterbin_2_8/10_2
### bits
Allowable values (8: 8-bit depth, 10: 10-bit depth), default (8)

The library supports 8 and 10-bit depth input. Use HEVC encoder to encoder yuv420p10le format.
```
./ffmpeg -y -i [10bits video clip] -vf "raisr=threadcount=20:bits=10" -c:v libx265 -preset medium -crf 28 -pix_fmt yuv420p10le output_10bit.mp4
```
### range
Allowable values (video: video range, full: full range), default (video)

The library caps color within video/full range.
```
./ffmpeg -y -i [image/video file] -vf "raisr=threadcount=20:range=full" outputfile
```
### blending
Allowable values (1: Randomness, 2: CountOfBitsChanged), default (2 )

The library holds two different functions which blend the initial (cheap) upscaled image with the RAISR filtered image.  This can be a means of removing any aggressive or outlying artifacts that get introduced by the filtered image.
### passes
Allowable values (1,2), default(1)

`passes=2` enables a second pass.  Adding a second pass can further enhance the output image quality, but doubles the time to upscale.  Note that for second pass to work, the filter folder must contain 3 additional files: Qfactor_cohbin_2_8/10_2, Qfactor_strbin_2_8/10_2, filterbin_2_8/10_2
### mode
Allowable values (1,2), default(1).  Requires flag passes=2”

Dictates which pass the upscaling should occur in.  Some filters have the best results when it is applied on a high resolution image that was upscaled during a first pass by using mode=1.  Alternatively, the Intel Library for VSR can apply filters on low resolution images during the first pass THEN upscale the image in the second pass if mode=2, for a different outcome.
```
./ffmpeg -i /input_files/input.mp4 -vf "raisr=threadcount=20:passes=2:mode=2" -pix_fmt yuv420p /output_files/out.yuv
```
### asm
Allowable values ("avx512","avx2","opencl"), default("avx512")

The VSR Library requires an x86 processor which has the Advanced Vector Extensions 2 (AVX2) available.  AVX2 was first introduced into the Intel Xeon roadmap with Haswell in 2015.  Performance can be further increased if the newer AVX-512 Foundation and Vector Length Extensions are available.  AVX512 was introduced into the Xeon Scalable Processors (Skylake gen) in 2017. The VSR Library will always check for AVX512 first, and fallback to AVX2 in its absense.  However if the use case requires it, this asm parameter allows the default behavior to be changed. User can also choose opencl if the opencl is supported in their system.

# How to Contribute
We welcome community contributions to the Open Visual Cloud repositories. If you have any idea how to improve the project, please share it with us.

## Contribution process
Make sure you can build the project and run tests with your patch.
Submit a pull request at https://github.com/OpenVisualCloud/Video-Super-Resolution-Library/pulls.
The Intel Library for VSR is licensed under the BSD 3-Clause "New" or "Revised" license. By contributing to the project, you agree to the license and copyright terms therein and release your contribution under these terms.

## How to Report Bugs and Provide Feedback
Use the Issues tab on Github.

Intel, the Intel logo and Xeon are trademarks of Intel Corporation or its subsidiaries.
