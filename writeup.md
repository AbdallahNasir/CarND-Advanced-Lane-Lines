# Advanced Lane finding
## Introduction
This project aims to find lane lines in an advanced way, using image processing techniques.

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

## Camera Calibration
To calibrate the camera, we used chess board images, identified where is the crosses, and calculated the camera matrix and the distortion coeffecients using them.

```python

# find corners
ret, corners = cv2.findChessboardCorners(gray, (9,6),None)

# If found, add object points, image points
if ret == True:
    objpoints.append(objp)
    imgpoints.append(corners)

# Calibrate

ret, mtx, dist, rvecs, tvecs = cv2.calibrateCamera(objpoints, imgpoints, gray.shape[::-1], None, None)

# Undistort

def undistort (img):
    return cv2.undistort(img, mtx, dist, None, mtx)

```

![alt text](/resources/calibration.PNG "Image Calibration")

## Color Transform and binary image creation
Images got converted to binary by combining S threshold binary, sobel x threshold binary and sobel y threshold binary. The min and max thresholds obtained experimantaly.

![alt text](/resources/binary.PNG "Binary transformation")

## Percpective Transformation
To convert image into Birds-Eye view, we need to apply a percpective transformation. Four points will be taken and predefined in the images to be used for the transformation.

The trick here is how to pick these four points, I tried with the straight lines image until I get the best points. I picked the points manually from the image, and tested if after tranforming, these lines will be a straight lines.

![alt text](/resources/percpective.PNG "Points for percpective transformation")

```python
spt = np.array([[ 270, 680],[ 595, 450],[688, 450],[1045, 680]])
offset = 250
dpt = np.array([[ offset, 720],[ offset, 0],[1280 - offset, 0],[1280 - offset, 720]], dtype='float32')
M = cv2.getPerspectiveTransform(spt.astype('float32'), dpt)
warped = cv2.warpPerspective(s_ud_img, M, gray.shape[::-1])
```
![alt text](/resources/bird-eye.PNG "Bird-eye view, the output of percpective transformation")

Applyting this code to the binary images results in the following.

![alt text](/resources/binary-warped.PNG "Bird-eye view of binary images")

## Lane lines fitting
In order to identify where the lane might be, a histogram of the points is calculated for the bottom half of the image.

![alt text](/resources/histogram.PNG)

This histogram is used to identify where the lines are starting from. 

Following, a windowed search is used to identify the points of each lane line.

Starting from the location identified by the historgram, and for each window, the average of points location is calculated, and it is used as the location for the next window. As the window size is fixed, part of the points are identified as lane points, those which only are inside the identified windows. those points will be used to fit a polynomial that represents the best fit for the lane line. This is done for each lane line, i.e. the right and the left.

After collecting the points that relies in each line, the polyfit is used to determine the best fit curve for each line.

![alt text](/resources/best-fit.PNG "Best fit line for the lane points")

## Curveture and distance from the center of the lane

Curveture and distance from the center are calculated from the best fit lines. The unit is converted from pixel to meter to make the result value reasonable.

```python
def calculateCurveture(left_fit, right_fit):
    # Fit new polynomials to x,y in world space
    left_fix_x = left_fit[0]*ploty**2 + left_fit[1]*ploty + left_fit[2]
    right_fix_x = right_fit[0]*ploty**2 + right_fit[1]*ploty + right_fit[2]
    left_fit_cr = np.polyfit(ploty*ym_per_pix, left_fix_x*xm_per_pix, 2)
    right_fit_cr = np.polyfit(ploty*ym_per_pix, right_fix_x*xm_per_pix, 2)
    # Calculate the new radii of curvature
    left_curverad = ((1 + (2*left_fit_cr[0]*y_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
    right_curverad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])
    # Now our radius of curvature is in meters
    return left_curverad, right_curverad
    
def CalculateDistanceFromCenter(left_fit, right_fit):
    rightpt = right_fit[0]*y_eval**2 + right_fit[1]*y_eval + right_fit[2]
    leftpt = left_fit[0]*y_eval**2 + left_fit[1]*y_eval + left_fit[2]
    centerpt = np.mean([rightpt, leftpt])
    deviationFromCenter = 1280/2 - centerpt
    return xm_per_pix * deviationFromCenter
```

## Final result
The area between the two lines fit is highlighted, and the values of curveture and distance from the cetner are printed into the final image.

![alt text](/resources/result.PNG "Final Output")

## video Submission
The pipeline is tested on a video.

[Final result](/output1.mp4)

## Future work
It would be better to include history from the previous frames into the implementation. Not only to speed up the proces, but also to filter spikes. For example, the line curveture cannot change from 1000 to 500 suddenly. This way, a smooth lines detection can be made to ensure that the system will not identify crazy lines in the system.

As the percpective transformation, a fixed four points were used, while that is not accurate. I guess using input from other devices can be useful to identify which points to be used for percpective tranformation.
