## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./examples/undistort_test_img.png "Undistortion Example"
[image3]: ./examples/binary_combo_example.png "Binary Example"
[image4]: ./examples/warped_straight_lines.png "Warp Example"
[image5]: ./examples/warped_binary_curved_lines.png "Warp Example for Curved Lines"
[image6]: ./examples/sliding_window.png "Sliding Window Search Example"
[image7]: ./examples/next_frame.png "Simpler Search Example"
[image8]: ./examples/image_draw.png "Lane Drawing Example"
[video1]: ./output_videos/project_video.mp4 "Video Output"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 3rd and 4th code cells of the Jupyter notebook located in "./CarND-Advanced-Lane-Lines.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To make the pipeline managed easier, I wrote a few helper functions for each image processing step.

The first one is `camera_calibration()` function (in the 5th code cell), where I feed in an image and apply camera calibration matrix calculated from the chessboard obtained from last step. The result from original image and undistorted one is shown below:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Next is `color_gradient_threshold()` function (in the 6th code cell). I used a combination of color and gradient thresholds to generate a binary image.
#### 1. I first fed in an image.
#### 2. Gradient thresholding:
2a. Coverted image to HLS color space.

2b. Then I seperated light channel and used it for Sobel to take derivative in horizontal direction.

2c. A binary image was obtained by filtering the scaled Sobel derivative within (20, 100) range.

#### 3. White color thresholding:
3a. Coverted image to Luv color space.

3b. Then I seperated l channel and used it for picking up white color.

3c. A binary image was obtained by filtering the luv_l_channel within (220, 255) range.

#### 4. Yellow color thresholding:
4a. Coverted image to Lab color space.

4b. Then I seperated b channel and used it for picking up yellow color.

4c. A binary image was obtained by filtering the lab_b_channel within (155, 210) range.

#### 5. Finally, I combined 3 binary images from 2., 3. and 4. for gradient and color thresholding.

Here's an example of my output for this step.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `perspective_transform()`, which appears in 7th code cell in the Jupyter notebook.  The `perspective_transform()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src_pts = np.float32([[595, 450], [200, 720], [1080, 720], [685, 450]])
dst_pts = np.float32([[300, 0], [300, 720], [980, 720], [980, 0]])
```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 595, 450      | 300, 0        | 
| 200, 720      | 300, 720      |
| 1080, 720     | 980, 720      |
| 685, 450      | 980, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

Besides straight lines, I also tested the `perspective_transform()` onto an binary curved lines image, the result is also showing parallel lines as below:

![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

In the code cell 9 and 11, I defined 2 functions called `sliding_window()` and `next_frame()`, which will take warped binary image from previous step.

The `sliding_window()` will be used for first frame in the video (or reset one after a series failure), which applies sliding window search to identify lane lines:
1. Take histogram of the bottom half of image to determin the rough position for each lane lines.
2. The image then would be seperated to several windows and the non-zero pixels would be detected inside each window.
3. If the detected pixels were above certain minimun, they would be consider as good lane lines indices.
4. After loop through all the window, those detected line indices would be fitted as 2nd degree polymonial fuction by using `np.polyfit()`.
5. X, Y coordinates and fits were returned for later use.

One visualization example for sliding window search:

![alt text][image6]

The `next_frame()` will be used for all subsequent frames in the video, which identifies lane lines as well but with a simpler approch since the lane lines location was roughly determined by the previous frame.

Visualization example for `next_frame()`.

![alt text][image7]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The next helper function I wrote is `find_curvature()`, which located in the code cell 13 of the Jupyter notebook.

This function takes polynomial fits from the previous step and calculates the curvatures for each lane lines. One thing to note is that the x, y coefficients were converted to meters from pixels before the calculation.

Equation for radius if curvature:
$ R_{curve} = \frac{(1+(2Ay+B)^2)^{3/2}}{|2A|} $

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The last fuction is `draw_image()`, which is in code cell 14. The fuction takes points from the fitted lines and plot them back onto the original undistorted image by using the Minv (inverse perspective transform matrix).

Then, I also showed the radius of curvature and vehicle off-center for the current frame.

Here is an example of my result on a test image:

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_videos/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

For the implementation of this project, I faced a couple problems. Here is the the list of them and my workarounds:

1. The lane lines detection were jumping around: 

Sometimes the detection was not so accurate, so is a good idea to do the sanity check for each frame processed. I made a `Line()` class to keep tracks on the chracteristics of left and right lands.

Each time I finished the image processing, I check (1) if the difference of radius of curvature is < 100m, (2) if the lane width is within 3.7m +/- 0.5m and (3) if the two lanes are roughly parallel.

If it passed the check, I will update the class with new data and take average of past 5 frames. If it failed, I would retain the results from previous good frame.

I put a lost counter inside the pipeline, so if it failed sanity check for 5 consecutive times, I would reset and treat it as the first frame and conduct sliding window search.

After the workaround, I was able to get the results smoother.

2. Bad detection on white concerte road surface:

However, even after item 1 above, I still failed to correctly identify the lane lines when the vehecle is on the white concrete surface. I think it is because my binary thresholding is not robust enough.

I go back to fine tune my gradient and color thresholding function. I found that by seperating while and yellow color filters, it is more robust if the background is very bright. The reason is that I can choose differnt Hue and Saturation ranges for each of the color.

The similar approach was also used in my first lane line project.

3. Beyond this point:

Up to this point, I was able to process the project video pretty well. But after trying the same pipeline on chllenging video, I got some issue when the road condition is not idea, such as crack/patched lines that would confuse my image processing pipeline.

To further improve my algorithm, I will probably need to revisit the binary thresholding part so that it can act even more robost. Perhaps converting to other color space (HSV) or applying more thresholds together to screen out unwanted line detection.

The other issue is when vehicle, instead of traveling on the highway, is on the local/hilly road. The sharp turns and rapidly level chaning will also make the line detection more diffcult.

A possible fix would be: establish even more strong algorithm so that we can reduce to averaging number of past iterations or put more weights on the current frame. To combat the leveing, other than hard-coded source points for perspective transformation, an adaptive points that can be ajusted by checking the end of road might also be helpful.
