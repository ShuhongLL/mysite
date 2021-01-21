---
title: Computer Vision - Stereo
date: 2021-01-07 20:02:19
tags: [CV, stereo]
katex: true
mathjax: true
photos: ["../images/stereo-cover.PNG"]
---

Stereo is the extraction of 3D information from digital images by comparing the same objects about a scene captured by different cameras located at two vantage points. The information such as depth of the object can be extracted by examining the disparity changes among two images.

<!-- more -->

## Dataset

Use one of the classic datasets from University of Tsukuba. The image, from back to the front, contains a bookshelf (background), a camera, a table, a plaster model and a lamp. Two images were taken from two parallel points, where the left image is referred as the base image (reference) and right image is referred to calculate disparities for each pixel (the pixels that refer to the same object will move leftwards while the camera is moving rightwards).

![stere dataset](stereo-dataset.png)

The ground truth image shows the correct object boundaries and their corresponding depth represented by different color scales.


## Intuition

It is easily to come up with that the closer objects (smaller depths) will have larger disparities during the movement of the camera compared with the further objects (larger depths).

![stere disparity map](stereo-intuition.PNG)

For the object which is in the same depth (e.g. the table and the cans on it), their pixels are very likely to have the same disparities among the two views. Therefore, if there is any approach which is able to figure out the correct disparity for each pixel, then by assigning the same label (color) to pixels that share the same disparity, the depth of each object can be extracted from the scene.


## Window based approach

A single pixel can hardly represent the features of an object. Therefore assign each pixel a rigid window with fixed height and width in the reference image (left image). The intensities within the window represents the weight of the central pixel. In the right image, find a matching window on the same scan line that looks most similar to the reference window. This can be achieved by enumerating disparity within a reasonable range (manually defined) and comparing the sum of squared intensity differences within the windows.

![stere window based approach](stereo-window-based-intensity.png)

