# **Finding Lane Lines on the Road** 

This is the first project of Term 1 of Udacity'sSelf-driving Nanodegree program. The main objective of the project is to develop a video processing pipeline that annotates input dashcam video with left and right lane lines.

Some concrete requirements for the project are:
* The output video is an annotated version of the input video.
* In a rough sense, the left and right lane lines are accurately annotated throughout almost all of the video. Annotations can be segmented or solid lines
* Visually, the left and right lane lines are accurately annotated by solid lines throughout most of the video.
* Describe the limitations of the pipeline and discuss possible ways to improve it. 



[//]: # (Image References)

[image1]: ./examples/grayscale.jpg "Grayscale"

---

### Reflection

### 1. Pipeline description

The video processing pipeline consists of the following steps:

# 1. Detecting yellow and white regions in the input image

The first step in the pipeline is to isolate the white and yellow regions in the input RGB image. This allows the lane lines (which are white and/or yellow in color) to be clearly identified in the input image. The RGB colorspace is not ideal for detecting yellow lines, particularly in low-light/shadowed images. This is because the RGB colorspace is highly sensitive to changes in brightness: hence in images with shadows or bright sunlight, the range for Yellow lanes in the RGB model will vary a lot. For a more in-depth description of this, refer to this [paper](https://www.researchgate.net/publication/220777443_An_Adaptive_Method_for_Lane_Marking_Detection_Based_on_HSI_Color_Model)

Instead we employ the HSL colorspace for detecting both yellow and white regions in the input image. HSL stands for Hue, Saturation and Lightness. The main advantage of HSL (over RGB) is that it makes it easy to select a color quickly and furthermore, it is easier to specify varying levels of lightness and saturation for a particular color. The HSL colorspace is shown below.

The Hue wheel on the left runs from 0-360 degrees and is essentially a color-picker. The distance from the middle of the color wheel is called the ‘Saturation’, or how much of a given hue is present. Looking closely at the colorwheel, it is apparent that the color becomes more pronounced and more vivid as one travels from the center of the circle to the edge. The Lightness value of an HSL color is in a third dimension, which actually makes the HSL system a cylinder, as shown in the right image. Both S and L components are specified as a percentage from 0-100%.

HSL Colorwheel              |  HSL Cylinder
:--------------------------:|:-------------------------:
![HSL Colorwheel](/desc_images/hue-wheel-300x300.jpg)    |     ![HSL Cylinder](/desc_images/hsl-cylinder-300x228.jpg)

It is easy to see that the HSL ranges for white color is:
* White Min: [0, 200, 0]
* White Max: [255, 255, 255]

And the corresponding ranges for yellow color is:
* Yellow Min: [10, 0, 100]
* Yellow Max: [40, 255, 255]

NOTE: The range for the respective HSL ranges are scaled to an 8-bit representation (0-255).

The detected images are shown below:

Original images             |  Yellow and White regions
:--------------------------:|:-------------------------:
![Original_im1](/desc_images/orig_solidWhiteCurve.jpg)  |  ![HSL_yellow_white1](/desc_images/wy_solidWhiteCurve.jpg)
![HSL Original_im2](/desc_images/orig_solidYellowCurve2.jpg)   |   ![HSL Cylinder](/desc_images/wy_solidYellowCurve2.jpg)


# 2. Applying Gaussian blurring

Gaussian blurring blurs sharp gradients and noise in the input image making it easier to detect pronounced edges later in the pipeline. We use a kernel of size 13 in order to ensure robust edge detection. Higher order kernels require increased computation time, which is an important factor for real-time video processing.

Yellow/White images            |  Gaussian blur with a 13x13 kernel 
:--------------------------:|:-------------------------:
![Yellow_white_im1](/desc_images/wy_solidWhiteCurve.jpg)  |  ![Blur_Yellow_white_im1](/desc_images/blur_solidWhiteCurve.jpg)
![Yellow_white_im2](/desc_images/wy_solidYellowCurve2.jpg)   |   ![Blur Yellow_white_im2](/desc_images/blur_solidYellowCurve2.jpg)

# 3. Grayscaling the image

Next, we apply grayscaling on the blurred images, which makes it easier (and faster) to apply the Canny edge-detection algorithm in the following step:

Grayscaled image 1           |    Grayscaled image 2
:--------------------------:|:-------------------------:
![Grayscaled Image 1](/desc_images/gray_solidWhiteCurve.jpg)  |  ![Grayscaled Image 2](/desc_images/gray_solidWhiteCurve.jpg)

# 4. Applying Canny edge detection

To the grayscaled images, we apply the well-known Canny edge detection algorithm with low and high thresholds set according to the pixel intensities in the input image. First, we compute the median single-channel pixel intensity in the input grayscale image: v. The low and high thresholds are respectively v\/3 and 2\*v/3.

Grayscaled images           |    Canny edge detection
:--------------------------:|:-------------------------:
![Grayscaled Image 1](/desc_images/gray_solidWhiteCurve.jpg)  |  ![Canny Image 1](/desc_images/canny_solidWhiteCurve.jpg)
![Grayscaled Image 2](/desc_images/gray_solidYellowCurve2.jpg)  |  ![Canny Image 2](/desc_images/canny_solidYellowCurve2.jpg)

# 5. Masking a region of interest (RoI) to only select potential lane lines

To isolate only the lane lines, we apply a RoI mask that is tailored to the dimensions of the input image:

RoI for isolating lanes     |  Masked lane lines in RoI
:--------------------------:|:-------------------------:
![RoI Image 1](/desc_images/roi_m_solidWhiteCurve.jpg)  |  ![RoI post Image 1](/desc_images/roi_solidWhiteCurve.jpg)]
![RoI Image 2](/desc_images/roi_m_solidYellowCurve2.jpg)]  |   ![RoI post Image 2](/desc_images/roi_solidYellowCurve2.jpg)]

