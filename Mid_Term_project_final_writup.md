# Project: 3D Object Detection with LiDAR Data
## Midterm Report


## Introduction
In this project, we explored 3D object detection using LiDAR point cloud data from the Waymo Open Dataset. The goal was to build a functional perception pipeline capable of converting raw LiDAR range images into 3D point clouds and detecting vehicles using pre-trained deep learning models. We implemented supporting utilities for dataset parsing, sensor preprocessing, visualization, and model evaluation.

For data preprocessing, we constructed pseudo-RGB represnetations (range images) using LiDAR's range, intensity, and density channels. Two normalization methods were tested: min-max and percentile-based. These pseudo-images were used as input to pre-trained detectors such as SFA3D with a ResNet-18 FPN backbone and Complex-YOLO. These models predicted output bounding boxes in 3D space as $(x, y, z, w, h, l, \text{yaw})$ which were transformed from BEV image space to real-world coordinates and visualized alongside camera and point cloud views


## Exploratory Data Analysis (EDA)
To better understand the dataset and its sensor characteristics, we conducted exploratory analysis across multiple segment files from the Waymo Open Dataset. The goal was to visually inspect the LiDAR point cloud and range image data to identify patterns, limitations, and key features relevant to 3D object detection.

### Vehicle Visibility in LiDAR Point Cloud Data
The first part of the analysis focused on evaluating vehicle visibility under different conditions. We manually analyzed a series of frames that highlight both challenging and clear visibility scenarios across diverse scenarios which include vehicles fully visible in the open field of view, partially occluded by other objects or sensor blind spots, and those at long distances where LiDAR returns become sparse. By comparing these robust scenarios, we gained insight into how factors like occlusion, proximity, and sensor angle impact the fidelity of the LiDAR data—key considerations when designing or evaluating object detection models.

#### Poor visibility
#### Blind spots
Vehicles partially occluded by others or near blind spots (e.g., right-rear) were harder to identify due to LiDAR coverage limitations. Examples for reference are as follows:
A vehicle adjacent to the Ego Vehicle is almost entirely occluded in the sensor’s right-side view, demonstrating one such vulnerability in the example below

<img src="Mid_term_project_figures/Jayant_Kumar_Right_Blindspot_speedview_Image_4.png" width="100%" height="100%">

$$
\begin{align}
\textrm{Figure 1. Front-right view showcasing a vehicle nearly fully occluded by a blind spot next to the Ego Vehicle.}
\end{align}
$$


We can clearly make out important road features, e.g., the concrete raised traffic median in this example. However, an adjacent truck-and-trailer combination is only partially visible due to sensor occlusion near the vehicle body.

<img src="Mid_term_project_figures/Jayant_Kumar_Rear_View_Divided_Road_Image_5.png" width="100%" height="100%" >

$$
\begin{align}
\textrm{Figure 2. Rear-view point cloud showcasing only partial visibility of an adjacent truck-trailer due to sensor blind spots.}
\end{align}
$$

#### Good visibility

Vehicles directly in front or to the side of the ego-vehicle appeared clearly in the point cloud, with well-defined rear-ends, headlights, and vehicle bodies. Examples for reference are as follows:

We can clearly identify the vehicles in front of the Waymo ego vehicle in the following point cloud image of the front view of the Ego Vehicle. 

<img src="Mid_term_project_figures/Jayant_Kumar_Front-View-Traffic_Image_1.png" width="100%" height="100%">

$$
\begin{align}
\textrm{Figure 3. Front-view point cloud showcasing clearly visible rear-ends of vehicles ahead of the Ego Vehicle (highlighted in red).}
\end{align}
$$

There are pedestrians (in blue) in the middle of the intersection controlling traffic. The oncoming vehicles to the left of the Ego Vehicle are clearly visible (in red) in the followng scene the vehicle is approaching a controlled intersection. 

<img src="Mid_term_project_figures/Jayant_Kumar_Front_Left_View_Opposing_Traffic_And_People_In_Intersection_Image_2.png" width="100%" height="100%">

$$
\begin{align}
\textrm{Figure 4. Front-left view point cloud showcasing opposing traffic (red) and pedestrians (blue) within a controlled intersection.}
\end{align}
$$


