

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Step 1: Computing the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Step 2: Applying the distortion correction from the step 1 to raw images
* Step 1: Using gray-scale transforms and gradients with a set of thresholdeds to create binary image.
* Step 2: Applying a perspective transform to rectify binary image ("birds-eye view")
* Step 3: Detecting lane pixels and fit to find the lane boundary using histogram information and sliding widnows
* Step 4: Calculating the curvature of the lane and vehicle position with respect to center.
* Step 5: Warp the detected lane boundaries back onto the original image.
* Step 6: Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

Goals:
* Developing a robust pipeline to detect laneline on road tracks
* Detecting outliers and how they affect the lines

## Steps Explained

### 1- Camera Calibration

In order to prepare input images for line detection, first it needs to be undistorted. I used the provided chessboard images in folder `camera-cal` of this project to collect detected coreners of gray-scaled chessboard images using opencv method `cv2.findChessboardCorners`. For internal corners I used 9x6 grid and prepared a default object-point with the same size, which wil be appended to list of object-points everytime all corners of a chesshboard image is detected, corners will be appended to image-points accordingly. 
Collected points are then passed to `cv2.calibrateCamera()` to get camera-matrix and distortion-coefficients in order to undistort images using `cv2.undistort()`, here are some of the successfully detected corners:
<table style="width:100%">
  <tr>
    <td><img src="./document/undist1.png" width="250" height="200"/></td>
    <td><img src="./document/undist02.png" width="200" height="200"/></td>
    <td><img src="./document/undist03.png" width="200" height="200"/></td>
  </tr>
</table>


Note: 9x6 corners failed on finding corners for the following images:

* camera_cal/calibration1.jpg
* camera_cal/calibration4.jpg
* camera_cal/calibration5.jpg

I used the first image to test my undistortion method, and here is the result
<table style="width:100%">
  <tr>
    <td><img src="./document/undist1-res.png" height="200"/></td>
  </tr>
</table>

### 2- Binarization

I used a combination of 2 transformations:

 * 1- HLS Thresholding: raw image is converted to HLS channel to only keep channel `s` and I applied a threshold of [90,200]  as it does a fairly robust job of picking up the lines under very different color and contrast conditions.
 * 2- Gradient Thresholding: raw image is converted to gray-scale and I used sobel operator in `x` orientations to find image emphasising edges vertically, threshold applied in this case [90,200].
 
 output of these 2 are then combined into 1 set of points:
 
 <table style="width:100%">
  <tr>
    <td>Original</td>
    <td>Sobel X</td>
    <td>HLS (S)</td>
    <td>Combined</td>
    
  </tr>
  <tr>
    <td><img src="./document/combined-1.png" width="450" height="200"/></td>
    <td><img src="./document/combined-2.png" width="450" height="200"/></td>
    <td><img src="./document/combined-3.png" width="450" height="200"/></td>
    <td><img src="./document/combined-4.png" width="450" height="200"/></td>
  </tr>
</table>
 

### 3- Perspective Transform

I selected four source points for perspective transform to get an approximation as a region where lanes tend to appear.With perspective transform this region is trasnformed to bird's eye view perspective where lines look parallel and vertical.
 <table style="width:100%">
  <tr>
    <td>Original</td>
    <td>Region</td>
    <td>Binarized</td>
    <td>Perspective Transform</td>
  </tr>
  <tr>
    <td><img src="./document/combined-1.png" width="450" height="200"/></td>
    <td><img src="./document/region.png" width="450" height="200"/></td>
    <td><img src="./document/combined-5.png" width="450" height="200"/></td>
    <td><img src="./document/warped.png" width="450" height="200"/></td>
  </tr>
</table>

### 4- Finding fits for the lines

After bringing the most important region of the image into focus it's time to decide explicitly which pixels are part of the lines and which belong to the left line and which belong to the right line.

For this step, I draw histogram of the lower half of the warped image as it has the starting pixels of the lines. Histogram would display the two most prominent peaks with spikes in x-axis and these will be of the base of the lane lines. From that point, I use a sliding window, placed around the line centers, to find and follow the lines up to the top of the frame.
The strategy to use base-lines is to divide image into multiple windows from y-axis and process each slide from bottom to the top to find good pixels and adjust the lines for the next slide.
Every time 50 pixels are detected in a slide, base-lines are moved to the average point of these pixels to be able to capture curves more precisley. All the good points are collected as a list and passed to polyfit to generate coefficents for drawing left & right lines.

 <table style="width:100%">
  <tr>
    <td>Histogram</td>
    <td>Sliding Windows</td>
  </tr>
  <tr>
    <td><img src="./document/histograp.png" width="550" height="200"/></td>
    <td><img src="./document/sliding.png" width="550" height="200"/></td>
  </tr>