# 6. Applying the Probabilistic Hough Transform for detecting lines

We next apply OpenCV's Probablistic Hough Transform for detecting lane lines in the RoI. The algorithm is implemented by the HoughLinesP function with the following parameters: 

```python
rho = 1 # distance resolution in pixels of the Hough grid
threshold = 20    # minimum number of votes (intersections in Hough grid cell)
min_line_len = 20  # minimum number of pixels making up a line
max_line_gap = 300  # maximum gap in pixels between connectable line segments
```
The Hough Transform outputs a set of line segments ('Hough lines') detected in the input image. The line segments are specified by the x,y coordinates of their endpoints in a list, like so: [x1, y1, x2, y2]. A list consisting of the detected Hough lines is output by the Hough Transform, like so:
```
[[  0 539 959 539]], [[  0 537 959 537]], [[  0 538 959 538]], [[521 342 958 536]], [[524 342 957 535]], [[ 11 536 448 342]], [[  1 536 436 342]], [[  8 536 445 342]], [[517 342 954 536]], [[  2 536 439 342]], [[511 342 948 536]], [[  6 536 442 342]], [[514 342 951 536]], [[527 342 805 466]], [[232 431 432 342]], [[ 15 536 452 342]], [[529 342 686 412]], [[487 342 946 536]], [[558 354 874 536]], [[446 342 551 351]], [[454 342 539 348]], [[291 462 438 343]], [[282 460 425 344]], [[589 368 893 536]], [[520 342 956 536]], [[508 342 930 529]], [[424 344 549 350]], [[102 490 434 342]], [[475 342 540 346]], [[426 345 496 348]], [[281 462 429 342]], [[405 353 498 342]]
```

# 7. Extrapolating the lane lines to the end and start of the road

The Hough lines are fed to a function that draws the final lane lines on the original image. 

This consists of several steps:
* Group left and right lane lines based on their slope: Left lane lines have negative slope; right lane lines have positive slope. This fact can be exploited to separate the Hough lines into left lane and right lane candidates.

* Best-fit line: For each lane, we compute the best fit line based on a first-order linear regression. Numpy's [`polyfit`](https://docs.scipy.org/doc/numpy/reference/generated/numpy.polyfit.html) function is used for this purpose:

```
        z_right = np.polyfit(x_right,y_right,1)
```

Here x_right and y_right contain the X coordinates and Y coordinates respectively of the Hough Lines corresponding to the Right lane. consists of a tuple `(m, b)` which describes the parameters of the best-fit line: `y = mx + b`. This is the line to be extrapolated to the start and end of the lane. we identify the coordinates of the endpoints of the extrapolated lane line as shown by the yellow points in the right figure below. Note that the coordinate system for vertices differs from the coordinate system for the image which are read as matrices by OpenCV and follow a row-major indexing.

Coordinate System     |  Lane line extrapolation end-points
:--------------------------:|:-------------------------:
![Coordinate System](/desc_images/coordinate_system.png)  |  ![Lane line extrapolation][/desc_images/line-segments-extrapolation.jpg]

If a left lane or right lane is not detected by the Hough detector, no line is drawn. This occurs a few tens of times during the video. 

# 8. Superimposing the detected lane lines on the original image

The extrapolated lane lines are superimposed on the original image using alpha-blending:

![alt text][image1]


### 2. Identify potential shortcomings with your current pipeline


One potential shortcoming would be what would happen when ... 

Another shortcoming could be ...


### 3. Suggest possible improvements to your pipeline

A possible improvement would be to ...

Another potential improvement could be to ...