as well to the right of the Ego Vehicle taking a right turn (in red).

<img src="Mid_term_project_figures/Jayant_Kumar_Front_Right_View_Car_On_Right_Controlled-Intersection_Image_3.png" width="100%" height="100%">

$$
\begin{align}
\textrm{Figure 5. Front-right point cloud view capturing a vehicle (red) making a right turn through the intersection.}
\end{align}
$$

#### Distance
Distant objects, such as parked semi-trucks on shoulders, often went undetected due to sparse LiDAR point clouds despite being visible in RGB images. Examples for reference are as follows:
A semi-truck parked on the paved shoulder  went undetected by the LiDAR unit due to poor resolution, despite being labeled in the ground truth in the example below. The semi-truck appears to be clearly distinguishable from the RGB images in this example.

<img src="Mid_term_project_figures/Jayant_Kumar_Front_View_Poor_Visibility_Image_6.png" width="100%" height="100%">

$$
\begin{align}
\textrm{Figure 6. Front-view point cloud where a semi-truck (visible in blue in RGB image) goes undetected by LiDAR on the shoulder.}
\end{align}
$$

This analysis revealed how vehicle visibility degrades with occlusion, proximity, and angle, justifying the need for multi-modal sensor fusion 

### Vehicle Landmarks in LiDAR Range Images
In this analysis, using the intensity channel of LiDAR range images we identified vehicle features that have consistent, highly distinguishable appearances across the frames, making them useful for detection and tracking modules

#### License plates
License plates is highly reflective and are prominent in intensity channels and consistently visible in rear views as bright spots, making them reliable indicators.

#### Windshields
Windshields appeared as `dark patches` / `void` due to low reflectivity. This characteristic absence helps distinguish between different parts of the vehicle structure and even detect vehicle orientation.

### LiDAR Range Image Pre-Processing
Before we move onto the next task, performing inference over the converted BEV image maps, let's briefly discuss the data pre-processing we have done in order to make the range images a bit clearer.

To improve clarity in visualizations and model input, we normalized the intensity channel using:

- **Min-max normalization**: Scales all values between 0 and 255.
- **Percentile normalization (1st and 99th)**: More robust to outliers and saturation effects.

We found percentile-based normalization produced cleaner, more informative range images, particularly for distinguishing vehicle surfaces.


#### Intensity Normalization Formulas

Below are the Python code snippets used for each normalization method:

**Min-max normalization:**
```python
# Min-max normalization on intensity values
scale_factor_intensity = np.amax(ri_intensity) - np.amin(ri_intensity)
ri_intensity_scaled = ri_intensity * np.amax(ri_intensity) / 2
ri_intensity_scaled = ri_intensity_scaled * 255. / scale_factor_intensity
```

**1st and 99th percentile normalization:**
```python
# 1st–99th percentile normalization
ri_min = np.percentile(ri_intensity, 1)
ri_max = np.percentile(ri_intensity, 99)
np.clip(ri_intensity, a_min=ri_min, a_max=ri_max)
ri_intensity_scaled = np.int_((ri_intensity - ri_min) * 255. / (ri_max - ri_min))
```

We found that percentile normalization significantly improved interpretability by limiting the effect of extreme values—especially important for regions with specular reflections like license plates or retroreflective surfaces.

---

### Implementation Details

#### Switching Between Sequence Files

To switch between `Sequence 1`, `Sequence 2`, or `Sequence 3` from the Waymo Open Dataset, update the relevant filenames in `loop_over_dataset.py`:
```python
# Waymo segment file selection
# Sequence 1
data_filename = 'training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord'
# Sequence 2
# data_filename = 'training_segment-10072231702153043603_5725_000_5745_000_with_camera_labels.tfrecord'
# Sequence 3
# data_filename = 'training_segment-10963653239323173269_1924_000_1944_000_with_camera_labels.tfrecord'
```

Then, update the `configs.rel_results_folder` path in `objdet_detect.py` to match the results folder for that sequence. For example, for Sequence 1:
```python
configs.rel_results_folder = 'results_sequence_1_resnet'
```

