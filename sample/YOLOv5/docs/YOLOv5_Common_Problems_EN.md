[简体中文](./YOLOv5_Common_Problems.md) | [English](./YOLOv5_Common_Problems_EN.md)

# YOLOv5 Common Problems
## 1. Anchors with Inconsistent Sizes
If the predicted detection box has a center position that closely matches the actual target center position, **but its size is significantly different**, such as being much larger than the target, it indicates that the preset anchor sizes are inconsistent with the target sizes.

Typically, the YOLO algorithm checks the effectiveness of the preset anchor sizes before training. If they do not meet the requirements, the algorithm will re-cluster the sizes of the anchors based on the sizes of the annotated detection boxes in the training dataset to generate suitable anchor sizes. Therefore, when transferring the algorithm for inference, **it is necessary to manually adjust the anchor sizes to match those generated by clustering before training.**

The general operation method is to load the model with the original code, set a breakpoint, and check the anchor parameters in the model. Below is the Python code provided to view the anchor parameters in the YOLOv5 weights in the open-source repository.

```python
import torch
from models.experimental import attempt_load
modelPath = 'Your_Model_Path'
model = attempt_load(modelPath, map_location=torch.device('cpu'))
m = model.module.model[-1] if hasattr(model, 'module') else model.model[-1]
print(f"little targets' anchors : {m.anchors[0]*m.stride[0]}")
print(f"middle targets' anchors : {m.anchors[1]*m.stride[1]}")
print(f"large  targets' anchors : {m.anchors[2]*m.stride[2]}")
```

## 2. TPU-NNTC Quantization Precision Loss
When using the Fp32 bmodel for model inference, the predicted results match the expected values. However, when using an automatic quantization tool to quantize the model, significant changes occur in the confidence and position of the detection boxes. Generally, the detection boxes tend to **shift towards the upper left corner.**

This issue is caused by unreasonable quantization settings. For YOLOv5 models, it is generally recommended **not to quantize the last three convolutional layers and subsequent processing**. To determine the output type of the bmodel, use `bmrt_test –bmodel YOUR_MODUL_PATH` command. If it shows as INT8, you need to use Netron software to examine the prototxt file in FP32 format and identify the names of the last three convolutional layers. Then, freeze these layers to prevent quantization.

> **Methods To View Weight Names**
After using the one-click quantization script, a `yolov5s_bmnetp_test_fp32.prototxt` file will be generated in the directory where the original model is located. You need to download this file to your local machine and open it using Netron. Netron is a neural network visualization software that can be accessed through the [website](https://netron.app/) to view the neural network computation graph. Locate the last three convolutional layers before the network output layer, click on the corresponding parameter buttons, and the corresponding layer names will be displayed on the right side of Netron.

When significant precision loss still occurs after performing the above operations, you can try the following strategies:

1. For users of  `ufw.cali.cali_model`, we provide an automatic search quantization strategy: `--postprocess_and_calc_score_class detect_accuracy`. The optional parameter commands include `detect_accuracy`, `topx_accuracy_for_classify`, `feature_similarity`, and `None`. Please note that this option may take longer to execute.
2. If you are familiar with model architecture design, you can manually design a quantization strategy using `--try_cali_accuracy_opt`. The available quantization strategies are derived from the `calibration_use_pb` module, and you can refer to `calibration_use_pb --help` for more information.
3. When the precision or flexibility of the one-click quantization script does not meet your requirements, you can use a step-by-step quantization approach, customizing the strategy used at each step of the quantization process. Please refer to the NNTC reference manual for specific methods.

For more methods on precision alignment, you can refer to the [Precision Alignment Guide](../../../docs/FP32BModel_Precise_Alignment.md).

## 3. Other Questions
1. Preprocessing is not aligned. For example, YOLOv5 uses gray edge filling (letterbox) to resize images and cannot be directly resized.
2. If you are using a single-output YOLOv5 model, the decoding part corresponding to the calculation layer should not be quantized.
3. When compiling the bmodel, make sure to specify the `--target` parameter consistent with the device you are deploying on (BM1684/BM1684X).
4. If you are unable to load the model properly, try using `bm-smi` to check the TPU status and utilization. If any exceptions occur (TPU status is Fault or utilization is 100%), please contact technical support or create an issue on GitHub.