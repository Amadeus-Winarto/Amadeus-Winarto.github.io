---
layout: posts
title: Variations of SSD — Deeply Supervised Object Detector (DSOD)
---
# Variations of SSD — Deeply Supervised Object Detector (DSOD)
The use of single-shot object detectors has been hailed as a faster and more efficient alternative to two-stage detectors such as Faster R-CNNs. The total removal of object proposals in single-stage detectors allows it to achieve high speeds at the cost of accuracy. Deeply Supervised Object Detector (DSOD), a variation of the original Single Shot MultiBox Detector (SSD), attempts to mitigate this problem.
## Principles and Intuition
The authors of the DSOD paper made the following principles to design an object detector:
### Proposal-free
There are three types of object detectors in general. Firstly, R-CNN and Fast R-CNN that use external proposal generator, for example, the Selective Search algorithm, to propose objects. Secondly, Faster R-CNN and R-FPN that use integrated proposal generator like the Regional Proposal Network (RPN) to generate proposals. Lastly, SSD and YOLO that are proposal-free by reframing localisation into a regression problem, that is, predicting the offsets of the anchor boxes from the objects’ actual bounding box
![Localisation](../imgs/DSOD/Localisation.png "Localisation in Proposal-Free Detectors")
<div align="center" markdown="1">
*Figure 1: Localisation in Proposal-Free detectors. The detector only need to classify which anchor boxes (grey striped lines) contain the desired object. Then, the localisation part of the detector will determine how best to resize the anchor boxes to properly fit the object i.e. predicting offsets.*
</div>

The authors observed that to train an object detector from scratch, models that are proposal-free should be used. This is because of the use of backpropagation to update network parameters. In models that use object proposals, a type of Region of Interest Pooling layer is typically used. However, this layer prevents good gradient flow that is necessary to train an entire object detector from scratch. Hence, DSOD uses SSD as a framework.

![RegionProposal](../imgs/DSOD/RegionProposal.png "RPN")
<div align="center" markdown="1">
*Figure 2: Faster R-CNN with RPN(left) vs SSD without RPN (right).*
</div>

### Deep Supervision
Deep Supervision (DS) was a concept developed to tackle the problem of vanishing gradient. In deep networks, the flow of gradient can be stunted if it is too small. This is because of the way gradient descent is implemented; weights are updated based on the partial derivatives of the loss function with respect to the current weights for each layer.
This is computed using the partial derivatives for previous layers closer to the loss function through the chain rule, which involves multiplication.
We can see how this may turn into a problem: multiply 0.9 with 0.8 and you get 0.72, a number smaller than the initial values. As networks grow deeper, more and more possibly small numbers are multiplied together. Thus, the final gradient becomes so small that the earlier parts of the model cannot learn anything.
The idea of DS is to bring the loss function, typically attached to the top part of the network, closer to different layers of the said network. This allows each layer to use a less “diluted” gradient to learn.
While DS was originally executed explicitly by attaching auxiliary losses, such as in Inception v4, there are implementations that apply DS implicitly, such as the proposed architecture DenseNet. DSOD thus use implicit DS like DenseNet to mitigate vanishing gradients.
### Transition Without Pooling
The transition without pooling layers are layers between dense blocks. The layer does the batch normalization operation, followed by a 1x1 convolution.
As its name suggests, the layer lacks the 2x2 average pooling layer compared to the transition layer used in DenseNet.
In DenseNet, the number of dense blocks is fixed to 4 to maintain the same scale of outputs. The only way to increase network depth is adding layers inside each block for the original DenseNet. The transition w/o pooling layer circumvents this restriction, allowing for more dense blocks to be used.
![DenseNet](../imgs/DSOD/Dense.png "DenseNet")
<div align="center" markdown="1">
*Figure 3: A 5-layered Dense Block with Growth Rate = 4. Transition w/o Pooling layer is applied to the end of the Dense Block in DSOD*
</div>

