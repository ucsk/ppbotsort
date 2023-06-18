## PPBoTSORT: Solution for the CTMC Challenge 2023

### 1. Summary

Our solution is the paradigm of **tracking-by-detection**, which allows us to improve detection and tracking separately and facilitate engineering practices.

The architecture can be divided into two parts: the detector and the tracker. We choose <font color="#0000fe">**PP-YOLOE+** (L)</font> as the detector and <font color="#7aabc9">**BoT-SORT** (w/o ReID)</font> as the tracker, and they are practical approaches in the field.

![](https://github.com/ucsk/ppbotsort/assets/53417456/b2d936ac-59df-4443-bc6d-3a004a92b828)

## 2. Data cleaning

The CTMC dataset[^CTMC-v1] is divided into the **first 75% (training set)** and the **second 25% (validation set)**.

To use this data for the detector, we sample different cells at certain intervals to reduce the number of redundant images. Finally, a subset of these image sequences was used to produce an object detection dataset in VOC/COCO format, yielding a training set of **6910 images** and a validation set of **2280 images**.

<img src="https://github.com/ucsk/ppbotsort/assets/53417456/fa64545e-f79a-45ae-a4f9-06e8f6897d2b"  />

When using this data for the tracker, this data is not needed for training and is only used for tracker evaluation since we are not using ReID. At this time, the detector will predict **all images in the latter 25% of CTMC** and will not sample frames as before.

### 3. PP-YOLOE+[^PP-YOLOE]

<details>
<summary><b>PP-YOLOE (Abstract)</b></summary>
In this report, we present PP-YOLOE, an industrial state-of-the-art object detector with high performance and friendly deployment. We optimize on the basis of the previous PP-YOLOv2, using anchor-free paradigm, more powerful backbone and neck equipped with CSPRepResStage, ET-head and dynamic label assignment algorithm TAL. We provide s/m/l/x models for different practice scenarios. As a result, PP-YOLOE-l achieves 51.4 mAP on COCO test-dev and 78.1 FPS on Tesla V100, yielding a remarkable improvement of (+1.9 AP, +13.35% speed up) and (+1.3 AP, +24.96% speed up), compared to the previous state-of-the-art industrial models PP-YOLOv2 and YOLOX respectively. Further, PP-YOLOE inference speed achieves 149.2 FPS with TensorRT and FP16-precision. We also conduct extensive experiments to verify the effectiveness of our designs.
</details>

![](https://ar5iv.labs.arxiv.org/html/2203.16250/assets/model_arch.png)

The "+" in PP-YOLOE+ means industrial practical upgrade, which includes the following three improvements:
1. prediction accuracy: pre-training on **Objects365**. A **learnable weight α** is added to the 1x1 convolution in RepResBlock. Modify the **NMS hyperparameters**.
2. Training speed: After pre-training weights using Objects365, the learning rate is adjusted to 1/10 of the original, and the number of training rounds is reduced from 300 to 80.
3. inference speed: the image normalization operation is removed in the image preprocessing stage, etc.

Visualization of training logs: [VisualDL](https://www.paddlepaddle.org.cn/paddle/visualdl/service/app/scalar?id=3757cf4f6f38b6fe33247dba2acad5b2).

This is the evaluation result of our detector on the validation set.

| Method (pretrained, epochs)    | AP          | AP50        | AP75        |
| ------------------------------ | ----------- | ----------- | ----------- |
| PP-YOLOE (IMAGENET, 300)       | 41.6        | 85.7        | 34.2        |
| PP-YOLOE+ (COCO, 80)           | 41.8 (+0.2) | 85.9 (+0.2) | 33.7 (-0.5) |
| **PP-YOLOE+** (Objects365, 80) | 42.2 (+0.8) | 86.5 (+0.8) | 35.0 (+0.8) |

<img src="https://github.com/ucsk/ppbotsort/assets/53417456/8cb6b0ef-35ec-4acc-9868-28414a02a693"  />

### 4. BoT-SORT[^BoT-SORT]

<details>
<summary><b>BoT-SORT (Abstract)</b></summary>
The goal of multi-object tracking (MOT) is detecting and tracking all the objects in a scene, while keeping a unique identifier for each object. In this paper, we present a new robust state-of-the-art tracker, which can combine the advantages of motion and appearance information, along with camera-motion compensation, and a more accurate Kalman filter state vector. Our new trackers BoT-SORT, and BoT-SORT-ReID rank first in the datasets of MOTChallenge [29, 11] on both MOT17 and MOT20 test sets, in terms of all the main MOT metrics: MOTA, IDF1, and HOTA. For MOT17: 80.5 MOTA, 80.2 IDF1, and 65.0 HOTA are achieved.
</details>

The flow of the method is shown in the figure below. Where the detector is replaced with our PP-YOLOE+. And the ReID module was not added considering the similarity of cell morphology and appearance.

![](https://ar5iv.labs.arxiv.org/html/2206.14651/assets/figures/BoT-SORT-Flow.png)

The following are the evaluation results of the related method on the CTMC validation set. *Where the FPS is measured in the model training phase, and no module such as RepVGG is exported for optimization.*

| Method               | MOTA ↑ | IDF1↑ | ID Sw.↓ | FPS↑  |
| -------------------- | ------ | ----- | ------- | ----- |
| PP-YOLOE & DeepSORT  | 70.5%  | 78.5% | 608     | 6.90  |
| PP-YOLOE & ByteTrack | 70.3%  | 79.1% | 445     | 18.23 |
| PP-YOLOE+ & BoT-SORT | 71.0%  | 81.6% | 324     | 17.39 |

![](https://github.com/ucsk/ppbotsort/assets/53417456/8c9a08bd-a2ec-49c7-8a3f-bd7734a24b52)

### 5. Evaluation on the test set

| Method      | MOTA ↑        | IDF1↑         | ID Sw.↓     | TRA↑          |
| ----------- | ------------- | ------------- | ----------- | ------------- |
| PPByteTrack | 51.0%         | 58.5%         | 2010        | 55.16         |
| PPBoTSORT   | 51.6% (+0.6%) | 62.2% (+3.7%) | 2211 (-201) | 58.28 (+3.12) |

Details about the code of this method are already available in **PaddleDetection**[^PaddleDetection].

---

[^CTMC-v1]: [CTMC: Cell Tracking with Mitosis Detection Dataset Challenge](https://motchallenge.net/data/CTMC-v1)
[^PP-YOLOE]: [PP-YOLOE: An evolved version of YOLO](https://arxiv.org/abs/2203.16250)
[^BoT-SORT]: [BoT-SORT: Robust Associations Multi-Pedestrian Tracking](https://arxiv.org/abs/2206.14651)
[^PaddleDetection]: [PaddleDetection: BoT-SORT](https://github.com/PaddlePaddle/PaddleDetection/tree/release/2.6/configs/mot/botsort)