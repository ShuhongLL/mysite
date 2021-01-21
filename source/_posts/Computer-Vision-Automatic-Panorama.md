---
title: Computer Vision - Automatic Panorama
date: 2021-01-21 16:25:50
tags: [CV, Panorama]
katex: true
mathjax: true
photos: ["../images/Panorama_cover.jpg"]
---

Panorama is a wide-angle view or representation of a physical space in photography, and this can be obtained by blending multiple mosaicing together to generate a wild scene.
<!-- more -->

## Intuition

We have a series of images taken from the same place but varies in different angles, and we would like to blend them together to obtain a panorama. Similar to solve puzzles, we firstly need to identify the same features among images, and we know that the images that contains the identical objects can be blended together. However, even the same object was captured by two images, the photos themselves were taken from different angles. As a consequence, the scenes are twisted and we cannot simple blend them together. 

![homography example 1](homography-example-1.jpg)

Therefore, we need to apply image warping to one of the image to project it back the the reference image so that it will look like that they were captured from the same optical center. The following images show the warping of the result (thrid image) of warping the first image referring to the second image. We will use two trival images as an example to demonstrate the process of automatic panorama.

![homography example 2](homography-example-2.png)


## Feature Matching

We need to identify the identical features among images. One way of doing this is to compute the [harris corners](https://en.wikipedia.org/wiki/Harris_Corner_Detector) over the entire images. Afterwards, we can manually choose good features from candidates.

```python
from skimage.feature import (corner_harris, corner_peaks, plot_matches, BRIEF, match_descriptors)
from skimage.transform import warp, ProjectiveTransform, SimilarityTransform
from skimage.color import rgb2gray

imLgray = rgb2gray(imLeft)
imRgray = rgb2gray(imRight)

# NOTE: corner_peaks and many other feature extraction functions return point coordinates as (y,x), that is (rows,cols)
keypointsL = corner_peaks(corner_harris(imLgray), threshold_rel=0.0005, min_distance=5)
keypointsR = corner_peaks(corner_harris(imRgray), threshold_rel=0.0005, min_distance=5)

extractor = BRIEF()

extractor.extract(imLgray, keypointsL)
keypointsL = keypointsL[extractor.mask]         
descriptorsL = extractor.descriptors

extractor.extract(imRgray, keypointsR)
keypointsR = keypointsR[extractor.mask]
descriptorsR = extractor.descriptors

matchesLR = match_descriptors(descriptorsL, descriptorsR, cross_check=True)
```

![feature matching harris corners](feature-matching-harris-corners.png)

Another way is using Ransac sampling by randomly choosing a fix amount of features from two images, compute the projection matrix (homography matrix), and count the amount of inliers that fit the projection matrix. Repeat the process multiple time and pick an optimal projection matrix which has the maximum inliers.

```python
src = keypointsR[matchesLR[:, 1]][:, ::-1]
dst = keypointsL[matchesLR[:, 0]][:, ::-1]

model_robust, inliers = ransac((src, dst), ProjectiveTransform, min_samples=4, residual_threshold=4, max_trials=100)
```

![feature matching ransac](feature-matching-ransac.png)


## Image Warping with Homography

Image warping is the process of digitally manipulating an image such that shapes portrayed in the image have been distorted.

![image warping](image-warping.jpg)

However, a general image warping can twist the image so that the original features and properties may no longer preserve. Therefore, we’d like to restrict the warping process to only use homograpies where the straight lines will be preserved after applying image transformation. This is important since we will use the warped images to blend together to form a view of panorama, where different images are only taken from different angle. 

Homography allows several types of transformation: Translation, Euclidean, Similarity, Affine and Projectivity. 

![homography](homography.png)

For a pixel in the image plane, it has horizontal axis (x) and vertical axis (y) which can be represented as a vector {% mathjax %}[x, y]{% endmathjax %}. In order to do arbitrary translation, we add a third parameter {% mathjax %}1{% endmathjax %} to the vector {% mathjax %}[x, y, 1]{% endmathjax %}. The homography matrix will contain a maximum of 9 unknown parameters and the transformation can be represented as:

{% mathjax %}
\begin{bmatrix}
wx'\\
wy'\\
w
\end{bmatrix} = \begin{bmatrix}
a & b & c\\
d & e & f\\
g & h & i
\end{bmatrix}\begin{bmatrix}
x\\
y\\
1
\end{bmatrix}
{% endmathjax %}

where {% mathjax %}w{% endmathjax %} is the scaling factor. Solve this matrix and we can obtain two functions with 8 unknowns with additional assumption that i = 1:

{% mathjax %}
ax + by + c - gxx' - hyx' - ix' = 0
{% endmathjax %},
{% mathjax %}
dx + ey + f - gxy' - hyy' - iy' = 0
{% endmathjax %}

To solve these 8 unknowns, we need 2x4=8 linear equations. Each pair of pixels contains 2 linear equations and hence we need at least 4 pairs of pixels to solve the homography matrix. After obtain the homography matrix, we can apply the matrix on the entire image which will generate a new projective image of the homography.

We can also use more than 4 pairs of matching pixels and use the least squared error to obtain an approximate homography matrix which is less sensitive to outliers (you may choose a matching pair that are actually outliers). 

```python
r, c = imR.shape[:2]
corners = np.array([[0, 0], [0, r], [c, 0], [c, r]])
warped_corners = model_robust(corners)
all_corners = np.vstack((warped_corners, corners))
corner_min = np.min(all_corners, axis=0)
corner_max = np.max(all_corners, axis=0)

output_shape = (corner_max - corner_min)
output_shape = np.ceil(output_shape[::-1])

offset = SimilarityTransform(translation=-corner_min)

wL = warp(imL, offset.inverse, output_shape=output_shape, cval=-1)
wR = warp(imR, (model_robust + offset).inverse, output_shape=output_shape, cval=-1)

bad_trans = ProjectiveTransform()
bad_trans.estimate(src, dst)
wR_bad = warp(imR, (bad_trans + offset).inverse, output_shape=output_shape, cval=-1)
```

![feature matching ransac](homography-result.png)


## Blending & Image Feathering

Now we have two images that are projected into the same image plane. There is one more step to do which is the image feathering. Firstly, we need to compute a distance transform for both images. Then Use boundary distance transforms to blend left and right images (reprojected) into the reference frame where the intensity of each pixel decreases in proportional to the distance to the edges.

```python
from scipy.ndimage.morphology import distance_transform_edt

def boundaryDT(image):
    x, y, z = image.shape
    image_t = image.reshape((x*y), z)[:,0]
    mask_zero = image_t > 0
    image_t[~mask_zero] = 0
    image_t = image_t.reshape(x, y)
    return distance_transform_edt(image_t)

dR = boundaryDT(wR)
dL = boundaryDT(wL)

alpha_R = dR/(dR + dL)
alpha_L = dL/(dR + dL)

i_L = wL*alpha_L[..., None]
i_R = wR*alpha_R[..., None]

point_R = keypointsR[matchesLR[inliers][:,1]][:, ::-1]
point_L = keypointsL[matchesLR[inliers][:,0]][:, ::-1]

point_L = offset(point_L)
point_R = (offset + model_robust)(point_R)
```

![panorama left and right](panorama-left-and-right.png)

And finally we can blend the two images together:

![panorama result](panorama-result.png)
