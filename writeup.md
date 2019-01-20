## Writeup 

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

[image1]: ./output_images/distortion_correction/out10.jpg "Undistorted"
[image2]: ./output_images/corrected_test_images/corrected_1.jpg "Road Transformed"
[image3]: ./output_images/thresholded_images/thresholded_1.jpg "Binary Example"
[image4]: ./output_images/warped/warped_3.jpg "Warp Example"
[image5]: ./output_images/lane_detection/detected_1.jpg "Fit Visual"
[image6]: ./output_images/final_output/final_1.jpg "Output"
[video1]: ./test_videos_output/project_video.mp4 "ProjectVideo"
[video2]: ./test_videos_output/challenge_video.mp4 "ChallengeVideo"


### Camera Calibration

The code for this step is contained in the IPython notebook **cameraCalibration.ipynb**. In the notebook distortion matrix and coefficients are calculated and stored in a *pickle* file.

I start by preparing *object points*, which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objPoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgPoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. For detecting the chessboard corners in the image `cv2.drawChessboardCorners()` function is used.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the images used for calibration using the `cv2.undistort()` function and saved it to the folder *./output_images/distortion_correction.* One Example corrected image is shown here:

![alt text][image1]

### Pipeline (single images)

Rest of the code is in the IPython notebook **DetectingLaneLines.ipynb**

#### 1. Distortion Corrrection

The code for distortion correction on the test images is in the cell no. 4 of the notebook. In the code the test images are undistorted using `cv2.undistort()` function and the distortion coefficients obtained after camera calibrtation. The output is stored in the folder *output_images/corrected_test_images.* One of the corrected image is shown here:

![alt text][image2]

#### 2. Gradient and Color thresholding

The main method that produces the combined thresholded images after various transform is named *image_thresholding* and is located at cell no. 6 and the helper function to achieve this are located in cell no. 5. Foloowing thresholds are used:
* Gradient value threshold in x direction with low and high threshold - (5,100)
* Gradient value threshold in y direction with low and high threshold - (30,100)
* Gradient magnitude threshold  with low and high threshold - (20,100)
* Gradient direction threshold
* H and S threshold in HLS space to clearly identify the lane lines in bright or very dark light
* L threshold from HLS color space to remove darker pixels from the image

All the threholded test image output is located in the folder *output_images/thresholded_images/* . An example is shown below:

![alt text][image3]

#### 3.Perspective Transform

The code for my perspective transform includes a function called `perspective_transform()`, which appears  the 8th code cell of the IPython notebook.  The `perspective_transform()` function takes as input an  image (`thrs_image`). I chose the hardcode the source and destination points in the following manner:

```

     bl = [220,720]
    br = [1110, 720]
    tl = [570, 470]
    tr = [722, 470]

    src = np.float32([tl,tr,br,bl])
    dst = np.float32([[320,0],[920,0],[920,720],[320,720]])
```

Apart from the warped image the function also outputs the tranformation and inverse transformation matrix.
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto  test images and its warped counterpart to verify that the lines appear almost parallel in the warped image. The output is located in the folder *output_images/warped/*. An example is given below:

![alt text][image4]

#### 4. Identifying Lane Line pixels
Three functions are used identifying  the lane-line pixels and fit their positions with a polynomial

* `sliding_window():` The code for the function can be found in the  code cell no. 10. This function is called to search the perpective transfomred thresholded image from the scratch for lane-line pixels. An histogram is obtained from the input thresholded bird's eye view of the image. The position of peaks in the histogram gives the indication of the base of the respectice lane pixels. Starting from that base position, an window of known height and width is slided, shifting up and also changing its center depending on the number of  of nonzero pixels identified in the image. The width of window used is 100 and number of windows used to cover the height of the entire image 9. 

The x and y cordinates of non zero lane pixels are found and a polynomial is fitted through it.

* `search_around_poly():` The code for the function can be found in the  code cell no. 11. Once an polynomial is fitted in previous frame of the video, searching from the scratch for lan-line pixels in the current frame is inefficeient. So, a focussed search is carried out with in a margin (in present case margin of 50)  of the polynomial identified in the previous frame to identify the laneline pixels in the currrent frame. If the identified lane pixels passes thte sanity check of having absolute difference in their curvature less than 2 and having almost same slope the lines are kept or they are discarded.

* `fit_poly():` The code for the function can be found in the code cell no. 10. The function is used to fit a polynomial over detected lane-line pixels.


Output on test images is located in the folder *ouput_images/lane_detection*.An Example where lanes are identified is :

![alt text][image5]

#### 5. Radius of curvature and Vehicle offset

The function for computing the radius of curvature is defined in *Line* class itself defined in code cell no. 2. The function computes the radius of curvature in meters. The radius of curvature is calculated using the formula discussed in class.
The vehicle offset is calculated as the difference between the center of the image and center of the lane converted to meters. the code is located in cell 13.

#### 6. Main pipeline for video

The main pipeline is a function named `detect_lanes().` The pipeline has following steps:

* Undistort the input image
* Use function `image_thresholding()` to obtained a binary thresholded image.
* Get a bird's eye view of the thresholded image using the function `perspective_transform()`
* If lane lines were detected in the previous frame of the video use the function `search_around_poly()` to perform a focussed search. If the fucntion fails to detect new lane-line pixels or detected lan-lines does not pass the sanity check based on radius of curvature and slope the frame is dropped.
* If no lane lines were detected in the previous frame, new lane lines are detected from scratch using the function `sliding_window()`. If the function fails to detect any new lane line pixels the frame is dropped.
* The detected lane area is covered with polygon on the warped image. An inverse perspective transform is performed and the resultant image is combined with the original image. While drawing the polygon, average value of the lane-line pixels are used. (avereged over last 30 frames)

The final output for all the test images is in the folder *output_images/final_output*. One example image is shown here:

![alt text][image6]

---

### Pipeline (video)

#### 1.Result videos

Here's a [link to my video result][video1]

Here's is the link to challenge video [link to challenge video][video2]

---

### Discussion

* Needed to experiment with various threshold value for gradient and direction based thresholds.
* Obtaining a proper perspective transform was challenging, needed to experiment with various values.
* The pipeline does very well for the project video, but it does average on the challenge video. The darker running line on the road along the lane gets detected as lane line which causes issues. Also there is section in the video where the car passes under a bridge, its very dark there and no lane lines are detected,specially left lane line pixels remains undetected. Better perpective transform and thresholding can solve the issue.
* The pipeline fails if there is a sharp turn in the video. Sliding window algorithm can be improved upon when the lane lines goes off the image and window reaches on left or right edge of the image. Also, averaging over less number of frames may also improve the results. 