### Stem block
A stem block is used to modify the original DenseNet architecture. Instead of a using a 7x7 convolution layer with stride 2 followed by a 3x3 MaxPooling operation with stride 2, DSOD uses a stack of 3x3 convolution layers followed by a 2x2 MaxPooling. The first convolution layer has stride 2 whereas the others use stride 1. This is to minimise information loss from the raw input image, as smaller filter sizes and strides tend to preserve information.
### Dense Prediction Structure
To improve detection accuracy, DSOD concatenates feature maps processed from previous layers with down-sampled, high-resolution feature maps in a one-to-one ratio for detection. This is because the high-resolution feature maps preserves spatial information, whereas processed feature maps from previous layers have information useful for classification.
In comparison, SSD uses only feature maps processed from previous layers to make multi-scale detection.

![DSOD](../imgs/DSOD/DSOD.png "DSOD")
<div align="center" markdown="1">
*Figure 4: Plain Connection in SSD vs Dense Connection in DSOD. In each scale, SSD learns all of the feature maps using convolution layers from previous scales. DSOD on the other hand down-samples feature maps from previous scale and concatenates them with the learned feature maps.*
</div>

The down-sampling operation is done using a 2x2 max pooling layer with stride 2 followed by a 1x1 convolution layer with stride 1. This operation provides each scale with down-sampled feature maps from all of the previous scales, which is essentially the same as the dense layer-wise connection introduced in DenseNet, hence its name.
## Overall Architecture
Due to the use of DenseNet-like architecture, DSOD has a few additional hyperparameters to tune. These include:
Channels in First Convolution Layer
Empirically, the higher the number of channels in the first convolutional layer, the better the performance. This is (maybe) due to the fact that there are less information loss from the images.
### Channels in Bottleneck-layer
The bottleneck layer is defined as the layer that does the batch normalization operation followed by application of an activation function and then ultimately a 1x1 convolution . For DSOD, the higher the number of channels in the bottleneck-layer, the better.
### Growth Rate
Assuming each layer produces K feature maps, then the nth layer will have K₀ + K ×(n-1) feature maps as input due to the dense connections within a dense block, where K₀ is the number of channels initially (for RGB images, K₀ = 3). K is thus called the growth rate. For DSOD, the higher the growth rate, the higher the accuracy. In fact, a 4.8% increase in mean average precision (mAP) was observed when K = 48 is used instead of K = 16 on the Pascal VOC Dataset.
### Compression Factor in Transition Layers
Compression factors keep the number of channels of feature maps from going too big. Because of the presence of dense connections, the more layers used, the higher the number of feature maps produced.
nth layer’s input = K₀ + K ×(n-1)
Hence, between dense blocks, the transition layers reduce the number of feature maps. If a dense block outputs M feature map, then the transition layers will take all m feature maps as input and produces floor(W ×M) feature maps. For example, if W = 0.5 and M = 45, then the transition layer will output floor(0.5 ×45) = 22 feature maps.
DSOD uses W = 1, however, which means no compression is done i.e. number of feature maps will be the same after the transition layer. W = 1 leads to 2.9% higher mAP than W = 0.5.
## Training Details
DSOD’s training strategy is largely similar to that of SSD in terms of data augmentation. The main difference is in regularization. L2 normalization is used to scale the feature norm to 20 on all outputs. This is contrast to the use of L2 normalization in SSD which only applies it to the first scale.
Results
To validate the architectural changes, several tests were conducted on the following datasets
PASCAL VOC 2007: mAP of 77.7%
PASCAL VOC 2012: mAP of 76.3%
MS COCO + PASCAL VOC 2007: mAP of 81.7%
To put into perspective, these scores are higher by less than 1% compared to SSD models, and is slightly worse, by less than a percentile point, compared to Region-based Fully Convolutional Network (R-FCN).
Conclusion and Afterthoughts
DSOD is a single-stage detector. Thus, it is smaller and faster compared to two-stage detectors like Faster R-CNN. It does this by employing strategies that increases efficiency of the network. However, there are some corners that cannot be cut. For example, the compression factor and growth rate had to be large for the detector to perform.
This is a trade-off between efficiency and information. A model performance is only as good as the information it gets. Thus, it appears that the old adage, “garbage in, garbage out”, is once again triumphant.
