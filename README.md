# Examples for NeuroProof

This directory contains examples for how to run the neuroproof
(https://github.com/janelia-flyem/NeuroProof) utilities
along with some sample data from FlyEM (http://janelia.org/team-project/fly-em)
and their efforts to reconstruct neurons from the Drosophila medulla prepared
using FIB-SEM imaging.  The h5 files and xml need to be unzipped before using.  

## 1.  Boundary Training

The 'training_sample1' directory contains a volume that was used to train a boundary
classifier to used in subsequent parts of the segmentation flow.  To
generate a classifier, one should use the Ilastik tool from Heidelberg
(http://www.ilastik.org).  One should open up the grayscale stack,
create multiple class labels (where class label 0 should be designated as
classifying cellular boundaries).  These labels can be efficiently
trained and saved to an ILP file format.  We provide an example from
our own pipeline for comparisons 'training_sample1/results/boundary_classifier_ilastik.ilp'.

## 2.  Agglomeration Training

The 'training_sample2' directory contains a grayscale image volume and its labeled groundtruth
and is used to train an agglomeration classifier.  This agglomeration
classifier is produced by calling 'neuroproof_graph_train'.  The inputs required
is an oversegmented (watershed) volume 'oversegmented_stack_labels.h5'
and its respective groundtruth 'groundtruth.h5'.  Also,
a prediction file ('prediction.h5') that contains multiple channels of class predictions for
each label needs to be provided.  (For now channel 0 is always treated as the
boundary channel and channel 2 is the mitochondria channel IF mitochondria options
are enabled in the pipeline, otherwise the predictions can be generic.)

The oversegmented volume can be achieved by running the Ilasitik boundary
predictor over the image stack and generating a watershed.  For now, NeuroProof
does not support this operation; however, the open-source tool Gala
(https://github.com/janelia-flyem/gala) can be used to take the boundary
classification file produced by Ilastik and an image volume to produce an
initial oversegmented stack.  An example of how Gala could be called:

gala-segmentation-pipeline -I '|graymaps|/*.png' --ilp-file |boundary classifier| --enable-gen-supervoxels
                --enable-gen-agglomeration --enable-gen-pixel --seed-size 5 |output directory| --segmentation-thresholds 0.0 

The output directory will contain a label volume, as well as, a prediction file and
other data not essential for agglomeration training.

Once an over-segmentation and ground-truth labeling exists, 'neuroproof_graph_train'
can be called to produce a prediction using a strategy similar to
[Nunez-Iglesias et al '13] (http://arxiv.org/abs/1303.6163).  The following
uses generates a classifier to be used if mitochondria labels are provided
in channel 3 and one desires to classify edges using two passes:

neuroproof_graph_learn |oversegmented_labels| |prediction| |groundtruth| --strategy_type 1

If classification that does not consider mitochondria information is desired
the following can be run:

neuroproof_graph_learn |oversegmented_labels| |prediction| |groundtruth| --strategy_type 2 --num_iterations 5

The output of these procedures is an agglomeration classifier.  We provide example
classifiers produced from the neuroproof using these two commands -- 'mito_aware.xml'
and  'nomito_aware.h5' respectively.


## 3.  Agglomeration Procedure

Once an agglomeration classifier is produced, one can efficiently apply this
classifier to a new dataset and generate a segmentation.  This is done by
calling 'neuroproof_graph_predict'. To run this algorithm, one needs an agglomeration
classifier as produced in #2, an over-segmented volume to be segmented, and a prediction
file.  As before, this over-segmented volume and prediction file will be produced
using Gala which needs a boundary classifier (#1) and an initial image volume.

neuroproof_graph_predict |oversegmented_labels| |prediction| |classifier| 

This will produce two files: a segmented label volume and graph.json which describes
the certainty of an edge be a true edge in the graph.  This graph file can be
used to focus manual correcting efforts in the segmentation.  These outputs are
provided in the results directory.

The prediction algorithm gives the option to specify a separate classifier for
annotating certainty values on the final exported graph (--postseg-classifier-file).
For example, one can use the mito-aware classifier to separate out mitochondria
properly when doing multi-stage classification, but then use the non-mito
classifier to generate the graph file so that edge certainty of mitochondria
bodies are treated in similar way to non-mitochondria bodies.

## 4.  Segmentation and Graph Analysis

The uncertainty in the resulting graph 'validation_sample/results/graph.json'
produced in #3 can be analyzed using the following NeuroProof command:

neuroproof_graph_analyze -g 1 -b 1

This provides an estimate of how many edits will be needed to fix the graph
and the overall uncertainty of the segmentation as defined in [Plaza '12] 
(http://www.plosone.org/article/info%3Adoi%2F10.1371%2Fjournal.pone.0044448).

To compare the segmentation to ground truth using the similarity
metric Variance of Information (VI), one can issue the following command:

neuroproof_graph_analyze_gt |segmentation| |groundtruth|

We provide a ground-truth labeling in 'validation_sample/groundtruth.h5'
and a test segmentation in 'validation_sample/results/segmentation.h5'
to allow one to test these command.


