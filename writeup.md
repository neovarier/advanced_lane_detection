##Writeup Template
###You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./submit/chess.jpg "Chessboard"
[image2]: ./submit/undistort_chess.jpg "Undistorted Chessboard"
[image3]: ./submit/original.jpg "Original Road"
[image4]: ./submit/undistort_output.jpg "Road Transformed"
[image5]: ./submit/original_road.jpg "Original"
[image6]: ./submit/S_Channel.jpg "S_Channel Thresholded"
[image7]: ./submit/R_Channel.jpg "R_Channel Thresholded"
[image8]: ./submit/original_road_2.jpg "Original"
[image9]: ./submit/S_Channel_2.jpg "S_Channel Thresholded"
[image10]: ./submit/R_Channel_2.jpg "R_Channel Thresholded"
[image11]: ./submit/SR_1.jpg "SR_Channel Thresholded"
[image12]: ./submit/SR_2.jpg "SR_Channel Thresholded"
[image13]: ./submit/orginal_road_x.jpg "Original"
[image14]: ./submit/gradient_x.jpg "Gradient X Thresholded"
[image15]: ./submit/SR_Channel.jpg "SR Channel Threshold"
[image16]: ./submit/combined.jpg "Combined"
[image17]: ./submit/warped.png "Warp Example"
[image18]: ./submit/histogram.png "Histogram"
[image19]: ./submit/warped_lines.png "Detected Lanes"
[image20]: ./submit/unwarped.png "Unwarped"
[video1]: ./out_project_final.mp4 "Video"
[video2]: ./out_project_final_smooth.mp4 "Smooth Video"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
My project contains:
1. lanedetect.ipynb
2. Ouput project video
3. Smoothened Output project video
4. Images for illustration in submit folder 

The example code in the lessons were helpful for reference.

###Camera Calibration

####1. I used more than 1 chessboard pattern images for camera calibration.
First I prepared the "object points", which are the real world 3-d coordinates (x,y,z) of the chessboard pattern.
Assuming that the chess board is lying in a plane z = 0, the real (x,y)  coordinates chessboard corners are fixed as per 9x6 chessbaord pattern using numpy.mgrid. in objp.
The same objp is used for calibration with all the captured chessboard pattern.
The same objp is appended in objpoints for each captured chessboard pattern.
The "image points" are the (X,Y) co-ordinates of the chessboard corners projected on different image planes. 
The image corner coordinates were detected using cv2.findChessboardCorners gray scale images of the chessboard patterns.
Using cv2.callibrateCamera, the camera matrix and the distortion coefficients were calculated.
Using cv2.undistort() and the camera matrix & distortion coefficient, I undistorted one of the chess board pattterns 

Distorted 
![alt text][image1] 

Undistorted
![alt text][image2]

The code is present in Cell 1 and 2
The top and bottom edges in the chessboard pattern are slightly curved which are getting corrected after undistorting the image

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
First step in the pipeline is to undistort the images of the road using the camera calibration matrix and distortion coefficients

Distorted Road
![alt text][image3] 

Undistorted Road
![alt text][image4]

####2. I tried various color space thresholding using cv2 color space conversion. I narrowed down to use the following color space channels to detect the lanes:

* S-Channel from HSL color space
* R-Channel from RGB color space

S-channel with thresholding is good at detecting bright yellow and white lanes.
But it also detects shadows. The code is rgb_r_select in Cell 5.

Original Image
![alt text][image5] 

S_Channel
![alt text][image6]

Original Image
![alt text][image8] 

S_Channel
![alt text][image9]

R-channel with threshold is good at detecting yellow and white lanes. It avoids detecting shadows as they are darker. 
But it detects road segments which are light gray (close to white). The code is  hls_s_select in Cell 7.

Original Image
![alt text][image5] 

R_Channel
![alt text][image7]

Original Image
![alt text][image8] 

R_Channel
![alt text][image10]

Combining the two channels by Bitwise AND operation would compliment each others negative effects

Original Image
![alt text][image5] 

S&R Channel
![alt text][image11]

Original Image
![alt text][image8] 

S&R Channel
![alt text][image12]

Besides color space, sobel operator for detecting lane edges when color thresholding does not work.
Using sobel operator for high x-gradient gives us edges which are close to vertical.
This aligns with the lane orientation that we need. The function is abs_sobel_thresh in Cell 4.

Original Image
![alt text][image13] 

Sobel Gradient X
![alt text][image14]

Bitwise ORing of sobel output with the color thresholded images
enhances the lanes detected.

S&R Channel
![alt text][image15] 

S&R | GradX
![alt text][image16]

####3. After doing the Color and Gradient thresholding, next step is to do a perspective transform on the binray images.
In the transform we intend to get the road from top view. This will be needed for calculating the real radius of curvature

I used the following src quadrilateral and destination quadrilateral for prespective transform