</table>

### 5- Draw lines on original image

The final step is projecting the measurement back down onto the image. I used the inversed prespective-transform Matrix that was generated in step 3 to trasform the line-fits from the previous step to the original view.
 
<table style="width:100%">
  <tr>
    <td><img src="./document/unwarped.png" width="550" height="200"/></td>
  </tr>
</table>
 
### Pipeline (single images)

As mentioned in previous section, each frame of image goes thorugh 5 main steps:
first, image is undistorted using callibration matrix and distortion coefficients from chessboard images. The image is then binarized through combination of gradient sobelx and saturation channel in HLS. To get perspective transform, I marked a region on image where lanes usually appear and warped them to bird's eye view usign `offset=200`:

| Source       | Destination   | 
|:------------:|:-------------:| 
| 520,520      | 320, 0        | 
| 830,520      | 320, 720      |
| 1150,690     | 960, 720      |
| 300,690      | 960, 0        |

```destination_vertices: [[offset,offset],[img_size[1]-offset,offset],[img_size[1]-offset,img_size[0]-offset],[offset,img_size[0]-offset]]```

Next, warped-image is passed to my module `sliding_window_histogram` to find line-fits, as explained earlier I split the warped-image into halves, and draw histogram of the lower half becuase it has the starting pixels of the lines. Histogram contains information about the two most prominent peaks with spikes in x-axis. From that point, I use a sliding window, placed around the line centers, to find and follow the lines up to the top of the frame.
I divide the warped-image into 9 windows from y-axis and start processing slides from the bottom to the top to find good pixels and adjust the center-lines for the next slide.
With pixel-threshold of 50 pixels, I decide to whether adjust my center-line based on new findings or not. If the condition met in a slide, center-lines are moved to the average point of these pixels which enables me to capture curves more precisley. All the good points are collected as a list and passed to polyfit to generate coefficents for drawing left & right lines.

After the fits are calculate, I create 2 Line() instances for the left & the right lines. Line class holds some feature information for doing frame correction in processing a sequence of images, so I keep all the good lines in a list.
Some of the main featres I keep my eyes on are:
    * `line_base_pos` : the position of the line relative to the center of the image; to check the position of the line with previous frame within an offset to avoid jumps
    * `radius_of_curvature`: curve of the line; to check the cruve of the line with previous frame to avoid jumps
    * `current_fit`: line coefficients; to keep track of current frame for next new frames in case I need to adjust the new frame
    * `allx` & `ally`: all x and y points of the line; to keep track of current fit to do further adjustments for new frames if necessary

As frames populate these class variables, I check for position to be in `1(meter)` offset from previous frame, then based on the previous frame coefficients I calculate an expected slope value with an offset of `0.15`. lastly I compare current curve with the previous frame curve with offset of`50`. If all conditions are met, frame is a good match and will be appended to my list of lines instances.

If a line couldn't meet the criteria, I'll copy the previous-frame coefficients,allx & ally to recalculate the fit & curves and slopes.
Finally I cast the lines onto the original frame and move on to the next frame.

Here's a full flow of original image through the entire pipeline:

<table style="width:100%">
  <tr>
    <td>Original</td>
    <td>Undistorted</td>
    <td>Gradient Sobel x Transform</td>
    <td>HLS (S) Transform</td>
    <td>Combined HLS & Sobelx</td>
  </tr>
  <tr>
    <td><img src="./document/fram1-original.png" width="250" height="200"/></td>
    <td><img src="./document/fram1-undistorted.png" width="250" height="200"/></td>
    <td><img src="./document/fram1-sobel.png" width="250" height="200"/></td>
    <td><img src="./document/fram1-hls.png" width="250" height="200"/></td>
    <td><img src="./document/fram1-hlsandsobel.png" width="250" height="200"/></td>
  </tr>
  <tr>
    <td>Region</td>
    <td>Warped</td>
    <td>Histogram</td>
    <td>Sliding Windows</td>
    <td>output</td>
  </tr>
  <tr>
    <td><img src="./document/fram1-region.png" width="250" height="200"/></td>
    <td><img src="./document/fram1-warped.png" width="250" height="200"/></td>
    <td><img src="./document/fram1-histogram.png" width="250" height="200"/></td>
    <td><img src="./document/fram1-sliding-widnows.png" width="250" height="200"/></td>
    <td><img src="./document/fram1output.png" width="250" height="200"/></td>
  </tr>
</table>


![alt text][image3]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.


I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

###Pipeline (video)

####1. Final video output. 

Here's a [link to my video result](./project_video.mp4)

Challenge video [](./project_video.mp4)

Harder Challenge [](./project_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

