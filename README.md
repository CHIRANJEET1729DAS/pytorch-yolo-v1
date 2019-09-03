# README

>This is my PyTorch implementation of 
[YOLO v1](https://pjreddie.com/media/files/papers/yolo.pdf) from scratch, which includes scripts for both 
**train/val** and **test**. 

>Results of [sanity check](#Sanity-Check) is shown below.

## Requirements
Python 3.7

CUDA 10.0

PyTorch 1.1

Numpy >= 1.15

Scikit-image >= 0.14

Matplotlib >= 2.2.3

## Descriptions

**Modules**

`utils.py` -- data format transformation and performance evaluation

`draw.py` -- output visualization

`dataset.py` -- dataset and dataloader

`model.py` -- build model 

`model_parallel.py` -- build model in parallel 
(**placing 2 different sub-networks of the model onto 2 GPUs**)

`train.py` -- calculate loss

**Scripts**

`main_model_parallel.py` -- **train** model on VOC  

`main_test_voc.py` -- **test** model on VOC





## Tweak

Since Pytorch does not come with the `same padding` option, minor modification is required:

**Step 1: Go to `conv` module**

Go to PyTorch site package folder.

In my case it is
`/venv/lib/python3.7/site-packages/torch/nn/modules/conv.py`


**Step 2: Add custom function**

Define `conv2d_same_padding` as follows.
    
    def conv2d_same_padding(input, weight, bias=None, stride=1, padding=1, dilation=1, groups=1):

        input_rows = input.size(2)
        filter_rows = weight.size(2)
        effective_filter_size_rows = (filter_rows - 1) * dilation[0] + 1
        out_rows = (input_rows + stride[0] - 1) // stride[0]
        padding_needed = max(0, (out_rows - 1) * stride[0] + effective_filter_size_rows -
                  input_rows)
        padding_rows = max(0, (out_rows - 1) * stride[0] +
                        (filter_rows - 1) * dilation[0] + 1 - input_rows)
        rows_odd = (padding_rows % 2 != 0)
        padding_cols = max(0, (out_rows - 1) * stride[0] +
                        (filter_rows - 1) * dilation[0] + 1 - input_rows)
        cols_odd = (padding_rows % 2 != 0)

        if rows_odd or cols_odd:
            input = F.pad(input, [0, int(cols_odd), 0, int(rows_odd)])

        return F.conv2d(input, weight, bias, stride,
                  padding=(padding_rows // 2, padding_cols // 2),
                  dilation=dilation, groups=groups)

**Step 3: Modify `forward( )`**

Modify `forward` function in `class Conv2d( _ConvNd)` by replacing `F.conv2d` with `conv2d_same_padding`.

    class Conv2d( _ConvNd):

        @weak_script_method
        def forward(self, input):
            #return F.conv2d(input, self.weight, self.bias, self.stride,
            #                        self.padding, self.dilation, self.groups)
            return conv2d_same_padding(input, self.weight, self.bias, self.stride,
                        self.padding, self.dilation, self.groups) ## ZZ: same padding like TensorFlow    

## Dataset
`dataset.py` follows the same data format as that of the original author.

Please follow instructions `Get The Pascal VOC Data` and `Generate Label for VOC` at
 https://pjreddie.com/darknet/yolo/.
 
**Warning**: Make sure you see these in the dataset directory (e.g. folder `VOC_yolo_format`):

    2007_test.txt   VOCdevkit
    2007_train.txt  voc_label.py
    2007_val.txt    VOCtest_06-Nov-2007.tar
    2012_train.txt  VOCtrainval_06-Nov-2007.tar
    2012_val.txt    VOCtrainval_11-May-2012.tar


## Training


*Note:* Require **2 GPUs** each with at least **11 GB** RAM.

**1. Train/Val Data**

Declare your training and validation data via `train_txt` and `val_txt`.
For example:

    train_txt = '/VOC_yolo_format/2007_train.txt'
    val_txt = '/VOC_yolo_format/2007_val.txt'

**2. Set Parameters**

    num_epoch = 1000
    batch_size = 32
    use_float64 = False
    use_scheduler = True
    use_bn = True
    learning_rate = 1e-5
    model_weights = None
    phases = ['train', 'val']

*Note*: For sanity check, set `use_bn = False` and `phase = ['train']` instead.

**3. Run `main_model_parallel.py`**

## Outputs
`Training log`, `plots`, `checkpoints` and **best** `weights` will be automatically saved in these folders.

                ./log
                ./plot
                ./checkpoints
                ./weights

## Evaluation
Similar to
https://github.com/rafaelpadilla/Object-Detection-Metrics. This repo uses YOLO data format.


## Change Log
**Activation function**

As the author mentioned:
``We use a linear activation for the final layer and all other layers use the leaky
rectified linear activation.``


## Experiments
### Sanity Check
Overfit the model with two samples. 

Set regularization to zero by `use_bn = False`, and use **train** mode ONLY via `phase = ['train']`.

To train from scratch, set `model_weights = None`.

**Parameters**
    
    num_epoch = 150
    use_float64 = False
    use_scheduler = False
    use_bn = False
    phase = ['train']
    learning_rate = 1e-5
    model_weights = None   

#### Detections

The following shows the detection output of epoch `25`, `50`, `100` and `150` respectively. 
Model converges after `100` epochs with `loss` close to `0` and `mAP` equals `1.0`.

Notes:

* By VOC convention, `IOU >=0.5` is considered as True Positive.

* Bounding boxes with `confidence_score <= 0.1` will be filtered out.
To customize, go to `utils.py`, and declare your preferred value for `conf_threshold` in `prediction2detection()`.

![image 1](./det_2008_000008_ep=25.png) *epoch = 25*
![image 2](./det_2008_000008_ep=50.png) *epoch = 50*

![image 3](./det_2008_000008_ep=100.png) *epoch = 100*
![image 4](./det_2008_000008_ep=150.png) *epoch = 150*
![image gt](./det_2008_000008_gt.png) *Ground Truth*

#### Loss

![image loss](./loss_history_lr=1e-05_ep=150_wo.png) 

#### mAP

![image mAP](./mAP_history_lr=1e-05_ep=150_wo.png) 



## Reference
[1] You Only Look Once: Unified, Real-Time Object Detection. https://pjreddie.com/media/files/papers/yolo.pdf




