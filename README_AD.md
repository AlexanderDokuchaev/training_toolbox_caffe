# Smart classroom scenario.

This [BVLC Caffe](https://github.com/BVLC/caffe) fork contains code for deployment of action detection and action recognition models for smart classroom use-case. You can define own list of possible actions (see annotation file [format]() and steps for model training to change the list of actions) but this repository shows example for 3 action classes: standing, sitting and raising hand.
## Data preparation
Assume next  structure of data:
<pre>
    |-- data_root
         |-- images
            |-- video_1
                frame_000000.png
                frame_000001.png
            |-- video_2
                frame_000000png
                frame_000001.png
         |-- annotation
            annotation_file_1.xml
            annotation_file_2.xml
         train_tasks.txt
</pre>
Each annotation file (see [this](#annotation-file-format) header) describes a single source of images (see [this](#image-file-format) header).

### Annotation file format
For annotating it's better to use [CVAT](https://github.com/opencv/cvat) utility. So we assume that annotation file is stored in appropriate `.xml` [format](https://github.com/opencv/cvat/blob/develop/cvat/apps/documentation/xml_format.md). In annotation file we have single independent track for each person on video which includes of bounding box description on each frame. General structure of annotation file:
<pre>
    |-- root
         |-- track_0
              bounding_box_0
              bounding_box_1
         |-- track_1
              bounding_box_0
              bounding_box_1
</pre>
 Toy example of annotation file:
```xml
<?xml version="1.0" encoding="utf-8"?>
<annotations count="1">
    <track id="0" label="person">
        <box frame="0" xtl="1.0" ytl="1.0" xbr="0.0" ybr="0.0" occluded="0">
            <attribute name="action">action_name</attribute>
        </box>
    </track>
</annotations>
```
where fields have next description:
 - `count` - number of tracks
 - `id` - unique ID of track in file
 - `label` - label of track (data loader will skips all other labels except `person`)
 - `frame` - unique ID of frame in track
 - `xtl`, `ytl`, `xbr`, `ybr` - bounding box coordinates of top-left and bottom-right corners
 - `occluded` - marker to highlight heavy occluded bounding boxes (can be skipped during training)
 - `name` - name of bounding box attribute (data loader is sensitive for `action` class only)
 - `action_name` - valid name of action (you can define own list of actions)

### Image file format
Our implementation of data loader works with independent images stored on the drive. Each image should be named in format `frame_xxxxxx.png` or `frame_xxxxxx.jpg` (where `xxxxxx` is unique image number).

### Train tasks file format
For more robust control of image sources we have created separate file where each row represents a single source in next format: `annotation_file_path.xml image_height,image_width images_directory_path`. We assume that all images from the same source are resize to `image_height,image_width` sizes (it needs to properly decode annotations).


## Model training
The train procedure for action detection&recognition model consists of two consistent stages:
 1. [Person Detection training (PD)](#person-detection-training)
 2. [Action Recognition training (AR)](#action-recognition-training)
 3. [Action Recognition model evaluation](#action-recognition-model-evaluation)
 4. [Conversion to MO-compatible format](#conversion-to-mo-compatible-format)


**Note 1**: To significantly speedup training you can initialize your model from our distributed `.caffemodel` snapshots:
 1. `$CAFFE_ROOT/models/person_detection_action_recognition/stage1__person_detector/person_detection_0022.caffemodel` - for training Person Detection model
 2. `$CAFFE_ROOT/models/person_detection_action_recognition/stage2__action_regognition/action_detection_0005.caffemodel` - for training Action Detection model

**Note 2**: if you want to change the list of supported actions follow next steps:

 1. Change `ACTION_NAMES_MAP` dictionary in data layer `$CAFFE_ROOT/python/custom_layers/actions_data_layer.py` according your list of actions.
 2. Replace fields `valid_action_ids` and `valid_class_ids` to list of you valid action IDs for next layers in `$CAFFE_ROOT/models/person_detection_action_recognition/stage1__person_detector/train.prototxt` and `$CAFFE_ROOT/models/person_detection_action_recognition/stage2__action_regognition/train.prototxt` files:
    - `data`
    - `detection_matcher`
    - `action/split_loss`
    - `action/glob_push_loss`
    - `action/local_push_loss`
    - `action/push_loss`
    - `action/center_loss`
    - `samples/center_loss`
    - `samples/extractor`
 3. Set a proper number of classes for each row marked with `# num_actions` in next files:
    - `$CAFFE_ROOT/models/person_detection_action_recognition/stage2__action_regognition/train.prototxt`
    - `$CAFFE_ROOT/models/person_detection_action_recognition/stage2__action_regognition/deploy.prototxt`
    - `$CAFFE_ROOT/models/person_detection_action_recognition/stage3__convert_for_inference/ie_conversion.prototxt`
    - `$CAFFE_ROOT/models/person_detection_action_recognition/stage3__convert_for_inference/inference.prototxt`
 4. If you have annotated bounding boxes which must be ignored during training you should set for them new `class_id` which is `num_valid_classes + 1` and add this `class_id` to 'valid_action_ids' in next layers:
    - `data`
    - `detection_matcher`
    - `action/glob_push_loss`


### Person Detection training
On first stage you should train the SSD-based person (two class) detector. To do this you should run single-GPU (python layers does not allow to run on multiple GPUs) training procedure (specify `GPU_ID`):
```
PYTHONPATH=$PYTHONPATH:$CAFFE_ROOT/python $CAFFE_ROOT/build/tools/caffe train --gpu=GPU_ID --solver=$CAFFE_ROOT/models/person_detection_action_recognition/stage1__person_detector/solver.prototxt
 ```
If it's needed the model evaluation can be performed by default pipeline in the original SSD [repository](https://github.com/weiliu89/caffe/tree/ssd). Moreover the training process of PD model can be carried out using SSD-original environment without any changes and after this the weights of trained model can be used as an initialization point on next [stage](#action-recognition-training).

Note: to get more accurate model it's recommended to use pre-training of backbone on default classification or detection datasets.

### Action Recognition training
On next stage we should train the Action Recognition (AR) model which reuses detections from Person Detector (PD) model part and assigns action label for each of them. To do this follow next steps:

 1. To initialize AR model part copy weights from twin PD branch:
```
PYTHONPATH=$PYTHONPATH:$CAFFE_ROOT/python python2 $CAFFE_ROOT/python/rename_layers.py -i pd_weights_path.cafemodel -o ar_init_weights_path.cafemodel -p "cl/"
 ```
 where `pd_weights_path.cafemodel` - weights of trained PD model (see [previous](#person-detection-training) section) and `ar_init_weights_path.cafemodel` - output path to init weights for AR model.

 2. Run single-GPU  training procedure:
```
PYTHONPATH=$PYTHONPATH:$CAFFE_ROOT/python $CAFFE_ROOT/build/tools/caffe train --gpu=GPU_ID --solver=$CAFFE_ROOT/models/person_detection_action_recognition/stage2__action_regognition/solver.prototxt --weights="pd_weights_path.cafemodel,ar_init_weights_path.cafemodel"
 ```

### Action Recognition model evaluation
To evaluate the quality of trained Action Recognition model on your test data you can use provided script. To do this you need the file with testing tasks in the same format as for training stage (see [this](#train-tasks-file-format) section). The model can be evaluated in two modes:

1. Frame independent evaluation:
```
PYTHONPATH=$PYTHONPATH:$CAFFE_ROOT/python python2 $CAFFE_ROOT/python/action_metrics.py -t path_to_testing_tasks.txt -p $CAFFE_ROOT/models/person_detection_action_recognition/stage2__action_regognition/deploy.prototxt -w action_detection_best_snapshot_path.caffemodel
 ```

2. Event-based evaluation:
```
PYTHONPATH=$PYTHONPATH:$CAFFE_ROOT/python python2 $CAFFE_ROOT/python/action_event_metrics.py -t path_to_testing_tasks.txt -p $CAFFE_ROOT/models/person_detection_action_recognition/stage2__action_regognition/deploy.prototxt -w action_detection_best_snapshot_path.caffemodel
 ```

### Conversion to MO-compatible format

 1. Run script to convert weights into MO-compatible format:
```
PYTHONPATH=$PYTHONPATH:$CAFFE_ROOT/python python2 $CAFFE_ROOT/python/convert_to_ie_compatible -m $CAFFE_ROOT/models/person_detection_action_recognition/stage3__convert_for_inference/ie_conversion.prototxt -w "action_detection_snapshot_path.caffemodel" -o "action_detection_inference_path.caffemodel"
 ```
where `action_detection_snapshot_path.caffemodel` is a selected snapshot of AR training procedure and `action_detection_inference_path.caffemodel` - path to save final model parameters.

 2. Run model optimizer:
```
python3 mo_caffe.py -m $CAFFE_ROOT/models/person_detection_action_recognition/stage3__convert_for_inference/inference.prototxt -w action_detection_inference_path.caffemodel -n RMNet_Action_Detection
 ```

