# *FaRL* for *Fa*cial *R*epresentation *L*earning
	
[![PWC](https://img.shields.io/endpoint.svg?url=https://paperswithcode.com/badge/general-facial-representation-learning-in-a/face-alignment-on-300w)](https://paperswithcode.com/sota/face-alignment-on-300w?p=general-facial-representation-learning-in-a)
[![PWC](https://img.shields.io/endpoint.svg?url=https://paperswithcode.com/badge/general-facial-representation-learning-in-a/face-alignment-on-aflw-19)](https://paperswithcode.com/sota/face-alignment-on-aflw-19?p=general-facial-representation-learning-in-a)
[![PWC](https://img.shields.io/endpoint.svg?url=https://paperswithcode.com/badge/general-facial-representation-learning-in-a/face-alignment-on-wflw)](https://paperswithcode.com/sota/face-alignment-on-wflw?p=general-facial-representation-learning-in-a)
[![PWC](https://img.shields.io/endpoint.svg?url=https://paperswithcode.com/badge/general-facial-representation-learning-in-a/face-parsing-on-celebamask-hq)](https://paperswithcode.com/sota/face-parsing-on-celebamask-hq?p=general-facial-representation-learning-in-a)
[![PWC](https://img.shields.io/endpoint.svg?url=https://paperswithcode.com/badge/general-facial-representation-learning-in-a/face-parsing-on-lapa)](https://paperswithcode.com/sota/face-parsing-on-lapa?p=general-facial-representation-learning-in-a)

This repo hosts official implementation of our paper "[**General Facial Representation Learning in a Visual-Linguistic Manner**](https://arxiv.org/abs/2112.03109)".


## Introduction

**FaRL** offers powerful pre-training transformer backbones for face analysis tasks. Its pre-training combines both the image-text contrastive learning and the masked image modeling.

<img src="./figures/framework.jpg" alt="framework" width="400"/>

After the pre-training, the image encoder can be utilized for various downstream face tasks. 

## Pre-trained Backbones

We offer different pre-trained transformer backbones as below.

| Model Name  |  Data | Epoch | Link |
| ----------- | -------------- | ----- | ---- |
| FaRL-Base-Patch16-LAIONFace20M-ep16 (used in paper) | LAION Face 20M  | 16 | [BLOB](https://facevcstandard.blob.core.windows.net/haya/releases/farl/FaRL-Base-Patch16-LAIONFace20M-ep16.pth?sv=2020-08-04&st=2021-12-17T13%3A00%3A07Z&se=2025-01-18T13%3A00%3A00Z&sr=b&sp=r&sig=D0ZPJgp8BrAgHIdACfZzqPnyOcX1ivGdHnF8qgtWdoI%3D) |
| FaRL-Base-Patch16-LAIONFace20M-ep64 | LAION Face 20M | 64 | [BLOB](https://facevcstandard.blob.core.windows.net/haya/releases/farl/FaRL-Base-Patch16-LAIONFace20M-ep64.pth?sv=2020-08-04&st=2021-12-27T05%3A22%3A56Z&se=2025-12-21T05%3A22%3A00Z&sr=b&sp=r&sig=til1J9u%2FQqf6qRc6cPx9nPyOGl%2F9ahTyvQ3VBPePs6A%3D) |
<!-- | FaRL-Base-Patch16-LAIONFace50M-ep16 | LAION Face 50M | [BLOB](https://facevcstandard.blob.core.windows.net/haya/releases/farl/FaRL-Base-Patch16-LAIONFace50M-ep16.pth?sv=2020-08-04&st=2021-12-17T13%3A01%3A48Z&se=2025-01-17T13%3A01%3A00Z&sr=b&sp=r&sig=6g1B3f4vEmFc1tmz8QWSH6lRoK%2BABA%2FWfmqXLGS61MM%3D) | -->


## Setup Downstream Training

We run all downstream trainings on 8 NVIDIA GPUs (32G). Our code supports other GPU configurations, but we do not guarantee the resulting performances on them.
Before setting up, install these packages:
* [PyTorch](https://pytorch.org/get-started/previous-versions/) 1.7.0
* [MMCV-Full](https://github.com/open-mmlab/mmcv) 1.3.14

Then, install the rest dependencies with `pip install -r ./requirement.txt`.

Please refer to [./DS_DATA.md](./DS_DATA.md) to prepare the training and testing data for downstream tasks.

Download [the pre-trained backbones](https://github.com/microsoft/FaRL#pre-trained-backbones) into `./blob/checkpoint/`.
Now you can launch the downstream trainings & evaluations with following command template.

```
python -m blueprint.run \
  farl/experiments/{task}/{train_config_file}.yaml \
  --exp_name farl --blob_root ./blob
```

The repo has included some config files under `./farl/experiments/` that perform finetuning for face parsing and face alignment.
For example, if you would like to launch a face parsing training on LaPa by finetuning our `FaRL-Base-Patch16-LAIONFace20M-ep16` pre-training, simply run with:

```
python -m blueprint.run \
  farl/experiments/face_parsing/train_lapa_farl-b-ep16_448_refinebb.yaml \
  --exp_name farl --blob_root ./blob
```

Or if you would like to launch a face alignment training on 300W by finetuning our `FaRL-Base-Patch16-LAIONFace20M-ep16` pre-training, you can simply run with:

```
python -m blueprint.run \
  farl/experiments/face_alignment/train_ibug300w_farl-b-ep16_448_refinebb.yaml \
  --exp_name farl --blob_root ./blob
```

It is also easy to create new config files for training and evaluation on your own. For example, you can customize your own face parsing task on CelebAMask-HQ by editing the values below (remember to remove the comments before running).

```yaml
package: farl.experiments.face_parsing

class: blueprint.ml.DistributedGPURun
local_run:
  $PARSE('./trainers/celebm_farl.yaml', 
    cfg_file=FILE,
    train_data_ratio=None, # The data ratio used for training. None means using 100% training data; 0.1 means using only 10% training data.
    batch_size=5, # The local batch size on each GPU.
    model_type='base', # The size of the pre-trained backbone. Supports 'base', 'large' or 'huge'.
    model_path=BLOB('checkpoint/FaRL-Base-Patch16-LAIONFace20M-ep16.pth'), # The path to the pre-trained backbone.
    input_resolution=448, # The input image resolution, e.g 224, 448. 
    head_channel=768, # The channels of the head.
    optimizer_name='refine_backbone', # The optimization method. Should be 'refine_backbone' or 'freeze_backbone'.
    enable_amp=False) # Whether to enable float16 in downstream training.
```

## Performance

The following table illustrates the performances of our `FaRL-Base-Patch16-LAIONFace20M-ep16` pre-training, which is pre-trained with 16 epoches, both reported in the paper (Paper) and reproduced using this repo (Rep). There are small differences between their performances due to code refactorization.

| Name | Task | Benchmark | Metric | Score (Paper/Rep) | Logs (Paper/Rep) |
| ---- | ---- | ---- | --- | --- | --- |
| [face_parsing/<br/>train_celebm_farl-b-ep16-448_refinebb.yaml](./farl/experiments/face_parsing/train_celebm_farl-b-ep16_448_refinebb.yaml) | Face Parsing  | CelebAMask-HQ | F1-mean ??? | 89.56/89.65 | [Paper](./logs/paper/face_parsing.train_celebm_farl-b-ep16-448_refinebb), [Rep](./logs/reproduce/face_parsing.train_celebm_farl-b-ep16_448_refinebb) |
| [face_parsing/<br/>train_lapa_farl-b-ep16_448_refinebb.yaml](./farl/experiments/face_parsing/train_lapa_farl-b-ep16_448_refinebb.yaml) | Face Parsing | LaPa | F1-mean ??? | 93.88/93.86 | [Paper](./logs/paper/face_parsing.train_lapa_farl-b-ep16_448_refinebb), [Rep](./logs/reproduce/face_parsing.train_lapa_farl-b-ep16_448_refinebb) |
| [face_alignment/<br/>train_aflw19_farl-b-ep16_448_refinebb.yaml](./farl/experiments/face_alignment/train_aflw19_farl-b-ep16_448_refinebb.yaml) | Face Alignment | AFLW-19 (Full) | NME_diag ??? | 0.943/0.943 | [Paper](./logs/paper/face_alignment.train_aflw19_farl-b-ep16_448_refinebb), [Rep](./logs/reproduce/face_alignment.train_aflw19_farl-b-ep16_448_refinebb) |
| [face_alignment/<br/>train_ibug300w_farl-b-ep16_448_refinebb.yaml](./farl/experiments/face_alignment/train_ibug300w_farl-b-ep16_448_refinebb.yaml) | Face Alignment | 300W (Full) | NME_inter-ocular ??? | 2.93/2.92 | [Paper](./logs/paper/face_alignment.train_ibug300w_farl-b-ep16_448_refinebb), [Rep](./logs/reproduce/face_alignment.train_ibug300w_farl-b-ep16_448_refinebb) |
| [face_alignment/<br/>train_wflw_farl-b-ep16_448_refinebb.yaml](./farl/experiments/face_alignment/train_wflw_farl-b-ep16_448_refinebb.yaml) | Face Alignment | WFLW (Full) | NME_inter-ocular ??? | 3.96/3.98 | [Paper](./logs/paper/face_alignment.train_wflw_farl-b-ep16_448_refinebb), [Rep](./logs/reproduce/face_alignment.train_wflw_farl-b-ep16_448_refinebb) |


Below we also report results of our new `FaRL-Base-Patch16-LAIONFace20M-ep64`, which is pre-trained with 64 epoches instead of 16 epoches as above, showing further improvements on most tasks.

| Name | Task | Benchmark | Metric | Score | Logs |
| ---- | ---- | ---- | --- | --- | --- |
| [face_parsing/<br/>train_celebm_farl-b-ep64-448_refinebb.yaml](./farl/experiments/face_parsing/train_celebm_farl-b-ep64_448_refinebb.yaml) | Face Parsing  | CelebAMask-HQ | F1-mean ??? | 89.57 | [Rep](./logs/reproduce/face_parsing.train_celebm_farl-b-ep64_448_refinebb) |
| [face_parsing/<br/>train_lapa_farl-b-ep64_448_refinebb.yaml](./farl/experiments/face_parsing/train_lapa_farl-b-ep64_448_refinebb.yaml) | Face Parsing | LaPa | F1-mean ??? | 94.04 | [Rep](./logs/reproduce/face_parsing.train_lapa_farl-b-ep64_448_refinebb) |
| [face_alignment/<br/>train_aflw19_farl-b-ep64_448_refinebb.yaml](./farl/experiments/face_alignment/train_aflw19_farl-b-ep64_448_refinebb.yaml) | Face Alignment | AFLW-19 (Full) | NME_diag ??? | 0.938 | [Rep](./logs/reproduce/face_alignment.train_aflw19_farl-b-ep64_448_refinebb) |
| [face_alignment/<br/>train_ibug300w_farl-b-ep64_448_refinebb.yaml](./farl/experiments/face_alignment/train_ibug300w_farl-b-ep64_448_refinebb.yaml) | Face Alignment | 300W (Full) | NME_inter-ocular ??? | 2.88 | [Rep](./logs/reproduce/face_alignment.train_ibug300w_farl-b-ep64_448_refinebb) |
| [face_alignment/<br/>train_wflw_farl-b-ep64_448_refinebb.yaml](./farl/experiments/face_alignment/train_wflw_farl-b-ep64_448_refinebb.yaml) | Face Alignment | WFLW (Full) | NME_inter-ocular ??? | 3.88 | [Rep](./logs/reproduce/face_alignment.train_wflw_farl-b-ep64_448_refinebb) |


<!-- We also report results using the 50M pre-trained backbone, showing further enhancement on LaPa and AFLW-19.

| Config | Task | Benchmark | Metric | Score | Logs |
| ---- | ---- | ---- | --- | --- | --- |
| [face_parsing/<br/>train_celebm_farl-b-50m-ep16-448_refinebb.yaml](./farl/experiments/face_parsing/train_celebm_farl-b-50m-ep16_448_refinebb.yaml) | Face Parsing  | CelebAMask-HQ | F1-mean ??? | 89.68 | [Rep](./logs/reproduce/face_parsing.train_celebm_farl-b-50m-ep16_448_refinebb) |
| [face_parsing/<br/>train_lapa_farl-b-50m-ep16_448_refinebb.yaml](./farl/experiments/face_parsing/train_lapa_farl-b-50m-ep16_448_refinebb.yaml) | Face Parsing | LaPa | F1-mean ??? | 94.01 | [Rep](./logs/reproduce/face_parsing.train_lapa_farl-b-50m-ep16_448_refinebb) |
| [face_alignment/<br/>train_aflw19_farl-b-50m-ep16_448_refinebb.yaml](./farl/experiments/face_alignment/train_aflw19_farl-b-50m-ep16_448_refinebb.yaml) | Face Alignment | AFLW-19 (Full) | NME_diag ??? | 0.937 | [Rep](./logs/reproduce/face_alignment.train_aflw19_farl-b-50m-ep16_448_refinebb) |
| [face_alignment/<br/>train_ibug300w_farl-b-50m-ep16_448_refinebb.yaml](./farl/experiments/face_alignment/train_ibug300w_farl-b-50m-ep16_448_refinebb.yaml) | Face Alignment | 300W (Full) | NME_inter-ocular ??? | 2.92 | [Rep](./logs/reproduce/face_alignment.train_ibug300w_farl-b-50m-ep16_448_refinebb) |
| [face_alignment/<br/>train_wflw_farl-b-50m-ep16_448_refinebb.yaml](./farl/experiments/face_alignment/train_wflw_farl-b-50m-ep16_448_refinebb.yaml) | Face Alignment | WFLW (Full) | NME_inter-ocular ??? | 3.99 | [Rep](./logs/reproduce/face_alignment.train_wflw_farl-b-50m-ep16_448_refinebb) | -->


## Contact

For help or issues concerning the code and the released models, feel free to submit a GitHub issue, or contact [Hao Yang](https://haya.pro) ([haya@microsoft.com](mailto:haya@microsoft.com)).


## Citation

If you find our work helpful, please consider citing 
```
@article{zheng2021farl,
  title={General Facial Representation Learning in a Visual-Linguistic Manner},
  author={Zheng, Yinglin and Yang, Hao and Zhang, Ting and Bao, Jianmin and Chen, Dongdong and Huang, Yangyu and Yuan, Lu and Chen, Dong and Zeng, Ming and Wen, Fang},
  journal={arXiv preprint arXiv:2112.03109},
  year={2021}
}
```


## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
