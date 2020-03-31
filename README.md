# MGVTON
**Unofficial PyTorch reproduction of MGVTON.**


I'm reproducing [MGVTON] (https://arxiv.org/pdf/1902.11026.pdf). The implementation is based on the reproduction of SWAPNET (https://github.com/andrewjong/SwapNet)

## Contributing

The following instruction is from author of SwapNet:

# Installation

This repository is built with PyTorch. I recommend installing dependencies via [conda](https://docs.conda.io/en/latest/).

With conda installed run:
```
cd SwapNet/
conda env create  # creates the conda environment from provided environment.yml
conda activate swapnet
```
Make sure this environment stays activated while you install the ROI library below!

## Install ROI library (required)
I borrow the ROI (region of interest) library from [jwyang](https://github.com/jwyang/faster-rcnn.pytorch/tree/pytorch-1.0). This must be installed for this project to run. Essentially we must 1) compile the library, and 2) create a symlink so our project can find the compiled files.

**1) Build the ROI library**
```
cd ..  # move out of the SwapNet project
git clone https://github.com/jwyang/faster-rcnn.pytorch.git # clone to a SEPARATE project directory
cd faster-rcnn.pytorch
git checkout pytorch-1.0
pip install -r requirements.txt
cd lib/pycocotools
```
Important: now COMPLETE THE INSTRUCTIONS [HERE](https://github.com/jwyang/faster-rcnn.pytorch/issues/402#issuecomment-448485129)!!
```
cd ..  # go back to the lib folder
python setup.py build develop
```
**2) Make a symlink back to this repository.**
```
ln -s /path/to/faster-rcnn.pytorch/lib /path/to/swapnet-repo/lib
```
Note: symlinks on Linux tend to work best when you provide the full path.

# Dataset
Data in this repository must start with the following:
- `texture/` folder containing the original images. Images may be directly under this folder or in sub directories.

The following must then be added from preprocessing(see the Preprocessing section below):
- `body/` folder containing preprocessed body segmentations 
- `cloth/` folder containing preprocessed cloth segmentations
- `rois.csv` which contains the regions of interest for texture pooling
- `norm_stats.json` which contain mean and standard deviation statistics for normalization

## Deep Fashion
The dataset cited in the original paper is [DeepFashion: In-shop Clothes Retrieval](http://mmlab.ie.cuhk.edu.hk/projects/DeepFashion/InShopRetrieval.html). If you plan to preprocess the images yourself, download the images zip and move the image files under `data/deep_fashion/texture`.

Alternatively, I've preprocessed the Deep Fashion image dataset already. The full preprocessed dataset can be downloaded here: https://drive.google.com/open?id=1oGE23DCy06zu1cLdzBc4siFPyg4CQrsj. If you want to use your own dataset, please follow the preprocessing instructions below while substituting "deep_fashion" for the name of your dataset.

Otherwise, jump ahead to the Training section.

## (Optional) Create Your Own Dataset
If you'd like to take your own pictures, move the data into `data/YOUR_DATASET/texture`.

### Preprocessing
The images must be preprocessed into BODY and CLOTH segmentation representations. These will be input for training and inference.

#### Body Preprocessing
The original paper cited [Unite the People](https://github.com/classner/up) (UP) to obtain body segmentations; however, I ran into trouble installing Caffe to make UP work (probably due to its age). 
Therefore, I instead use [Neural Body Fitting](https://arxiv.org/abs/1808.05942) (NBF). [My fork of NBF](https://github.com/andrewjong/neural_body_fitting-for-SwapNet) modifies the code to output body segmentations and ROIs in the format that SwapNet requires. 

1) Follow the instructions in my fork. You must follow the instructions under "Setup" and "How to run for SwapNet". Note NBF uses TensorFlow; I suggest using a separate conda environment for NBF's dependencies.

2) Move the output under `data/deep_fashion/body/`, and the generated rois.csv file to `data/deep_fashion/rois.csv`.

*Caveats:* neural body fitting appears to not do well on images that do not show the full body. In addition, the provided model seems it was only trained on one body type. I'm open to finding better alternatives.

#### Cloth Preprocessing
The original paper used [LIP\_SSL](https://github.com/Engineering-Course/LIP_SSL). I instead use the implementation from the follow-up paper, [LIP\_JPPNet](https://arxiv.org/pdf/1804.01984.pdf). Again, [my fork of LIP\_JPPNet](https://github.com/andrewjong/LIP_JPPNet-for-SwapNet) outputs cloth segmentations in the format required for SwapNet.

1) Follow the installation instructions in the repository. Then follow the instructions under the "For SwapNet" section.

2) Move the output under `data/deep_fashion/cloth/`

#### Calculate Normalization Statistics
This calculates normalization statistics for the preprocessed body image segmentations, under `body/`, and original images, under `texture/`. The cloth segmentations do not need to be processed because they're read as 1-hot encoded labels.

Run the following: `python util/calculate_imagedir_stats.py data/deep_fashion/body/ data/deep_fashion/texture/`. The output should show up in `data/deep_fashion/norm_stats.json`.

# Training

Train progress can be viewed by opening `localhost:8097` in your web browser.

1) Train warp stage
```
python train.py --name deep_fashion/warp --model warp --dataroot data/deep_fashion
```
Sample visualization of warp stage:
<p align="center">
<img src="media/warp_train_example.png" alt="warp example" width=500>
</p>

2) Train texture stage
```
python train.py --name deep_fashion/texture --model texture --dataroot data/deep_fashion
```
Below is an example of train progress visualization in Visdom. The texture stage draws the input texture with ROI 
boundaries (left most), the input cloth segmentation (second from left), the generated 
output, and target texture (right most).

<p align="center">
<img src="media/texture_train_example.png" alt="texture example" width=600>
<img src="media/texture_custom_data_example.png" alt="texture example" width=600>
</p>

# Inference
To download pretrained models, download the `checkpoints/` folder from [here](https://drive.google.com/open?id=1kaeHlGf9h3vZLBxr4D5yQUgToxEUWuMh) and extract it under the project root. Please note that these models are not yet perfect, requiring a fuller exploration of loss hyperparameters and GAN objectives.


Inference will run the warp stage and texture stage in series.

To run inference on deep fashion, run this command:
```
python inference.py --checkpoint checkpoints/deep_fashion \
  --dataroot data/deep_fashion \
  --shuffle_data True
```
`--shuffle_data True` ensures that bodys are matched with different clothing for the transfer. 
By default, only 50 images are run for inference. This can be increased by setting the value of `--max_dataset_size`.


Alternatively, to translate clothes from a specific source to a specific target:
```
python inference.py --checkpoint checkpoints/deep_fashion \
  --cloth_dir [SOURCE] --texture_dir [SOURCE] --body_dir [TARGET]
```
Where SOURCE contains the clothing you want to transfer, and TARGET contains the person to place clothing on.

# Comparisons to Original MGVTON
### Similarities
- Stage I
  - [x] Test
- Stage II
  - [x] Test
- Stage III: Refinement render
  - [x] Test

### Differences


### TODO:
- [ ] Implement Stage I: generator and Discriminator
- [ ] Implement Geometric matching module GMM(body shape, target cloth mask) --> warped cloth mask
- [ ] Implement Geometric Matcher GMatcher(References parsing) --> Synthesys Parsing
- [ ] Implement Warp-GAN: Generator and Discriminator
- [ ] Implementation of refinement render
- [ ] Add regularize to GMM and GMatcher
- [ ] DeformableGAN --> Decomposed DeformableGAN

# What's Next?


# Credits

