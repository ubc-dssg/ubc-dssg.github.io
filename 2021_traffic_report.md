layout: page
title: "Unbiased Mobility Project"
permalink: /unbiased_mobility/

[TOC]

# 1. Introduction

Academic communities and public sectors have been using mobility and traffic data to address social issues  (i.e GDP prediction, environmental impacts, crime, etc). One method to measure mobility and traffic state is by analyzing a series of pictures and footage from traffic cameras installed at fixed locations.

In order to utilize such data for analysis, a key starting point is to evaluate current approaches of retrievement, identify potential biases in the models, and make statistical inferences with consideration of existing biases. Inductive-loop traffic detectors are accurate yet rather expensive as it requires to be installed in the pavement. In the contrary, modern computer vision technologies' recent breakthroughs in object detection have enabled us to use the much lower-cost data from public accessible traffic cameras. However, such methods are less reliable for various reasons, such as high storage cost of highly granular data, low image quality, variability of camera types and roadtypes, and fundamental limitations of object detection accracy. In Section 3, we explore these caveats and identify some biases introduced into the traffic flow data due to these problems. With the biases acknowledged and potentially compensated, we wish to provide a pilot study on the usage of traffic data.

As the COVID-19 pandemic unfolds, traffic data has helped a lot with indicating the mobility level of certain business areas, providing a reflection of business activities. The [Dallas Fed Mobility and Engagement Index(MEI)](https://www.dallasfed.org/research/mei), an aggregation of several relevant covariates based on geo-location data collected from a large sample of mobile devices, has been used to give insights in post-pandemic economic recession. However, most of the existing studies are dedicated to finding a high-level trend, while our project concentrates on a more localized and granular analysis of the business activities in the post-pandemic era with the target population being the city of Surrey.

Throughout this project, we will discuss our data sources and methods of utilization. Next, we will discuss the computer vision pipeline and address its biases as well as other issues that were found. Finally, we will showcase an R shiny application which supports integrative and accessible analysis with intuitive interface, to provide insights for the general public.


# 2. Data Sources

### 2.1. Primary Data Sources

#### 2.1.1 Image Dataset captured from traffic cameras
For representative image examples, we extracted 60-min gap raw JPEG images from public traffic cameras dated from December 1st to 31st 2020 at 2 locations (104 Ave & 140 St, 104 Ave & City Hall Driveway, Burnaby). For higher granularity data to investigate changed within short period such as within a day, We have 2-min gap raw JPEG images on May 2nd 2021 for the camera located at 104 Ave & 140 St.


#### 2.1.2. Vehicle Count For Traffic Cameras

This is the mean dataset of analysis in the following sections.  
The Surrey vehicle count dataset consists of 245,499 records extracted from traffic footages using Computer Vision methods (details introduced in Section 3),taken at 1-hour interval from December 01, 2020 to December 31, 2020 for 341 camera stations across Surrey, BC, Canada. They are associated with coordinated in latitude and longtitude. Variables of interests include counts for count for car, bicycle, bus, person, motorcycle, truck (See Figure 1).
 
![dataset example](https://i.imgur.com/GaT63zE.png)

*Figure: 1 Preview of the initial dataset given for the month of December 2020*

 
### 2.2. Other Data Sources


#### 2.2.1. Business
A list of all registered business data within the city of Surrey was provided. Among the dataset, we excluded 5% of the entries that did not contain a buisness name as they were registered as a home occupation. The important entries that were taken into consideration were the name of the business, its category, its status (whether if it is still active or terminated) and the coordinate locations. Unfortuantely, it did not contain any data on earnings as our initial intention was to investigate on how the trends in traffic pattern affect public business revenues given the COVID-19 pandemic. Nevertheless, we used the nearby traffic cameras within close proximity of businesses as well as the intensity of the traffic activity to guage earning performance. In other words, we operated under the notion that as traffic intensity increases, so does the performance of a business.
 
The status of the business was divided into 10 stages of approval from initial draft to active status. In the dataframe, 80% of the businesses were active and therefore, we focused on currently active businesses and disregarded 20% of businesses as they were either in various stages of approval, or were terminated or rejected. Overall, 25% of the dataframe was excluded from out analysis

#### 2.2.2. COVID-19 Data
Provincial COVID-19 lab data dating from Janurary 2020 to July 2021 was taking from the BC CDC website. Each entry included the date and the health authority region. Within the date and the health authority region, there are also data entries that highlight the type of tests. Specifically, the tests can be broken into 4 categories:

- The number of new tests which describes the cumulative number of new COVID-19 tests performed within the last 24 hours prior to the update of the ArcGIS dashboard found on the BC Centre for Disease Control
- The number of total tests which describes the cumulative number of COVID-19 tests since testing commenced mid-January
- The positivity rate which is calculated as an average of a 7 day period as a ratio of test specimens that were confirmed positive to the total amount of specimens being tested

- The turn around time which is calculated as the daily average time between a collection of a specimen and the report of a test result.

We used this data source as a confounding variable to determine if the relationship between the business activities and the traffic patterns within the city of Surrey was influenced by the COVID-19 pandemic. As such, we focused on the cases within the Fraser Health Region (as that is where the city is located in) as well as the dates in October and December to see the change in business activity. 


#### 2.2.3. Day and Night time
The detection algorithm is severely vulnerable to vehicle's **over-exposed headlight issue** during night time. To help reduce the detection biases due to this reason, we applied several image-preprocessing methods(i.e. gamma adjustment, histogram equalization, etc.) with reasonable parameters to the filtered night-time image dataset to see the overall performance. 

The night-time image dataset was generated by filtering out nightime images within a specific time period. Our initial step was to find the dusk and sunrise times for each day in the month of December using the `astral` package as we wanted to compare the difference in the biases between dusk, the darkest part of the evening, and sunrise, where the camera sees initial light. Next, we took the average dusk and sunrise times of all 31 days of December. From there, we filtered out the nighttime image datset between our averaged dusk and sunrise time obtained from the previous step.

#### 2.2.4. Bike Lanes
A dataset of all bike routes within the city of Surrey current to 2020 was provided. The bike routes contained 7 different type of lanes and current status of the bike routes were placed in 3 categories: 
- `In Service` for currently operating routes, 
- `Proposed` for future or planned bike routes for development
- `NA` for bike routes that do not fall into either the `In Service` nor the `Proposed` bike routes.

The main use of this dataset was to utilize it in our later development of our R Shiny app for traffic analysis. Since the initial datasets provided from our sponsor were from the month of December, we decided to only include serviceable bike routes. Therefore, we excluded 11% of the dataset as they were either listed as `Proposed` or `NA`.

#### 2.2.5. Boundaries 
A`GEOJSON` datafile that consists of the administrative boundary lines of the city of Surrey along with its neighbourhoods of Guildford, Whalley, City Centre, Cloverdale, Fleetwood, Newton,South Surrey, and Whalley was provided.



# 3. Computer Vision

## 3.1 Pipeline Overview
![data pipeline](https://i.imgur.com/KyrXbCS.png)


In this pilot study, snapshots are collected from 341 camera stations in city Surrey. Computer vision pipeline extracts information of vehicle counts of different categories from these snapshots and output count dataset. Specifically, it detects the location of different vehicles in the image, then translates them into counts of objects of interest.

## 3.2 YOLO

![](https://i.imgur.com/lwKFqIE.png) 
*State of the art of object detection algorithms on COCO text dataset by July 2021*

YOLO (You Only Look Once) is one of state-of-the-art object detection algorithms. The algorithm localizes bounding boxes locations and predict corresponding class probability by regression using a single neural network from the entire target image/frame. The simple yet elegant algorithm achieves the heretofore best performance on real-time object detection in major test dataset. We chose YOLO as our object detection algorithm in the project, as there is future possiblity of higher granularity data (video stream of traffic cameras instead of snapshots with minutes' interval), and YOLO is known for high FPS (frame per second) object detection compared to other CNN methods.

We explored popular version of YOLO, including YOLOv3 (by [Joseph Redmon](https://pjreddie.com/)), YOLOv4 (publised in April 2020, [arXiv here](https://arxiv.org/abs/2004.10934)), and YOLOv5. The original YOLOv3 is most popularly used, implemented in various frameworks and open source libraries. We initially used [cvlib](https://www.cvlib.net/) implementation in the server. This pipeline generated majority of count dataset used in later statistical analysis. YOLOv4 using darknet implementation was also tested on the same image dataset and it shows fair improvement on accuracy and speed on a dozen sample images (see Figure 2). Both models were trained using MS COCO dataset. We dropped YOLOv5 as no academic paper had been published and controversies remained around its validity.

![Comparison between YOLOv3 and v4 on a sample image](https://i.imgur.com/AoadLqi.png)
*Figure 2: Comparison between YOLOv3 and v4 on a sample image*


## 3.3 Undercount Issue and Post-detection Correction Model

One important thing to recognize about our dataset is that, public accessible traffic cameras produce low resolution images due to both hardware constraint and privacy concerns. Government stakeholders intentionally blur some of them to anonymize pedestrian faces and plate numbers. 

The other feature of the images is that they usually have vehicles suffer from high occlusion. When vehicles line up at the intersection, their semantic features heavily overlaps with each other. Thus the object detection faces challenge to correctly detect vehicles especially when there is a long line-up. On the other hand, it is able to detect correctly at far distance when the traffic is free of flow and vehicles are more far apart from each other. 

Hence, the count data generated using the current pipeline has two major issues. 

- **Saturation**: the maximum vehicle count at a moderate busy intersection is almost *capped at around 15* where one can easily find snapshot with 20+ cars under visual check.
- **Inconsistency**: when the road is busy, the vehicle counts shows obvious fluctations as line-up resulting from red light could appear out-of-sync from the snapshot frequency. 

> “Consecutive samples at a given camera can give very different impressions of traffic depending on the state of the traffic intersection. For example, during rush hour, we may see twenty cars waiting at an intersection in one image, then two in the next image if we are witnessing the end of the signal clearing the traffic queue.” (Snyder, 2019)

This might not be an issue with higher frequency snapshots (the current dataset has per 2 minute), but introduces inconsistentcies when the images used for statistical inference are sampled with a lower frequency (See Figure 3).

An example is given below.

![undercount example](https://i.imgur.com/WAhf8Jw.png)  
*Figure 3: Comparison between queue (left) and free of flow (right) images shown*

Low **detected count** does not necessarily mean low traffic. In the below examples, the entire queue incoming is hardly captured. Also, when switching between free-of-flow (**F**) and queue (**Q**) traffic state, the pipeline undercounts inconsistently. The top image should count more cars than the bottom one while the detected count indicates the opposite.

Besides, the detection also failed during nighttime due to headlights, overexposure and reflections.

These biases impairs statistical analysis afterwards. Thus we propose a mitigation strategy via a post-detection correction model.  

## 3.4 Manual Label Dataset

We manually labeled 720 images of one representative camera of intersection for 24 hours, from May 2nd to May 3rd, 2021. Note that due to limited time and manpower, we did not label the images for bounding boxes and classes, and we do not intend to retrain or fine tune the object detection model. Instead, we labeled them for the following data:

- Vehicle counts of categories of interest.
- State of traffic, defined by 'Q' (queue) when more than 3 cars line up at the intersection, otherwise 'F' for free-of-flow (see traffic state classification from [STREETS Dataset](https://experts.illinois.edu/en/datasets/data-for-streets-a-novel-camera-network-dataset-for-traffic-flow-2); see Figure 4).

![Queue and Free-of-flow example](https://i.imgur.com/WRQyUkM.png)  
*Figure 4: Queue vs Free flowing traffic comparison*

State of traffic is usually a variable of interest in traffic network modeling and flow prediction. Given the insights from the previous section, we believe it would be a relevant covariate in the correction model.

The human labelling results shed light on the **difficulty** of human annotation on vehicle identification. *Cohen's Kappa score* measures the interrater reliability (sample size of 3). It can be mapped empirically to label accuracy. As a result, car count yields around +0.62 which maps to around 0.8 accuracy, which is fairly high. In the contrary, Q/F category gives +0.31 which is fairly low and indicates that it is unreliable. The difficulty measure not only quantifies the reliability of our "ground truth", but also estimates the cost-benefit for human annotation task in the future when we need training data. 

A comparison between human labeled car counts (dark blue) and automatically detected car counts (light blue) is shown below, plotted over a 24 hour period at the intersection camera (see Figure 5).
![human labeled car counts (dark blue) and automatically detected car counts (light blue)](https://i.imgur.com/M2DFkra.png)

*Figure 5: Human labeled car counts (dark blue) in comparison with automatically detected car counts (light blue)*

From Figure 5, we can clearly see the change of car counts varying inside and outside rush hours. It drops at midnight and early morning, reaches the first peak at around 8 to 9 am, then second peak around lunchtime, with another significant increase in the afternoon around 3 o'clock, reaches the highest at around 5 pm which is when people get off work.

Two key observations from the comparison:
- The bias is always undercounting. Our algorithm does not hallucinate.
- The bias is more significant during rush hours, when there are more cars, whether detectd or labeled as the detection does capture the general trend. On the contrast, bias is neglibile during low traffic hours for example midnight.

## 3.5 Post-detection Correction Model

In order to mitigate the undercounting issue, we propose a post-detection correction model.

We do not retrain or fine tune the computer vision model mainly due to constraint on time and manpower for annotation. More important, the goal of our project is not to improve model accuracy on detection and localization, but to capture numerical trends.

Given the insights from the plot, we assume, the biases in car counts depends on, first, the traffic density levels, second, the state of traffic, defined by Q and F. Queue and high-count manifest a crowded scene in which the detection undercount significantly. If we look at the distribution under different traffic states, under state ‘Q’, the human labeled count deviate from detected count more and is noisier (See Figure 6).

![pairplot](https://i.imgur.com/tcGswpO.png)

*Figure 6: Distribution plots under different traffic conditions: Queue (Q) and Free-of-flow (F)*

Providing these assumptions, one naive model would be for we check whether it is Queue or free of flow, then fit different models for the two scenarios.

$$ Count = Queue? Fitted OLS model : Detected car count $$

R-CNN methods are used in literature to detect for Q/F status.

The OLS result shows that Q/F categorical variable is a significant predictor and verified our proposition. It gives a Root Mean Square Error (RMSE) of 1.5, which is not great but fair compared to our data ranging from 15 to 25. 

 In this pilot study we include the model as a feature in the unbiased mobility app, to produce count data after adjustment.

This model is highly limited as our training data is a single camera footage. Although we selected the camera to be as general as possible (moderate car counts, distribution of different vechiles, intersection); we expect the model itself to vary depending on camera perspectives and road types, which leads to the discussion of camera metadata.

## 3.6 Other issues discovered

We also observed some corrupted images and repeated images potentially due to transmission errors or camera failures. There are 3 cases of such corruption in our dataset, around 0.2%.

The last key insight from this study is the importance of **camera metadata**. Cameras have different perspectives, installed at different height with varying field of views, and distributed at different road types with different number of lanes for example intersections, highways and residential areas (See Figure 7 below). A direct consequence is that the same number of vehicle counts could mean very different things for camera A and B since we do not model it as real traffic flow.  This presents challenges later in our statistical analysis.

![metadata](https://i.imgur.com/xkidHNF.png)
*Figure 7: Various types of traffic data shown in everyday traffic cameras*


## 3.7 Nighttime Detection 

Another major issue we came across is the vehicle headlight glares/over-exposure in nighttime images, which made it even more difficult for the detector to capture ample vehicle features to make the true recognition. One tangible solution would be applying an image pre-processing(i.e. filtering) to the raw image before feeding into the detection algorithm. 

One of the most intuitive approaches suggested by some [previous work](https://www.ijariit.com/manuscripts/v4i2/V4I2-1789.pdf) is to adjust the **intensity contrast** through either [*Gamma Correction*](https://docs.opencv.org/3.4/d3/dc1/tutorial_basic_linear_transform.html) or [*Histogram Equalization*](https://docs.opencv.org/3.4/d4/d1b/tutorial_histogram_equalization.html). However, it is not guaranteed that an enhancement in light contrast would help with the undercounting; It could be the opposite, depending on the area's local intensity level. For example in our case, a lower contrast is usually more applicable(i.e. $\gamma<1$), though finding the best parameter would require more training images as well as computing power to run detections on a batch of images. Besides, other approaches like [*Blob Detection/Masking*](https://learnopencv.com/blob-detection-using-opencv-python-c/) and [*In-painting*](https://docs.opencv.org/4.5.2/df/d3d/tutorial_py_inpainting.html) were also experimented and found to help locate and fine-tune the over-exposed blobs; While on the other hand, it could blur out some key features of a vehicle, thus worsening the problem. 

That being said, there hasn't been found a universal solution at the pre-processing stage that could improve the undercounting for all types of nighttime images, yet further experiments could be conducted in **intensity adjustment** with more potential parameters.



# 4. Statistical Analysis

## 4.1 Camera Comparisons

### 4.1.1 Goal Revision

The initial objective of the project was to identify a correlation between camera locations and traffic states. The original proposal states that:

> *Cameras are installed near locations with heavy traffic. This leads to a biased dataset and exaggerates nearby mobility levels due to [preferential sampling](http://www.unavarra.es/metma3/Papers/PDFS_POSTER/Menezes.pdf).*


However, it has been later found out **not** being the problem in our scenario, due to the facts that: 
* We are mostly interested in point of interest along the traffic.
* We **do not** interpolate traffic density off the lanes or areas far from the sampling locations. 

Our new objective is not entirely deviated from the original goal, yet shifting towards a comparison between cameras. We are interested in pairs of cameras with **high spatial correlations** and conduct **comparisons** of their vehicle counts to see if they share similar traffic states concurrently. **AM(7-10**) and **PM(16-19)** rush hours are targeted due to intense traffic activities over the periods. The vehicle count was only specific to car count since it is the majority of all vehicle types, and it was to dissipate any random effect introduced by vehicle types.

To be specific, the pair of spatially-correlated cameras are identified as either:

- Two cameras installed on the **same intersection(SI)**
- Two cameras which are **spatial nearest neighbors(NN)**


### 4.1.2 Data Processing
Our target population is all intersections(i.e. street addresses) in City of Surrey with at least two cameras installed. In order to downsize the problem scope, we chose 10 sampling point-of-interests(POI's) among all 32 such intersections, including the followings (see Figure 8):

* 96 Ave And King George Blvd (near Surrey Memorial Hospital) 
* 104 Ave And City Parkway (near Central City)
* 68 Ave And Fraser Hwy (near Business area)
* 72 Ave And 120 St (near Shopping Center)
* 98 Ave And King George Blvd (near Intersection of Fraser Hwy and King George Blvd)
* 96 Ave And 152 St (near Johnston Heights Church)
* 88 Ave And King George Blvd (near Bear Creek Park)
* 72 Ave And King George Blvd
* No 99 Hwy And King George Blvd
* 16 Ave And 152 St (near White Rock business area)



<img src="https://i.imgur.com/yfoMcRW.png"         alt="drawing" style="display: block; margin: auto; width: 40%;"/>


*Figure 8: selected POI's and their Nearest Neighbors*

First, the selected intersections were grouped by `street_address` to fetch all the SI camera pairs, whereafter the `slice()` was used to separate the pairs. A new categorical variable was then created to classify the `Pair 1` and `Pair 2` cameras. 

On the other hand, the data for the NN cameras were retrieved through invoking the `nngeo::st_nn()` method on the `Pair 1`'s `sf` object and its complementary set in `surrey_desc`, and the results were merged to the dataset as `Pair 3`. 


### 4.1.3 Paired t-test and Two-sample t-test
The comparisons between the two cameras in the 10 pairs were conducted through a **paired** t-test for the **SI** cameras and a **two-sample** t-test for **NN** cameras. Both t-tests were two-sided and the hypotheses were:

* $H_0$: There is no significant difference in car count between the two camera groups($\mu_d=0$)
* $H_a$: There is a significant difference in car count between the two camera groups($\mu_d\neq0$)

In terms of the potential covariates that could explain the differences in camera pairs, several key factors include camera metadata, time and locations. The major focus was on the **temporal** factor, and in order to investigate any temporal pattern in the pairs' differences, we used the filtered hourly data during **AM(7-10)** and **PM(16-19)** rush hours to run the two comparison tests, assuming a relatively higher traffic volume over the periods. 

Prior to the t-tests for comparison, we also carried out a chi-squared test to examine if there's a statistically significant dependency between the two groups(`Pair 1` and `Pair 2`) and the two temporal levels(`AM` and `PM`). The outcome was positive and therefore, we were able to make the assumption that the `AM` and `PM` categorical variable plays a significant role in explaining the cross-camera differences.


### 4.1.4 Preliminary Results
#### Temporal: Greater Differences during PM Rush Hours
The preliminary results have shown that for SI cameras, there was no significant difference($p$-value= $0.1286$) found between the pairs during AM rush hours, whereas a significant difference($p$-value<$2.2e^{-16}$) was found during PM rush hours; Meanwhile, for the NN cameras, there were significant differences($p$-value<$2.2e^{-16}$ found between the pairs during **both** AM and PM rush hours (See Figure 9).



![](https://i.imgur.com/5HN26ln.png)
*Figure 9: Individual results of the paired t-tests for all 10 sampling SI camera pairs*

Overall, a smaller difference was found in the car count between SI cameras than those of NN cameras. The difference in car count has also been found to have a close relationship with the temporal variable, particularly having greater differences occuring over the **PM** rush hours.


### 4.1.5 Individual Camera Comparisons

Having some general insights from the preliminary results, we were then interested in conducting a more in-depth comparison for individual camera pairs and an integrated analysis from perspectives including temporal, camera metadata, and hot-spot areas. As for the use cases, there are two major ones: (1) The ideal case where the two cameras are perfectly comparable; (2) A common but **not** ideal case is where the two cameras are in interplay of other covariates, along with the **camera metadata**. An example are the two cameras installed at 56 Ave And King George Blvd intersection, which will be discussed in the following section.

#### Case Study: 56 Ave And King George Blvd
We selected the 56 Ave And King George Blvd intersection to represent the incomparable case, and targeted at camera 103 and 104, which are recognized as SI cameras facing opposite directions with 103 facing North and 104 facing South. This is an important intersection as it captures the highway traffic for commuters and thus, is a justifiable area for analyzing traffic pattern during rush hours. 

As same as before, we did a paired t-test for the two cameras. As seen here, there are more car counts captured in 103 than 104, and even though both differences were significant, a more significant one was found in PM rush hours (See Figure 10).



<div style="display: flex; align-items:center;">
<img src="https://i.imgur.com/o15ZRlr.png"         alt="drawing" style="width: 50%;"/>
<img src="https://i.imgur.com/EZfcOkG.png"         alt="drawing" style="width: 50%;"/>
</div>

*Figure 10: Car count's Comparison of 103(red) and 104(blue) during AM(left) and PM(right) rush hours*

Based off the two key observations, we did a source of analysis with the help of the in-app features. 

#### 103 captures more car count than 104

One apparent reason why 103 has captured more car count than 104 is that the cameras' perspectives are drastically different, which could be eaily told from the real-time images. Additionally, this is a one-way traffic lane from North to South, thus 103 would typically capture queues and 104 would then capture mostly flows.

#### More significant differernce during PM rush hours

A more significant difference found in PM rush hours could be mainly explained by more people traveling back home from workplaces during the PM rush hours (See Figure 11).

![](https://i.imgur.com/Msj4s4i.png)
*Figure 11: Real-time images of camera 103 and 104*


#### Weekend Effect
One may also wonder if there would be a more consistent rush hour pattern for weekdays-only data, or in other words, any increase in differences. Since we are filtering on weekdays, the sample size is shrinked. So here we used the *Cohen’s d score* to compare the two camera groups which is similar to the t-tests but is standardized and thus, not subjective to sample size changes. We’ve seen that there is a slight increase in d-values, meaning more consistency during weekdays-only rush hours and noticeably PM rush hours having a very large effect size($\approx1.23$) during weekdays (See Figure 12).

<div style="display: flex; align-items:center;">
<img src="https://i.imgur.com/gtxVMtt.png"         alt="drawing" style="width: 70%; margin: 5px;"/>
<img src="https://i.imgur.com/aWwNxtf.png"         alt="drawing" style="width: 25%; margin: 5px;"/>
</div>

*Figure 12: Cohen's d scores for Overall and Weekdays-only data during AM and PM rush hours*

### 4.1.6 Limitations and Potentials
Due to the lack of **camera metadata** and a **small sample size** with **sparse** data points, it was hard to investigate any dependency on the metadata covariates nor any spatial pattern over the entire Surrey. Thus, with more relevant data, it would be much easier to identify all comparable camera pairs, making the comparison analysis more valid and less prone to noises from other confounding factors. Furthermore, the extra information may also facilitate the **traffic flow modelling**. Since the footprint area captured by each camera varies across different angles, heights, zoom levels, etc., it would also pose potential biases in classifying queue(Q) or flow(F) without considering those attributes.


## 4.2 Correlation of Traffic Activity and COVID-19 Pandemic Activity

In addition to investigating high spatial correlations and comparisons of vehicle counts, we were also interested in investigating whether the COVID-19 pandemic had a
confounding effect on the relationship between the economic recovery of small businesses in Surrey and nearby traffic activity. We also would like to observe the state of the traffic activity and predict the effects of it on the COVID-19 positivty rates in the near future. We hypothesized that if the COVID-19 positivity rates were able to be predicted, then we can also indirectly forecast upcomming municipal or provincial lockdowns which, we can also forecast possible economical disruptions. 

To do this, we used the entire car counts of 2020 as other vehicle types had limited data to draw conclusions from. However, due to the limited internal support from our sponsors, we were not provided with our intended dataset. However, the sponsors were supportive enough to construct the initial visualization and the statistical hypothesis test by utilizing their own technical teams and have the results reported to us. 

We will narrate the methods in this report as if we conducted the tests ourselves. However, all narration in the methods section were instructions to the sponsor's technical team to conduct the correlation analysis.   

 
### 4.2.1 Methods
#### Initial Visualization
To visually confirm the changes in traffic activity from January to December, we plotted the average car counts of all cameras found in Surrey throughout 2020. Next, the positivty rates from the COVID-19 cases found in the Fraser Health Authority was overlayed onto the yearly car plot. Initialy, we wanted to find COVID-19 that was specific to Surrey however, we quickly came to conclusion that the desired data we wanted was very limited and that accessing it would require our sponsors and their internal support to collaborate with health officials. For the sake of simplicity and the time constraint of this project, we decided to use the open data from BC CDC that reflects the Fraser Health Authority region since Surrey is located within that health region (See Figure 13).

![](https://i.imgur.com/TCHknQ9.png)  
*Figure 13: Initial Plot of Mobility in Surrey  and COVID-19 Positivity Rates* 



Upon visualizing the initial plot from Figure 13, we observed that there was a sharp reduction in the car counts following the rapid increase in COVID-19 cases from March to April 2020. This was expected as the province of British Columbia begun to took notice of the novel virus and announced the first lockdown. Interestingly, this was not the case in the months of October to December 2020 where there was only a little decline of car counts compared to the inception of the lockdowns in early March.

#### Granger Causality Test
We decided to use the Granger causality test since allows us to find significant correlations between two independent time series datasets thus, making it useful to draw insights on future predictions on possible lockdowns and potential economic disruptions. For our formula, we had the `Positivity` rate as the independent variable and the `car_count` as the dependent variable. In other words, the formula translated to plain English is simply asking the question:

*"Does the number of car counts seen in traffic rates affect the number of COVID cases in the future?"*

![](https://i.imgur.com/PXNG0MA.png)  
*Figure 14: Granger Causality Test using mobility (car counts) as a predictor variable and positivty rates as the predicted* 

### 4.2.2 Results

The findings that came from the Granger causality test from Figure 14 displayed that there were certain degrees of optimal correlation between COVID-19 positivty rates and car counts. $p$ values of the Granger causality test were also plotted against the lag times to identify correlation. To interpret the graph, the number on the $x$ axis shows number the days from the present day of the initial plot where it is possible to predict a rise in COVID-19 postivity rate. Interestingly, when the formulas were reversed (i.e COVID-19 positivty rates as a predictor of car count), there was no correlation whatsoever. One might interpret that the increased amount of car counts may be an indicator of rising COVID-19 cases however, rising COVID-19 cases may not entirely alter traffic patterns within the city.

It is important to note that although the test conducted showed correlation, there are also real world covariates at play. For example, health authorities may have a delay in reporting COVID-19 cases as there is a lag between the initial time of testing and the reporting period. Next, the data from the Fraser Health region not only encompasses Surrey, but other large municipalities as well such Langley and Burnaby. This may inflate the results of the test as uncessary municipalities were included. The finding of these results, although may be inaccurate, may serve as a baseline to interpret future lockdowns and possible economic disruptions.



# 6. Unbiased Mobility Web App

![](https://i.imgur.com/Lpoc8Wr.png)
Note: The app deployment is current offline. A demo could be found as part of [DSSG 2021 presentation](https://dsi.ubc.ca/data-science-social-good).

## UI Features 
In general the app uses the two maps for its basemap: Positron and  Dark Matter. The Positron map is used in the Camera Map menu for users to view street names and routes while the DarkMatter map is used to assist in heatmap visualization in order to predict traffic patterns.

### Business Overlays

There are 6 business overlays included in this application. All of them are clustered into groups. Those are:    
-  Stores
-  Foods and Restaurants
-  Liquor Stores
-  Health and Medicine
-  Businesses and Finance
-  Services.  

Services are a vague term to incorporate businesses that are not categorized into the other 5 categories above.  This does not include home based businesses.  As the map is zoomed in, the clusters become more dispersed and each individual business icon is shown more in detail. In order to view the name of the business, the cursor must be hovered ontop of the icon.

### Nearby Cameras of Businesses

As one may notice, there is another option in the business overlays' drop-down called `Nearby Cams`. This toggle control will allow the user to locate nearby cameras upon selecting on a business marker and to view the traffic counts captured by them in the explorer panel. This is aimed to give user a brief understanding of the nearby traffic states of a business point of interest.

The range the uses wishes to find nearby cameras can also be customized through the `red gear button` on the top left corner with the radius being adjustable in meters through a slider input. The current maximum number of nearby cameras is set to default **3** (considering the basic usage) and therefore, the highlighted cameras will not exceed 3.

### Heatmap

An option to switch to the heatmap view is provided on the sidebar on the left labelled `Heatmap`. Similar to the basemap in the `Camera Map`, the heatmap has boundaries of all the neighbourhoods of Surrey. However unlike the boundaries found in `Camera Map`, where the user can filter out individual neighbourhoods, the boundaries for `Heatmap`is only fixed to the entire city of Surrey. As of now, the heatmap only displays car count. 


### Bicyle Routes
An overlay of bicycle routes is included in this application along with the overlay panels. Proposed bicycle routes are not included in this overlay. To view the bicycle routes, ensure that the label "Bike Routes " is checked. The bicycle routes are coloured in 7 types. This is shown in the legend listed in the top left corner. The legend is partially hidden from the menu and is undraggable. In order to fully view the legend, the panel (with  the 3 bars) must be collapsed.

### Neighbourhood Filters 
A selection of neighbourhoods is available for selection. The following neighbourhoods are:
-  City Centre for `CITY CENTRE` 
-  Cloverdale for `CLOVERDALE`
-  Fleetwood for `FLEETWOOD`
-  Newton for `NEWTON`
-  South Surrey for `SOUTH SURREY`
-  Whalley for `WHALLEY`

The option panel is on the left hand side and it is titled "Select a Neighbourhood". The default selected is `SURREY`, where it displays all the neighbourhoods within the city of Surrey. When a neighbourhood is selected, the map will shift its view to the selected neighbourhood.



# 7. Conclusion
One important takeaway from this project is the importance of not only quantity but also variety of data. For example, camera metadata such as facing direction, height, and angles are needed to evaluate and improve vehicle detection, to construct flow models, as well as to rectify errors in analysis respectively. 

This project highlights the significance of having the accessibility to different kind of data (i.e numerical and imagery). We have demonstrated how one can benefit from it. We were able to create a 'story' for future users to draw inferences. This allows stakeholders from a non-technical background to utilize the information and apply it in their own area of expertise. In short, combining various data sources is crucial as it can assist in interpreting the statistical findings and applying it to real world scenarios.

Finally, the app created is dedicated to addressing the culmination of findings of the issues in traffic cameras and traffic data throughout our time reseraching this issue. Although still in the early stages, we hope to improve our application so that real world stakeholders can utilize our work in assisting their analysis in the near future. 



# 8. Implications for Social Good
Overall, the findings from this project can be applied to many nonprofit and industrial areas. For example, non profit organizations can use the findings from camera comparisons and the post-detection correction model to provide aid in underrperesented areas where there may be little to no traffic activity. Futhermore, governments can use this information to construct essential services and provide population and economic growth. This may be helpful to areas that contain new immigrants where they need assistance to adjust to Canadian society. Lastly, small businesses can leverage these findings to assist in identifying areas of potential investment. In closing, the applications are versatile enough to cover a domain of topics that can be catered towards one's needs.



# 9. Acknowledgement

Special thanks to these individuals who were able to assist in this project. Their expertise and insights were highly valued during the course of the entire project.

Firstly, we would like to extend our thanks to the entire DSSG team including Dr. Raymond Ng as well as Dr. Kevin Lin, the director and the research administrator of the DSSG respectively. We would also like to recognize Dr. Jean-François Rajotte and Dr. Rob Bergen for their expertise in data science and providing us hands on guidance on the direction of the project. We would also like to extend our thanks to Ms. Hermie Lam for providing us excellent logistics and admistration during our time throughout this project.

We would also like to thank our project sponsor, Mr. Dave Dong from the Cedar Academy Society, for liasing with their internal affairs to provide us data to utilize. Without this partnership, this project would not have been a huge success as it was today.

Lastly, thanks to Mr. Joe Watson from UBC, Mr. Corey Snyder and Dr. Min N. Do from UIUC, as well as Ms. Natasha Mattson for providing expert knowledge from their domains of work and allowing us to leverage their knowledge into this project.
