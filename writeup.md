## Writeup Template

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

[image1]: ./examples/distorted-to-undistorted.png "Undistorted"
[image2]: ./examples/test-image-undistorted.png "Road Transformed"
[image3]: ./examples/binary-example.png "Binary Example"
[image4]: ./examples/warped_straight_lines.png "Warp Example"
[image5]: ./examples/fit-lane-lines.png "Fit Visual"
[image6]: ./examples/example_output.png "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first few code cells of the IPython notebook located in "./project.ipynb".  

I start by preparing the variable `object_point_grid`, which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `object_point_grid` is just a replicated array of coordinates, and the `object_points` buffer will be appended with a copy of it every time I successfully detect all chessboard corners in a calibration image.  `image_points` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I then used the output `object_points` and `image_points` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  

I finally apply this distortion correction to a calibration image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The code for this step is contained in the provided IPython notebook.

To display a distortion corrected test image, I simply read the first of the test images and applied the same `cv2.undistort()` call to this image. I used the same camera matrix and distortion coefficients because this is presumably the same camera being used here, and will account for its particular distortions.

This resulted in the following image transformation:

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for this step is contained in the provided IPython notebook.

I used a combinatino of color and gradient thresholds to generate my binary image. I first created an HLS version of the image, then a grayscaled version. Then I used a configurable sobel filter in the x directions, and made it a scaled absolute value sobel filter output. With those transformations performed, the thresholding operation can be done to generate a separate binary image from the sobel filter output and the S channel of the HLS image version.

The thresholds used are configurable by passing in parameters, but default to some sane defaults for the test image used during development. The two generated binary images are then combined by simply using a logical OR operation on each. That is to say, if a pixel has a value in either of the generated binary images, it will be added to the combined binary image. This combined binary is then returned.

Here is an example of the output of my binary generation function on a test image:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warper()` The `warper()` function takes as input an image (`image`). I chose the hardcode the source and destination points for the perspective transform in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by performing the perspective transform on a straight-line image and verifying that the lines appear parallel in the perspective transform. Here is an example output to demonstrate that:

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for lane polynomial fitting is contained in the provided IPython notebook.

I implemented both a sliding window approach as well as a local search if expected lane lines are already known. Both of these approaches require a warped (top-down view) and binary image. 

The sliding window approach converts the bottom portion of the image into a histogram in order to detect sane starting points for the lanes. After starting points are decided, the method splits the input image into a series of windows ascending from these starting points within which the positive pixels are collected and a new center is determined before moving to the next window. This ensures that we do a semi-local search from window to window, as we move up the image, never straying too far from the known previous good lane line pixels. When we reach the top, the pixels gathered in each window can be returned as a polyfit with degree 2. As we kept separate windows for each lane (left and right), we can return a polynomial in the form of a polyfit for each of the lanes.

The local search will accept a left and right polynomial in the form of a numpy polyfit, as well as the input image. This method expects the lanes to be nearby the provided lane polynomials, and so can be used in situations where we can assume each image varies only slightly from the last. This is a reasonable assumption with self-driving vehicles, since we are process forward facing images at a high enough framerate that the lanes should not be too varied.

The output from this part of the pipeline is demonstrated below. The thin yellow lines are the polynomial fits graphed stacked above the left (red) and blue (right) pixel points which were used to fit them. The green rectangles are the windows which were used to discover the lanes as we ascend from the bottom to the top of the image.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code for radius of curvature calculation is contained in the provided IPython notebook.

I first set out the conversion from pixel units to the physical world. Then I defined a function which calculates the radius of curvature for a second degree polynomial at a given value. This will be used later by passing in the polynomials for the left and right lane lines and the bottom of the image (max y value) in order to calculate the radius of curvature at the point closest to the vehicle.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The code for drawing the detected lane back onto the original image is contained in the provided IPython notebook.

Here's what the lane looks like when it is drawn back onto the original image.

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I ran into a few issues when I was using only the sliding window search method for detecting the lane markers. This technique wasn't robust in the face of some of the frames of the video that did not have good lane markers at all. To work around this issue, I used the local search method, only falling back to the sliding window method when necessary (a polynomial fit couldn't be found nearby the last frame's polynomial fit for each lane).

I would expect for this pipeline to produce unreliable results if the lane markings were more sporadic or if the camera mounting were moved to a different position on the vehicle. I also think a particular weakness of this pipeline would be it's lack of smoothing. If the lane detection fails on a few frames in a row, the output would probably not be very good, since it doesn't smooth at all over previous frames. 