This will direct output to the correct subfolder, e.g., `results/fpn-resnet/results_sequence_1_resnet/`.

---

#### Executing Specific Tasks

To visualize vehicle visibility in the LiDAR point cloud and compare it against BEV and camera views, modify `loop_over_dataset.py` like so:
```python
exec_data = []
exec_detection = []
exec_tracking = []
exec_visualization = ['show_pcl', 'show_objects_in_bev_labels_in_camera']
```

This opens:
- An Open3D window for 3D point cloud visualization
- An OpenCV window with RGB and BEV overlays

To explore LiDAR range images and intensity patterns:
```python
exec_data = []
exec_detection = []
exec_tracking = []
exec_visualization = ['show_range_image']
```

This launches a frame-by-frame OpenCV viewer rendering the LiDAR range image with intensity values, allowing you to inspect features such as license plates, windshields, and vehicle outlines.



## Inference and Results
We evaluated the SFA3D model with a ResNet-18-based Keypoint Feature Pyramid Network (KFPN) backbone on `Sequence 1` from the Waymo Open Dataset (frames 50–150). Detection results showed high precision, recall, and IoU, with slightly lower accuracy along the z-axis.


The following detection results have been observed for the Sequence 1 clip from the Waymo Open Dataset across (frames 50-150).

```python
# Sequence 1 file used for evaluation
data_filename = 'training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord'
```

### Detection Performance Metrics

Below is the performance visualization for the model on the selected sequence. It includes standard detection metrics such as **Precision**, **Recall**, and **Intersection over Union (IoU)**, as well as histograms for spatial error along the $x$, $y$, and $z$ axes.

<img src="Mid_term_project_figures/Detection-Performance-Metrics.png" width="100%" >

$$
\begin{align}
\textrm{Figure 7. Showcasing Detection performance metrics (precision, recall, IoU) and spatial error histograms for SFA3D model evaluated on Waymo Sequence 1.}
\end{align}
$$

- **Precision and Recall:** Near-perfect scores indicating robust detection consistency.
- **IoU:** High overlap between predicted and ground-truth bounding boxes.
- **Error Histograms:** Low positional errors in $x$ and $y$, slightly higher in $z$ (depth), likely due to reduced LiDAR fidelity for small vertical surfaces or occluded structures.

Despite being trained on KITTI, the model demonstrated strong cross-dataset generalization to Waymo data. While our initial evaluation results look strong, further analysis needs to be performed across other segment files.

### Bounding Box Predictions

We visually assessed the model’s predictions on two representative frames. The bounding boxes predicted by the SFA3D model are shown in red in the BEV (Bird’s Eye View) visualizations. Ground truth labels, when available, are shown in green in the RGB image projections.

<img src="Mid_term_project_figures/Detections-in-BEV-Shown-with-LiDAR-PCL-1_Image_7.png" width="100%">

$$
\begin{align}
\textrm{Figure 8. First evaluation frame showcasing BEV predictions and RGB overlays with 3D bounding boxes.}
\end{align}
$$

<img src="Mid_term_project_figures/Detections-in-BEV-Shown-with-LiDAR-PCL-2_Image_8.png" width="100%">

$$
\begin{align}
\textrm{Figure 11. Second evaluation frame with BEV and camera-space bounding box visualizations.}
\end{align}
$$

These visual results confirm tight alignment of predicted boxes with visible vehicles. Minor drifts in orientation and depth are observed in cluttered or partially occluded scenes, but ground-truth labels have been preserved in their 3D format with respect to the camera sensor space. 

## Closing Remarks

### Alternatives Considered
- Testing different pre-trained architectures .
- Evaluation of other sequence files from Waymo to generalize conclusions.

### Potential Extensions
- Visualize predictions in camera space for deeper calibration validation.
- Conduct fine-tuning of the SFA3D model on Waymo-specific distributions.
- Add comparative benchmarking with other models like Complex-YOLO.



## Future Work

- Visualize BEV and camera-space predictions for more segment files. (Done)
- Fine-tune existing models using domain-specific Waymo data.
- Benchmark against Complex-YOLO and other architectures.
- Evaluate SFA3D across additional Waymo sequences.

