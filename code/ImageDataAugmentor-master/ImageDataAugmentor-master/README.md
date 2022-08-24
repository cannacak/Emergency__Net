>**NOTICE!**
> * Support has moved from `keras` to `tensorflow.keras` framework. 
> * There were large updates in Dec 2020, see in [Changelog](CHANGELOG.md) what has changed.

# ImageDataAugmentor
`ImageDataAugmentor` is a custom image data generator for `tensorflow.keras` 
that supports `albumentations`.

To learn more about:
* `ImageDataGenerator`, see:
  https://www.tensorflow.org/api_docs/python/tf/keras/preprocessing/image/ImageDataGenerator
* `albumentations`, see:
  https://github.com/albumentations-team/albumentations


### Installation 
> For the installation of the prerequisites, see these two gists: [NVIDIA-driver installation](https://gist.github.com/mjkvaak/5769ebdfbf6598b3fd0ece66b3c12a3c) and [TF2.x installation](https://gist.github.com/mjkvaak/ad0fea057d5a733dc76524825f21dee2)

```shell
$ pip install git+https://github.com/mjkvaak/ImageDataAugmentor
```

### How to use
The usage is analogous to `tensorflow.keras.ImageDataGenerator` with
the exception that the image transformations will be generated using
external augmentations library `albumentations`.
> Tip: Complete list of `albumentations.transforms` can be 
> found [here](https://albumentations.ai/docs/api_reference/augmentations/transforms/). 
> See also [this](https://albumentations-demo.herokuapp.com/) handy tool for testing
> the different transforms.

The most notable added features are:
* Augmentations are passed to `ImageDataAugmentor` as a single `albumentations` transform
  (e.g. `albumentations.HorizontalFlip()`) or a composition of multiple transforms as
  `albumentations.Compose` object
* `albumentations` can transform various types of data, e.g. imagery, segmentation mask, 
  bounding box and keypoints. 
  `input_augment_mode` (resp. `label_augment_mode`) can be used to select which type
  of transforms to apply to the (model) inputs (resp. model labels) 
* `.show_data()` can be used to visualize a random bunch of images 
  generated by `ImageDataAugmentor`

Below are a few examples of some commonly encountered use cases.
More complete examples can be found in [`./examples`](./examples) folder.

**Example of using `.flow_from_directory(directory)` with `albumentations`**:
```python
import tensorflow as tf
from ImageDataAugmentor.image_data_augmentor import *
import albumentations
...
    
AUGMENTATIONS = albumentations.Compose([
    albumentations.Transpose(p=0.5),
    albumentations.Flip(p=0.5),
    albumentations.OneOf([
        albumentations.RandomBrightnessContrast(brightness_limit=0.3, contrast_limit=0.3),
        albumentations.RandomBrightnessContrast(brightness_limit=0.1, contrast_limit=0.1)
    ],p=1),
    albumentations.GaussianBlur(p=0.05),
    albumentations.HueSaturationValue(p=0.5),
    albumentations.RGBShift(p=0.5),
])

# dataloaders
train_datagen = ImageDataAugmentor(
        rescale=1./255,
        augment=AUGMENTATIONS,
        preprocess_input=None)
train_generator = train_datagen.flow_from_directory(
        'data/train',
        target_size=(224, 224),
        batch_size=32,
        class_mode='binary')
val_datagen = ImageDataAugmentor(rescale=1./255)
validation_generator = val_datagen.flow_from_directory(
        'data/validation',
        target_size=(224, 224),
        batch_size=32,
        class_mode='binary')
#train_generator.show_data() #<- visualize a bunch of augmented data

# train the model with real-time data augmentations
model.fit(
        train_generator,
        steps_per_epoch=len(train_generator),
        epochs=50,
        validation_data=validation_generator,
        validation_steps=len(validation_generator))
...
```

**Example of using `.flow(x, y)` with `albumentations`:**
```python
import tensorflow as tf
from ImageDataAugmentor.image_data_augmentor import *
import albumentations
...

AUGMENTATIONS = albumentations.Compose([
    albumentations.HorizontalFlip(p=0.5), # horizontally flip 50% of all images
    albumentations.VerticalFlip(p=0.2), # vertically flip 20% of all images
    albumentations.ShiftScaleRotate(p=0.5)
],)  

# fetch data
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.cifar10.load_data()
num_classes = len(np.unique(y_train))
y_train = tf.keras.utils.to_categorical(y_train, num_classes)
y_test = tf.keras.utils.to_categorical(y_test, num_classes)

# dataloaders
datagen = ImageDataAugmentor(
    featurewise_center=True,
    featurewise_std_normalization=True,
    augment=AUGMENTATIONS, 
    validation_split=0.2
)
# compute quantities required for featurewise normalization
datagen.fit(x_train, augment=True)
train_generator = datagen.flow(x_train, y_train, batch_size=32, subset='training')
validation_generator = datagen.flow(x_train, y_train, batch_size=32, subset='validation')
# train_generator.show_data()

# train the model with real-time data augmentations
model.fit(
  train_generator,
  steps_per_epoch=len(train_generator),
  epochs=50,
  validation_data=validation_generator,
  validation_steps=len(validation_generator)
)

# evaluate the model with test data
test_datagen = ImageDataAugmentor(
    featurewise_center=True,
    featurewise_std_normalization=True,
    augment=albumentations.HorizontalFlip(p=0.5), 
)
test_datagen.mean = datagen.mean #<- stats from training dataset 
test_datagen.std = datagen.std #<- stats training dataset
test_generator = test_datagen.flow(x_test, y_test, batch_size=32)
model.evaluate(test_generator)
```    

**Example of using `.flow_from_directory()` with masks for segmentation with `albumentations`:**
```python
import tensorflow as tf
from ImageDataAugmentor.image_data_augmentor import *
import albumentations
...

SEED = 123
AUGMENTATIONS = albumentations.Compose([
  albumentations.HorizontalFlip(p=0.5),
  albumentations.ElasticTransform(),
])

# Assume that DATA_DIR has subdirs "images" and "masks", 
# where masks have been saved as grayscale images with pixel value
# denoting the segmentation label
DATA_DIR = ... 
N_CLASSES = ... # number of segmentation classes in masks

def one_hot_encode_masks(y:np.array, classes=range(N_CLASSES)):
    ''' One hot encodes target masks for segmentation '''
    y = y.squeeze()
    masks = [(y == v) for v in classes]
    mask = np.stack(masks, axis=-1).astype('float')
    # add background if the mask is not binary
    if mask.shape[-1] != 1:
        background = 1 - mask.sum(axis=-1, keepdims=True)
        mask = np.concatenate((mask, background), axis=-1)
    return mask

img_data_gen = ImageDataAugmentor(
    augment=AUGMENTATIONS, 
    input_augment_mode='image', 
    validation_split=0.2,
    seed=SEED,
)
mask_data_gen = ImageDataAugmentor(
    augment=AUGMENTATIONS, 
    input_augment_mode='mask', #<- notice the different augment mode
    preprocess_input=one_hot_encode_masks,
    validation_split=0.2,
    seed=SEED,
)
print("training:")
tr_img_gen = img_data_gen.flow_from_directory(DATA_DIR, 
                                              classes=['images'], 
                                              class_mode=None,
                                              subset="training", 
                                              shuffle=True)
tr_mask_gen = mask_data_gen.flow_from_directory(DATA_DIR, 
                                                classes=['masks'],
                                                class_mode=None, 
                                                color_mode='gray', #<- notice the color mode
                                                subset="training",
                                                shuffle=True)
print("validation:")
val_img_gen = img_data_gen.flow_from_directory(DATA_DIR, 
                                               classes=['images'],
                                               class_mode=None,
                                               subset="validation", 
                                               shuffle=True)
val_mask_gen = mask_data_gen.flow_from_directory(DATA_DIR, 
                                                 classes=['masks'], 
                                                 class_mode=None, 
                                                 color_mode='gray', #<- notice the color mode
                                                 subset="validation",
                                                 shuffle=True)
#tr_img_gen.show_data()
#tr_mask_gen.show_data()

train_generator = zip(tr_img_gen, tr_mask_gen)
validation_generator = zip(tr_img_gen, tr_mask_gen)

# visualize images
rows = 5
image_batch, mask_batch = next(train_generator)
fix, ax = plt.subplots(rows,2, figsize=(4,rows*2))
for i, (img,mask) in enumerate(zip(image_batch, mask_batch)):
    if i>rows-1:
        break
    ax[i,0].imshow(np.uint8(img))
    ax[i,1].imshow(mask.argmax(-1))
    
plt.show()

# train the model with real-time data augmentations
model.fit(
  train_generator,
  steps_per_epoch=len(train_generator),
  epochs=50,
  validation_data=validation_generator,
  validation_steps=len(validation_generator)
)
...

```


## Citing (BibTex):<br />
```
@misc{Tukiainen:2019,
  author = {Tukiainen, M.},
  title = {ImageDataAugmentor},
  year = {2019},
  publisher = {GitHub},
  journal = {GitHub repository},
  howpublished = {https://github.com/mjkvaak/ImageDataAugmentor/} 
}
```

## License
This project is distributed under [MIT license](LICENSE). 
The code is heavily adapted from
https://github.com/keras-team/keras-preprocessing/blob/master/keras_preprocessing/ (also MIT licensed)