src = np.float32([[685,445],
          [1052,673],
          [256,673],
          [595,445]])

dst =np.float32([[950,0],
          [950,720],
          [300,720],
          [300,0]])

Using the cv2.getPerspectiveTransform, I computed the transform and inverse transform matrix.
The code is present in Cell 11
Using the transform matrix I tested one of the binary images

![alt text][image17]

####4. After doing the perspective transform, I detected the lane lines using sliding window method described in the lesson

First I got the histogram of the columns in the bottom half of the image, to determine which columns have high number of pixels.
From the centermost column, select the peak column adjacent on the lefthand side and righthand side.
This gave me the x-coordinates of the start of the lanes from the bottom.

![alt text][image18]

I employed the sliding window approach to search for lane pixels while traversing the image from bottom to top.
Taking this as the starting point, I took window size x=200, y=image_height/9. The function is detect_line() in Cell 13

1. set the window centered around the histogram peak column at the bottom of the image.
2. Check if there are enough white pixels are present in the window to be deduced as lane pixels.
3. Store the lane pixel positions.
3. Calculate the mean x-coordinates of pixels for the current window.
4. Recenter the window and move it upwards and restart with step 1 till we reach top edge of image.

The above steps are done for both the lanes.
Using the stored pixels, I tried to fit a 2nd order polynomial using np.ployfit
Using the computed 2nd order polynomial equation, x-coordinates for the lane line is computed for each y from y_image=0 to y_image=720.
 
x_image = Ai(y_image)^2+Bi(y_image)+Ci ----------- eq 1

This gave a image coordinates for the curved line passing through the lanes.
Ai, Bi and Ci are coefficients with respect to the image pixels.

![alt text][image19]

Few more example images, they are present in the lanedetect.ipynb

####5. At the stage where the coefficients for the polynomial are found with respect to the image plane pixels.
To calculate the radius of curvature, the lane pixels positions are scaled to real world distances (meters) by using the following
conversion factor:

ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/650 # meters per pixel in x dimension

Using conversion factor the lanes pixel positions are transformed to real-world conrdinates.
Please note that this conversion is possible because we are dealing with warped images which are on ground zero plane.
Using the real-world lane pixel coordinates, again a 2nd order polynomial is fit.

x_real = Ar(y_real)^2+Br(y_real)+Cr  -------------- eq 2

Where Ar,Br and Cr are coefficients for real world

After computing the equation, the radius of curvature is computed as

R = ((1+(2Ay+B)^2)^1.5)/|2A|

The radius of curvature is computed for both left and right lanes. The function is find_curvature in Cell 12

The following code is present in function detect_line() in Cell 13
Average radius of curvature is calculated.

R_average = (R_left + R_right)/2

For determining whether the car is aligned with the lanes,

car_center = 1280/2 = 640
For calculating the center_lane, the x-cordinates of both lane lines is calculated for y_image = 719 using eq 1

x_image = Ar(719)^2 + Bi(719) + C

center_lane is calculated by averaging the x-coordinated of both the lanes
The car_offset in real world is calculated as,

car_offset = (center_lane - car_center)*xm_per_pix

####6. 
The polygon is fit using these lane lines coordinates (calculated by eq 1) on the warped image using cv2.fillPoly. 
Using cv2.warpPerspective, the images are unwarped back. The function is draw_lane_lines() in Cell 14

![alt text][image20]

The polygon edges are falling properly on the lanes lines

###Pipeline (video)

####1. I used the above pipeline on the project video in function process_image() in Cell 17 and following is the result:

Here's a [link to my video result](./out_project_final.mp4)

There further optimize my search of lanes lines, I used the lane line locations of previous image to to higly targeted search.
This narrowed my search for subsequent frames. If highly targeted search does not work, the algorithm will fall back to
histogram peak method. This code is present in detect_line() in Cell 13

The lane polygon edges wobble sometimes when pipeline encounters
* shadows
* change in road color

I further employed smoothing of polygon edges by storing the lane line cordinated for the past 9 images.
A weighted average with the lanes of last 9 frames smoothened the lane polygon.
Higher weights are assigned for the latest frames and lower weights for the older frames.
With this approach, the lanes are detected in a good way. This code is present in process_image() in Cell 17

Here's a [link to my smoothened video result](out_project_final_smooth.mp4)

###Discussion

####1. I tried my pipeline on challenge video and it was not giving very good results.
It was not working for few reasons:
* The yellow lane lines are very faint
* There are extra lane lines besides yellow and white lane lines
* There is a tunnel, where no lanes are detected.
* The sliding window for left lane was ending up with the right lane

The color and gradient thresholding were not able to detect the lane lines properly.
I will employ more combinations of color space thresholding to make the pipeline more robust.
Appropriate check is needed for the highly target sliding window approach, so that it can fall back to histogram peak method