Sum of Squared Difference: {% mathjax %}\sum (I(x,y) - I'(x-d,y))^2{% endmathjax %}
For any pixel p compute SSD between two windows for all disparities {% mathjax %}d{% endmathjax %}, where {% mathjax %}d \in [d_{min},d_{max}]{% endmathjax %}, then obtain an optimal {% mathjax %}d_p{% endmathjax %} by optimizing {% mathjax %}d_p=argmin_d SSD(p,d){% endmathjax %}.

This approach can be optimized using dynamic programming which calculate the current summation of intensities and subtract the uncovered region to obtain the sum of intensities within the current window size.

![stere image integral](stereo-image-integral.PNG)

{% mathjax %}
W=f_{in}(br)- f_{in}(bl)- f_{in}(tr) + f_{in}(tl)
{% endmathjax %}

The algorithm has runtime of {% mathjax %}O(I*d){% endmathjax %}, where {% mathjax %}I{% endmathjax %} is the number of intensities and {% mathjax %}d{% endmathjax %} is the enumerated disparities.

![stere SSD](stereo-SSD.png)

The above images show the intensity difference choosing different range of disparities. The shadowed regions represent the pixels that look similar (less intensity difference). A small disparity can easily recognized the further objects in the scene (e.g. the bookshelf) and a large disparity helps to extract closer objects (e.g. the lamp).

Set disparity range.
```python
d_min = 2
d_max = 15
```

Calculate squared differences for different shifts.
```python
def SD_array(imageL, imageR, d_minimum, d_maximum):
    SD = np.zeros((1+d_maximum-d_minimum, np.shape(imageL)[0], np.shape(imageL)[1]))
    for d in range(d_minimum, d_maximum+1):
        rshift = np.roll(imageR, d, axis=1)
        SD[d-d_minimum,:,:] = (imageL[:,:,0] - rshift[:,:,0])**2 +\
                              (imageL[:,:,1] - rshift[:,:,1])**2 +\
                              (imageL[:,:,2] - rshift[:,:,2])**2  
    return SD

SD = SD_array(im_left, im_right, d_min, d_max)
```

Map summation of squared differences to disparity map (image)
```python
def SSDtoDmap(SSD_array, d_minimum, d_maximum):    
    dMap = np.full(np.shape(SD[0]), d_minimum)
    inf_mask = np.full(np.shape(SD[0]), False)
    mMap = np.amin(SSD_array, axis=0)
    for i in range(d_maximum-d_minimum+1):
        dMap = np.where(SSD_array[i]==mMap, i, dMap)
        inf_mask = inf_mask | (SSD_array[i] == INFTY)
    dMap[inf_mask] = 0
    return dMap
```

summation in a certain window size.
```python
INFTY = np.inf

def windSum(img, window_width):
    win_img = np.zeros(img.shape)
    # shift bottom/right value
    bshift = window_width//2
    # shift top/left value
    tshift = (window_width+1)//2
    simg = integral_image(img)
    br = np.roll(np.roll(simg, -bshift, axis=1), -bshift, axis=0)
    tl = np.roll(np.roll(simg, tshift, axis=1), tshift, axis=0)
    bl = np.roll(np.roll(simg, tshift, axis=1), -bshift, axis=0)
    tr = np.roll(np.roll(simg, -bshift, axis=1), tshift, axis=0)
    win_img = br - bl - tr + tl
    # bottom right margin value
    bmargin = window_width//2
    # top left margin value
    tmargin = window_width//2 if window_width%2 == 1 else (window_width-1)//2
    # top margin
    win_img[:tmargin+1,:] = np.inf
    # left margin
    win_img[:,:tmargin+1] = np.inf
    # bottom margin
    win_img[img.shape[0]-bmargin:,:] = np.inf
    # right margin
    win_img[:,img.shape[1]-bmargin:] = np.inf
    return np.abs(win_img)
```

Find the optimal disparity.
```python
def Dmap_Windows(imageL, imageR, d_minimum, d_maximum, window_width):
    SD = SD_array(imageL, imageR, d_minimum, d_maximum)
    SSD = np.zeros(np.shape(SD))
    for Delta in range(1+d_maximum-d_minimum):
        SSD[Delta] = windSum(SD[Delta], window_width)
    return SSDtoDmap(SSD, d_minimum, d_maximum)
```

![stere window based result](stereo-window-based-result.png)

The chosen size of the window also has effect to the result as larger window size will blur the result disparity map and smaller window size will generate more noise.


## Scan-line approach

The scan-line approach seeks the shortest paths for pixels in the same scan line. An {% mathjax %}n * n{% endmathjax %} grid graph can be built using each scan line (n is the number of pixels located in the scan line). Horizontal and vertical edges on this graph describe occlusions, as well as disparity jumps (discontinuities). The diagonal lines on the graph represents disparity levels (shifts) that can be seen as depth layers.

![stere scan line graph](stereo-scan-line-graph.PNG)

The loss function for a single pixel in the scan line can be expressed as the combination of the cost of photo consistency and the cost of the spatial coherence
{% mathjax %}
E(p)=|I_p - I_{p+d_p}| + w|d_p - d_{p+1}|
{% endmathjax %}
where the photo consistency represents the intensity change and spatial coherence penalize the disparity jump. The overall loss can be expressed as
{% mathjax %}
E(d)=\sum_{p \in S} D_p(d_p) + \sum_{p \in S} V(d_p, d_{p+1}) = \sum_{p \in S} |I_p - I_{p+d_p}| + \sum_{p \in S} w|d_p - d_{p+1}|
{% endmathjax %}
The loss can be optimized using Viterbi algorithm since the graph does not contains any loop.

Implementation of Viterbi Algorithm.
```python
def traverse_disparity(SD, row, d_min, d_max, w, normalization='standard', threshold=0, sigma=0):
    K = d_max - d_min + 1
    T = SD.shape[2]
    T1 = np.empty((K, T))
    T2 = np.empty((K, T))
    
    # KxK matrix
    index = np.array([np.arange(1, K+1)]*K).T
    # record E_bar[i]
    T1[:, 0] = 0
    # record i-1
    T2[:, 0] = np.arange(1, K+1).T
    
    # forward pass
    for i in range(1, T):
        if normalization == 'standard':
            normal = w * np.abs(index - [T2[:, i-1]]*K)
        elif normalization == "quadratic":
            normal = w * np.square(index - [T2[:, i-1]]*K)
        elif normalization == "robust":
            normal = w * np.square(index - [T2[:, i-1]]*K)
            normal[normal > threshold] = threshold
        elif normal == "gaussian":
            p = -np.square(SD[1, row, i]) / (2 * sigma**2)
            wp = w * np.exp(p)
            normal = wp * np.abs(index - [T2[:, i-1]]*K)
        m = np.array([SD[:, row, i]]*K).T + normal
        T1[:, i] = np.min(m, axis=0)
        T2[:, i] = np.argmin(m, axis=0)
    
    x = np.empty(T)
    x[-1] = np.argmin(T1[:, T-1]) + d_min
    
    # backward pass
    for i in reversed(range(1, T)):
        x[i - 1] = T2[int(x[i] - d_min), i] + d_min
    return x
```

Compute scan line disparity map.
```python
_, R, C = SD.shape
R = SD.shape[1]
d_scan = np.zeros((R, C))

for row in range(R):
    d_scan[row,:] = traverse_disparity(SD, row, d_min, d_max, 0.7, normalization='standard')

```

The following images show the result applying scan line approach.
![stere scan line result1](stereo-scan-line-result1.png)

A better result can be obtained by replacing the original images with the computed SSD maps so that each pixel will also contains information of adjacent pixels within the same window. This helps to reduce the noise.
![stere scan line result2](stereo-scan-line-result2.png)


## Full Grid approach

Full grid stereo is a more advanced approach using fully connected graph. In scan line stereo, the loss function is computed only using the pixels within the same scan line. To consider the contiguous among scan lines, vertical edges are added.

![stere full grid graph](stereo-full-grid-graph.PNG)

Now, the loss function of a single pixel is not only decided by pixels on horizon, but also related to upper and lower pixels. Viterbi algorithm (DP) can be no longer applied since the graph apparently contains loops. Instead, apply graph cut to optimize the loss function. The weights of neighborhood edges on a single layer of the graph represents the spatial coherence (disparity jump). Multiple layers are required to represent the photo consistency (intensity change). The cost function becomes
{% mathjax %}
E(d)=\sum_{p \in G} D_p(d_p) + \sum_{p, q \in N} V(d_p, d_q)
{% endmathjax %}

![stere graph cut](stereo-graph-cut.PNG)

The orthogonal edge represents the cost of photo consistency of the current pixel refers to a specific disparity change to the refer image. For example, the edge at the bottom layer represents the intensity difference between the current pixel and the pixel at the minimal disparity; the edge at the top layer represents the intensity difference between the current pixel and the pixel at the maximum disparity. After defining the 3D graph and applying max-flow, the cut occurred at the orthogonal edges is the result disparity for each pixel.

Build full gird and apply graph cut. Use a python wrapper library [maxflow](http://pmneila.github.io/PyMaxflow/maxflow.html).
```python
import maxflow

def graph_cut_3D(im_left, SD, d_min, d_max, w):
    num_rows = im_left.shape[0]
    num_cols = im_left.shape[1]

    g = maxflow.GraphFloat()
    depth = d_max - d_min + 1
    
    # depth x height x width
    nodeIds = g.add_grid_nodes((depth, num_rows, num_cols))
    structure = np.array([[[0, 0, 0],
                           [0, 0, 0],
                           [0, 0, 0]],
                          [[0, 0, 0],
                           [0, 0, 1],
                           [0, 1, 0]],
                          [[0, 0, 0],
                           [0, 1, 0],
                           [0, 0, 0]]])
    g.add_grid_edges(nodeIds, structure=structure, symmetric=True)

    # add n-links for spatial coherence
    structure_x = np.array([[0, 0, 0], [0, 0, 1], [0, 0, 0]])
    structure_y = np.array([[0, 0, 0], [0, 0, 0], [0, 1, 0]])
    
    right = np.roll(im_left, -1, axis=1)
    below = np.roll(im_left, -1, axis=0)
    n_right = (im_left[:,:,0] - right[:,:,0])**2 +\
              (im_left[:,:,1] - right[:,:,1])**2 +\
              (im_left[:,:,2] - right[:,:,2])**2
    
    n_below = (im_left[:,:,0] - below[:,:,0])**2 +\
              (im_left[:,:,1] - below[:,:,1])**2 +\
              (im_left[:,:,2] - below[:,:,2])**2
    
    g.add_grid_edges(nodeIds, weights=n_right, structure=structure_x, symmetric=True)
    g.add_grid_edges(nodeIds, weights=n_below, structure=structure_y, symmetric=True)

    # add n-links for photo consistency
    structure_z = np.array([[[0, 0, 0],
                             [0, 0, 0],
                             [0, 0, 0]],
                            [[0, 0, 0],
                             [0, 0, 0],
                             [0, 0, 0]],
                            [[0, 0, 0],
                             [0, 1, 0],
                             [0, 0, 0]]])
    
    # weight is multiplied here
    g.add_grid_edges(nodeIds, weights=SD*w, structure=structure_z, symmetric=True)
    
    # find maximum weight of n-link edges
    max_right = np.max(n_right)
    max_below = np.max(n_below)
    max_depth = np.max(SD)
    max_edge = np.max([max_right, max_below, max_depth])

    # assign t-links
    t_source = np.zeros((depth, num_rows, num_cols))
    t_sink = np.zeros((depth, num_rows, num_cols))

    t_source[0] = np.ones((num_rows, num_cols)) * max_edge * 4
    t_sink[-1] = np.ones((num_rows, num_cols)) * max_edge * 4
    g.add_grid_tedges(nodeIds, t_source, t_sink)

    g.maxflow()
        
    return g, nodeIds
```

Apply full grid stereo and return the disparity map.
```python
def full_grid_stereo(im_left, SD, d_min, d_max, w):
    flow_g, nodeIds = graph_cut_3D(im_left, SD, d_min, d_max, w)
    num_rows = im_left.shape[0]
    num_cols = im_left.shape[1]

    # traverse each depth of the full grid
    # and find out where the grid was cut
    disparity_map = np.zeros((num_rows, num_cols))
    cur_disparity = ~np.ones((num_rows, num_cols), dtype=bool)

    for d in range(d_min+1, d_max+1):
        index = d - d_min
        cur_depth = flow_g.get_grid_segments(nodeIds[index])
        # compute xor to find any label change
        cut = cur_disparity ^ cur_depth
        disparity_map[cut] = d - 1
        cur_disparity = cur_depth
        
    return disparity_map
```

The following images show the result of applying full grid stereo.
![stere graph cut result1](stereo-full-grid-result1.png)

Apply full grid approach on other datasets ([Middlebury](https://vision.middlebury.edu/stereo/data/]))
![stere graph cut test](stereo-test.png)

The corresponding results show as follows.
![stere graph cut test result](stereo-test-result.png)