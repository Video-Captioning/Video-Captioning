# Video-Captioning
## Identify and describe the most interesting moments in sports videos
--------
[Overview](Overview.md)
--------

To illustrate the concepts, let's focus on this specific application:

* The sport of Ultimate Frisbee, specifically women's ultimate
* Interesting moments: 
  - During Play: Pulls, Catches, Layouts
  - Between Plays: Scoreboard, High-Fives, Fans in Stands, Miscellaneous

--------

## Inspiration
During MLConf Seattle May 19, 2017, Serena Yeung's presentation, 'Towards Scaling Video Understanding' piqued my curiosity.
Soon after the conference, I read a paper she co-authored, 'End-to-end learning of action detection from frame glimpses in videos'.

The main idea is
  an observation network encodes visual representations of video frames
  a recurrent network sequentially processes these observations and decides both which frame to observe next and when to emit a prediction.

  Beginning at an arbitrarily selected point in the video, repeat the following:
    examine a few frames to find out which actions are occurring at that point in time
    decide where to look next, i.e., ahead or behind, and by how far
  Once the prediction confidence has improved sufficiently, output the prediction
  
--------

## Step-By-Step Exploration

### Step 1: Get some video
First, we will need some data. Let's use the [Women's Final from the 2015 National Championships](https://www.youtube.com/watch?v=ULzQS2rv34s), in which Boston Brute Squad takes on Seattle Riot in Frisco, Texas.

One way to save a video from the web is to use the command-line interface to VLC, for example:
```bash
$ vlc -v -I rc --no-sout-audio --sout='#transcode{ vcodec=h264, vb=800, scale=1 }:std{ access=file,
mux=ts, dst=/path/to/resulting/video.mp4 }' https://www.youtube.com/watch?v=ULzQS2rv34s/ vlc://quit
```
A few techniques will be needed in order to achieve our goal: 
* Video Excerpting
* Feature Extraction
* Image Classification
* Learning Where to Look
* Captioning

### Step 2: Extract still images from the video

Since our video is nearly two hours long, let's grab an excerpt of just a few minutes, and sample a few frames per second from that excerpt. See the Video Excerpting repository, [Practical-VLC](https://github.com/KarlEdwards/Practical-VLC.git)

#### Parameters

* Input: `~/VLC_stuff/2015bsVriot.mp4`
* Start time: `3600` ( seconds from beginning of video )
* Duration: `60` ( seconds )
* Output: `./`  ( current directory )

#### Execution
Putting it all together as follows:

```$ sh excerpt.sh ~/VLC_stuff/2015bsVriot.mp4 ./ 3600 60```

yields a series of a few hundred frames from the video, each frame in a file having a name like frame_0####.png

### Step 3: Extract Features From Images
Next, we are going to use a pre-trained model to extract features from the images we just excerpted from the video.

See the Feature Extraction repository, [ImageFeatures](https://github.com/KarlEdwards/ImageFeatures.git)

1. Activate TensorFlow:
`$ source ~/tensorflow/bin/activate`

2. Extract features, in this case, using MOBILENET:

    `python3 im2fea.py -m MOBILENET --path /to/image/files/ --output features_mobilenet.csv`

   For VGG16, with our data:
    `python3 im2fea.py -m VGG16 --path ./data/2_images/ --output ./data/3_features/features_VGG16.csv`

3. Now repeat for all available models

```
for model in XCEPTION VGG16 VGG19 RESNET50 INCEPTIONV3 INCEPTIONRESNETV2 MOBILENET
do
 echo 'python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m '$model' --path ~/VLC_stuff/frames/2015bsVriot_play --output features_'$model'.csv'
done
```

This script produces another script to repeat the process for all models. Why write a script to produce another script? To allow for a sanity-check before starting a time-consuming process.

```
python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m XCEPTION --path ~/VLC_stuff/frames/2015bsVriot_play --output features_XCEPTION.csv
python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m VGG16 --path ~/VLC_stuff/frames/2015bsVriot_play --output features_VGG16.csv
python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m VGG19 --path ~/VLC_stuff/frames/2015bsVriot_play --output features_VGG19.csv
python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m RESNET50 --path ~/VLC_stuff/frames/2015bsVriot_play --output features_RESNET50.csv
python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m INCEPTIONV3 --path ~/VLC_stuff/frames/2015bsVriot_play --output features_INCEPTIONV3.csv
python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m INCEPTIONRESNETV2 --path ~/VLC_stuff/frames/2015bsVriot_play --output features_INCEPTIONRESNETV2.csv
python3 -W "ignore:compiletime:RuntimeWarning::0" im2fea.py -m MOBILENET --path ~/VLC_stuff/frames/2015bsVriot_play --output features_MOBILENET.csv
```

After convincing myself this is really what I want to do, I paste the generated script into the terminal window and go do something else for a few minutes while the feature extraction occurs.

Which pre-trained model is best (for our purposes?)

If there were no discernable difference in the ability of each model to classify our images, the model having the smallest number of features might be preferrable.

How many feature columns have been produced by each model?

For mobilenet:

`$ awk -F , '(NR<2){print NF}' features_mobilenet.csv`

`1025`

For all models:

```
$ for model in XCEPTION VGG16 VGG19 RESNET50 INCEPTIONV3 INCEPTIONRESNETV2 MOBILENET
    do
      awk -F , '(NR<2){ fn=gensub( "(features_)(.*)(.csv)","\\2", "g", FILENAME ); printf( "%20s\t%4d\n", fn, NF ) }' features_$model.csv
    done
```

```
            XCEPTION	2049
               VGG16	 513
               VGG19	 513
            RESNET50	2049
         INCEPTIONV3	2049
   INCEPTIONRESNETV2	1537
           MOBILENET	1025
```

Let's see what VGG16 can do!

### Step 4: Classify the Images
See the Image Classification and Collage repository

We are going to use the scikit-learn implementation of K-means clustering, and the following files:

* cluster.py
* all_clusters.sh
* make_collage.py
* all_collages.sh


#### Parameters

* Input: `features_VGG16.csv`
* Number of Clusters: `90`    ( 2, 5, 10, 20, ..., 90 )
* Output: `labels_vgg16_90.csv`

#### Execution
First, let's try making 10 clusters:

`$ python3 cluster.py ./data/3_features/features_VGG16.csv -k 10 -o ./data/4_labels/labels_VGG16_10.csv`

Now let's try making various numbers of clusters:

`$ python3 cluster.py features_vgg16.csv -k 2 -o labels_vgg16.csv`

```bash
for n in 05 10 15 20 25 30 40 50 60 70 80 90
do
 echo 'python3 cluster.py ./data/3_features/features_VGG16.csv -k '$n' -o ./data/4_labels/labels_VGG16_'$n'.csv'
done
```

```bash
python3 cluster.py ./data/3_features/features_VGG16.csv -k 05 -o ./data/4_labels/labels_VGG16_05.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 10 -o ./data/4_labels/labels_VGG16_10.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 15 -o ./data/4_labels/labels_VGG16_15.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 20 -o ./data/4_labels/labels_VGG16_20.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 25 -o ./data/4_labels/labels_VGG16_25.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 30 -o ./data/4_labels/labels_VGG16_30.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 40 -o ./data/4_labels/labels_VGG16_40.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 50 -o ./data/4_labels/labels_VGG16_50.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 60 -o ./data/4_labels/labels_VGG16_60.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 70 -o ./data/4_labels/labels_VGG16_70.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 80 -o ./data/4_labels/labels_VGG16_80.csv
python3 cluster.py ./data/3_features/features_VGG16.csv -k 90 -o ./data/4_labels/labels_VGG16_90.csv
```

#### Collages
How well did the clustering work?

Let's make a collage to find out.

`python3 make_collage.py ./data/2_images/ ./data/4_labels/labels_vgg16_05.csv`

![Figure 1.](fig/fig1.png)

Figure 1 shows the first four clusters. Clusters 0 and 2 are good examples of active play. 

To train a classifer that can find images of active play:
1. Make a collage with a large number of clusters

   `python3 make_collage.py ./data/2_images/ ./data/4_labels/labels_vgg16_30.csv`

2. Manually select cluster IDs that are good examples of the desired activity

* choose clusters 0, 2, 5, 8, 11, 13, 15, 19, 23
* make a list of images in these clusters
  - From label file, labels_VGG16_30.csv:
      Label	File
      0	frame_00196.png
      0	frame_00199.png
      ...

```bash
for id in 0 2 5 8 11 13 15 19 23
do
  echo "awk -F '\t' '/^"$id"/{print \$2;}' ./data/4_labels/labels_VGG16_30.csv"
done
```

```
awk -F '\t' '/^0/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^2/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^5/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^8/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^11/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^13/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^15/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^19/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
awk -F '\t' '/^23/{print $2;}' ./data/4_labels/labels_VGG16_30.csv
```

```bash
$ for id in 0 2 5 8 11 13 15 19 23; do   echo "awk -F '\t' '/^"$id"/{print \$2;}' ./data/4_labels/labels_VGG16_30.csv"; done | sh > keep.txt
```

List all the files:
`$ awk -F '\t' '{print $2;}' ./data/4_labels/labels_VGG16_30.csv > all.txt`

* make a list of images NOT in these clusters

Subtract keepers from all:
`grep -v -f keep.txt all.txt > toss.txt`

* build a training data set from these lists

`$ sed 's/.*/toss &/g' toss.txt > training`
`$ sed 's/.*/keep &/g' keep.txt >> training`
   
3. Train a classifier with two classes: desired activity( keep ) / other ( toss )

Dimensionality is too high! 513 features, only 630 rows
Cluster around average keeper, average tosser?


10. Featurize examples

`(tensorflow) ~/Dropbox/Projects/Video-Captioning$ python3 im2fea.py -m VGG16 --path ./examples/catches/ --output ./examples/catches/features_vgg16.csv`

4. Test the classifier
5. Refine and repeat


### Step 5: Learning Where to Look

### Step 6: Captioning

--------

## Hardware & Configuration

* Model Name:	Mac mini
* Model Identifier:	Macmini7,1
* Processor Name:	Intel Core i7
* Processor Speed:	3 GHz
* Number of Processors:	1
* Total Number of Cores:	2
* L2 Cache (per Core):	256 KB
* L3 Cache:	4 MB
* Memory:	8 GB
* Boot ROM Version:	MM71.0220.B07
* SMC Version (system):	2.24f32

### Bibliography
* Yeung, S., Russakovsky, O., Jin, N., Andriluka, M., Mori, G., Fei-Fei, L.: Every moment counts: dense detailed labeling of actions in complex videos (2015). arXiv preprint arXiv:1507.05738

* Yeung, S., Russakovsky, O., Mori, G., Fei-Fei, L.: End-to-end learning of action detection from frame glimpses in videos. In: CVPR (2016)
@article{yeung2015end,
  title={End-to-end Learning of Action Detection from Frame Glimpses in Videos},
  author={Yeung, Serena and Russakovsky, Olga and Mori, Greg and Fei-Fei, Li},
  journal={arXiv preprint arXiv:1511.06984},
  year={2015}
}

* Scikit-learn: Machine Learning in Python, Pedregosa et al., JMLR 12, pp. 2825-2830, 2011
@article{scikit-learn,
 title={Scikit-learn: Machine Learning in {P}ython},
 author={Pedregosa, F. and Varoquaux, G. and Gramfort, A. and Michel, V.
         and Thirion, B. and Grisel, O. and Blondel, M. and Prettenhofer, P.
         and Weiss, R. and Dubourg, V. and Vanderplas, J. and Passos, A. and
         Cournapeau, D. and Brucher, M. and Perrot, M. and Duchesnay, E.},
 journal={Journal of Machine Learning Research},
 volume={12},
 pages={2825--2830},
 year={2011}
}

* Keras
@misc{chollet2015keras,
  title={Keras},
  author={Chollet, Fran\c{c}ois and others},
  year={2015},
  publisher={GitHub},
  howpublished={\url{https://github.com/keras-team/keras}},
}

### Toolbox
* [VLC](https://www.videolan.org/vlc/index.html)-2.2.8
* [Tensorflow](https://www.tensorflow.org/install/install_mac)
* [Python3](https://wsvincent.com/install-python3-mac/)
* [scikit-learn](http://scikit-learn.org/stable/)
* [keras](https://keras.io/applications/)
