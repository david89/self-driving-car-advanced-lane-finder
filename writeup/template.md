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
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

---

# Camera Calibration

## 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

In order to find the camera matrix and distortion coefficients we need a set of white and black chessboard pictures for calibration purposes (it's recommended to have at least 20). For each picture, we can use the **findChessboardCorners** openCV function in order to find the pixels that correspond to the corners of the chessboard.

![Find corners in a chessboard image](find_corners.png 'Find corners in a chessboard image')

Now, in order to use the **calibrateCamera** openCV function, we need to provide an aggregated set of all retrieved corners and their corresponding object points. The object points correspond to the coordinates of all chessboard corners in a (x, y, z) coordinate system. For example, the first coordinate will be (0, 0, 0) and the last one will correspond to (nx - 1, ny - 1, 0), where `nx` is the number of corners in a chessboard row and `ny` corresponds to the number of corners in a particular chessboard column.

After aggregating all corners and object points, and feeding them to the calibrateCamera function, we get the camera matrix and distortion coefficients we can use in order to undistort an image. For example:

![Undistort a chessboard image](undistorted_chess.png 'Undistorted chessboard image')

# Pipeline (single images)

## 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![Undistorted image](undistorted_image.png 'Undistorted image')

## 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

### Sobel method (gradients)

There are different Sobel operatos we can apply, so let's go through each one.

**Sobel X**: gradient over the x axis.
![Sobel X image](sobel_x.png 'Sobel X image')

**Sobel Y**: gradient over the y axis.
![Sobel Y image](sobel_y.png 'Sobel Y image')

**Sobel XY**: combination of the Sobel X and Sobel Y operators.
![Sobel XY image](sobel_xy.png 'Sobel XY image')

**Sobel magnitude**: represents how much the gradient is changing in the X and Y directions.
![Sobel magnitude image](sobel_magnitude.png 'Sobel magnitude image')

**Sobel direction**: returns a rough estimate of what's the direction of the gradient at each point.
![Sobel direction image](sobel_direction.png 'Sobel direction image')

**Sobel magnitude and direction**: combination of the Sobel magnitude and Sobel direction operators.
![Sobel magnitude and direction image](sobel_magnitude_direction.png 'Sobel magnitude and direction image')

**All Sobel masks**: combination of Sobel XY and Sobel magnitude and direction.
![All Sobel masks](all_sobel.png 'All Sobel masks')

As you can see, the Sobel operator can identify some lines in our image (like the dashed white line on the right), however, it's not able to identify the yellow line on the left side. That's because we need to transform an image into gray scale before feeding it to the Sobel operator. When we do this color space transformation, we lose a lot of information. So, in order to properly identify the lines in the given images, we need to use some additional transforms.

Feel free to check the **Sobel** section in the notebook.ipynb file for more details.

### Color spaces

#### [RGB](https://en.wikipedia.org/wiki/RGB_color_model)

This is the color space we usually read images in.

#### [HSV](https://en.wikipedia.org/wiki/HSL_and_HSV)

Is important to notice that during the RGB to HSV conversion process, some scaling happens ([more information](https://docs.opencv.org/2.4/modules/imgproc/doc/miscellaneous_transformations.html?#cvtcolor)). Therefore, the range of values we operate on are:
* H ∈ [0, 180]
* S ∈ [0, 255]
* V ∈ [0, 255]

If you are curious how to transform RGB values to HSV, we used [this color picker](https://alloyui.com/examples/color-picker/hsv) to find the corresponding HSV values for some colors like white or yellow.

#### [HSL](https://en.wikipedia.org/wiki/HSL_and_HSV)

Similarly to HSV, some scaling happens in the RGB to HSL conversion process. But another important detail is that opencv switches the order of saturation and lightness; so we are actually dealing with a HLS color space.

The range of values we operate on are:
* H ∈ [0, 180]
* L ∈ [0, 255]
* S ∈ [0, 255]

Additionally, we used [this color picker](https://www.w3schools.com/colors/colors_hsl.asp) to find the HSL values for some colors like white or yellow.

Now that we have briefly gone over the most common color spaces, let's discuss how to get the pixels we are interested at based on their colors.

#### Detect yellow pixels

![Detect yellow pixels](yellow.png 'Detect yellow pixels')

We see good results after trying to identify the yellow color in the **HSV** color space.

#### Detect white pixels

![Detect white pixels](white.png 'Detect white pixels')

We see good results with different color spaces. However, we think the **HSL** color space was easier to tune, since we only care about lightness.

#### Combining white and yellow pixels 

![Combine white and yellow pixels](white_and_yellow.png 'Combine white and yellow pixels')

Merging the white and yellow masks (2nd image) give us good results. However, we do a better job of detecting white and yellow lanes by using the saturation channel in the HSL color space (third image). Although, the HSL mask also detects the entire sky, but we can solve that problem by selecting a region of interest. Therefore, we are going to go for **HSL** for detecting white and yellow pixels using the **saturation channel**.

Feel free to check the **Color Spaces** section in the notebook.ipynb file for more details.

### Combine Sobel operators and color transforms

![Combine Sobel operators and color transforms](sobel_and_color.png 'Combine Sobel operators and color transforms')

After combining the Sobel operators and the color transforms we see that we have detected the pixels we mostly care about.

## 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Before getting into the details of the perspective transform, let's discuss the region of interest.

### Region of interest.

The region of interest corresponds the area in the image we are going to apply the perspective transformation on. In our case, we are using a trapezoid as showed in the following picture:

![Region of interest](region_of_interest.png 'Region of interest')

### Bird's eye view

Now that we have a region of interest, we need to apply a perspective transformation into that specific section of the image to get a bird's eye view that we can later use for lane detection purposes. In our case, these are the vertices that identify our region of interest:

```python
array([[  50.,  670.],
       [  50.,   50.],
       [1230.,   50.],
       [1230.,  670.]], dtype=float32)
```

And these are the vertices that represent the new perspective we want to get:

```python
array([[  50.,  670.],
       [  50.,   50.],
       [1230.,   50.],
       [1230.,  670.]], dtype=float32)
```

Finally, we can call the **getPerspectiveTransform** openCV function in order to get the perspective transform we need to apply in order to get the bird's eyes view.

For example, this is one example of the perspective transform applied:

![Perspective view](perspective.png 'Perspective view')

Feel free to check the **Region of interest** and **Perspective transform** sections in the notebook.ipynb file for more details.

## 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

We identified the left and right lines using different steps we are going to discuss right now.

### Histograms

First of all, we calculate a histogram of the bottom section of a bird's eye view image in order to calculate the "hot" x coordinates, i.e., the x coordinates with more activated pixels.

For example, if we calculate the histogram on the bottom half of the perspective image we showed in the previous section, we get something like this:

![Histogram](histogram.png 'Histogram')

As you can see in the histogram, the pixels 189 and 1156 are the most relevant pixels in our case (one for the left line and the other one for the right line).

### Sliding windows

Now that we have some base x coordinates, we can perform a sliding window search. We create `n` windows of a fixed size. For each line, we create a window of a fixed size centered at the x coordinate calculated in the histogram section. Then, for subsequent iterations, we update the x coordinate of each window based on the mean of activated x coordinates in the previous iteration.

![Sliding window](sliding_window.png 'Sliding window')

Feel free to check the **Find lane pixels** section in the notebook.ipynb file for more details.

### Fit polynomial

Now that we have the set of pixels that belong to a certain line, let's fit a polynomial of second degree that represents such line. For that, we can use the **polyfit** numpy function.

The yellow line in the following image represents the fitted polynomial for the left and right lines.

![Fitted polynomial](fitted.png 'Fitted polynomial')

## 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The second degree polynomial calculated in the previous step can be represented as:

f(y) = A * y <sup>2</sup> + B * y + C

In order to calculate the radius of the curvature of the lane, we need to compute the following:

R<sub>curve</sub>(y) = (1 + f'(y)<sup>2</sup>)<sup>3/2</sup>) / |f''(y)|

Where

f'(y) = 2 * A * y + B

f''(y) = 2 * A

Where all reduces to:

R<sub>curve</sub>(y) = (1 + (2 * A * y)<sup>2</sup>)<sup>3/2</sup>) / |2 * A|

In our particular case, we can use y = image.shape[0] in order to measure the curvature at the pixel closest to the vehicle. However, it's worth mentioning, that the measurements we are getting are in pixels, and we need to transform that into real word units like meters. In order to accomplish that, we are going to use measurements of a lane in real life, and the size of our warped picture, which are:

**y_meters_per_pixel = 30 / 720**

**x_meters_per_pixel = 3.7 / 700**

For example, if we calculate the curvature of the fitted polynomials mentioned in the last step, we get the following measurements:

* Left curvature is 780.68m
* Right curvature is 1481.69m
* The car is 0.17m off center

Feel free to check the **Measure curvature** section in the notebook.ipynb file for more details.

## 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Finally, in order to show the area that corresponds to the current lane, we need to project the polynomials fitted in the previous step.

For example:

![Curvature projection](projected.png 'Curvature projection')

Feel free to check the **Project our curvatures into the original image** section in the notebook.ipynb file for more details.

---

# Pipeline (video)

In order to improve the accuracy of the lane detection pipeline in a video, we had to introduce two more concepts.

## Search from prior

If we have a high confidence measurement from a previous frame, we could search within a certain margin of the fitted polynomial, instead of restarting the whole sliding window search.

For example:

![From prior](from_prior.png 'From prior')

Feel free to check the **Search from prior** section in the notebook.ipynb file for more details.

## Keeping track of previous frames

Analyzing frame by frame may result in lines jumping a bit from frame to frame, leading to innaccurate results. In order to fix that problem, we decided to keep the measurements of the last n iterations, and use an averaged fitted polynomial.

Feel free to check the **Keeping track of previous frames** section in the notebook.ipynb file for more details.

## Final result

After implementing all the necessary pieces and adding the search from prior and keeping track of previous frames, we ended up with the [following result](../output_videos/output_project_video.mp4)

---

# Discussion

## Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

There are several improvements that can be applied.

1. The pipeline is slow. There are several operations we are calculating over and over again in several places.

2. Sometimes the line detection fail when noise is introduced. For example, in the challenge video, there are some changes in the color of the pavement that confuse our pipeline. In order to improve this situation, we need to measure how confident we are in the measurements we are taking. For example, if we have n high confidence measurements for the last several frames, and we get an outlier where the curvature of the left line doesn't match the curvature of the right line, or the x coordinates have changed drastically, we can just discard that frame and report the weighted measurements from the previous frames.

3. The line detection fails when we take sharp turns. For example, in the harder challenge video, there are some frames where one of the lines dissappear. We can keep track of the progression of the x coordinates for each line, if at some point we are getting too close to the borders and the line dissappear in the subsequent frames, we should stop looking for that frame and assume that's out of the picture and has a similar curvature to the other line we have knowledge of.
