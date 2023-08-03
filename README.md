# ProCanLoad

![ProCAncer-I](https://www.procancer-i.eu/wp-content/uploads/2020/07/logo.png)

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-dd4814?logo=ubuntu&logoColor=white&style=flat-square)](https://ubuntu.com/)
[![Windows](https://img.shields.io/badge/Windows-11-0078d4?logo=windows&logoColor=white&style=flat-square)](https://www.microsoft.com/en-us/windows/)

ProCanLoad is an image loader for ProCAncer-I dataset and segmentation files. After you filtered the .parquet file on which series to keep, and drop out the unnecessary ones. You can input the modified parquet to:

i) Create a dictionary with all the patients, stored as image_loader.json.

ii) Extract the available segmentations from the segmentation .dcm file.

iii) Extract DICOM images to Niifty (.nii.gz) which their path will be stored in nifty_files.nii

iv) Provides a logger to catch any warning, for the user to check, stored in issues/issues_image_loader.json

The package makes use of SimpleITK and pydicom.

# Install repository:

run 
```bash
    pip install git+http://git.procancer-i.eu/ChKalantzopoulos/ProCanLoad.git
```

# Demonstration

In data directory, in UC1FilterData.ipynb, you can see the followed process to filter the ecrfs-series-20230801.parqet (MD5: e337672d2c2b9dacfcc0577a527882be) & segments-20230801.parquet (MD5: bc214112612fb1ac25e299a9df83734a).

This will create UseCase1-v1.parquet and Seg_UseCase1-v1.parquet which will be fed to this package.

In LoaderDEMO.ipynb, it is shown how to read the images and extract them in Niifty format.

# Image directory organization

```
{your-directory}
        ├─  patient_id
        |   ├─  study_uid
        │   |   ├─  series_uid
        |   |   |   ├─  *.dcm
        |   |   |   .
        |   |   |   └─    ...
        |   |   .
        |   |   └─ ...
        |   .
        |   └─  ...
        .
        └─  ...
```
# SimpleITK VS Pydicom

- SimpleITK misses encoded b-values in DWI:

  Even though, simpleITK can handle most of the cases, it was found to fail loading a certain type of encoded b-value (returns null!). In addition, decoders are required
  
  This is the main reason why pydicom was prefered. Pydicom in most of the cases, it was already de-coded. However, some cases needed further decoding (More in DWI b-values)

- Pydicom does not automatically transform the image according to vendor's VOI LUT.

  SimpleITK was used to load the image, since any transformation required by the given dcm files, is performed automatically.

- SimpleITK failed to load segmentation files with only one slice.

  Pydicom is been used for loading segmentation. Also, it is easier to detect to put the segmentation slice on the right location.

# Actions in ProCanLoad

Check the LoaderDEMO.ipynb in how to utilize the package.

## Ordering image slices

ProCanLoad main purpose was to pick and set the image slices in the right order. For this, DICOM tag Image Position (Patient) (0020,0032) is been used. This tag is necessary for any image transformation (e.g. resample)

Other alternatives were found not to work for all the cases ( Slice Location (0020,1041), Instance Number (0020,0013) ), as they were missing or did not change between the slices. Although, this could be an issue of older versions of parquet.

Image Position returns the location on X, Y, Z. By calculating the cross product from image directions, the main direction is found and then we use it for ordering the slices (e.g. if image is Axial, Z position is used from Image Position).

## ADC rescale type

Some ADCs were found to have a very low voxel value (e.g. max voxel intensity value < 10).

From :
https://radiopaedia.org/articles/apparent-diffusion-coefficient-1 and

dicom tag Rescale Type (0028,1054) returning value "10^-3 mm^2/s"

It is clear that ADC having thousand value are stored in 10^-6 mm^2/s and unit values are are 10^-3 mm^2/s.

So, if an ADC's max voxel intensity value is unit value, then it is multiplied by 1000 before it is stored in .nii.gz. This transformation is logged in issues.

## DWI

In the special case of DWI sequences, the ordering will be also performed for each b-value given by the image slice

According to the consortium, the highest b-values are prefered.

However, there might be situations where multi-series' b-values are unknown. 

In this scenario, according to the formula S_DWI = S_b=0 * e^(-b * D) https://radiopaedia.org/articles/diffusion-weighted-imaging-2

Assuming for the same patient, with unknown b-values, S_b=0 and D are constants, then if the b becomes larger, then S_DWI becomes smaller, for the slice in the same location.

Another ordering is performed. For each slice position found in dcm files, the max and mean slice's intensity value is found. Then we order the slices by larger to smaller. This results in 'Unknown' having the slices with smaller be value, followed by Unknown-X which will have smaller max_mean value, thus higher b-value.

## Segmentations

For label segmentations, pydicom is utilized. First, we take the name of the segmentations reside in the dcm file and then we extract them to a zero array which has the same shape as the T2w image.

To put the slices on the correct position (0008,0018) SOP Instance UID from T2w and (0008,1155) Referenced SOP Instance UID is been used.

#logger messages.

For now logger may return these warnings/ issues:


* MissingBValue: B-value for at least one .dcm file was not found
* OneSliceSegmentation: Segmentation dcm consists of one slice (2D array)
* DWIMultiSeriesNotSameSliceNumber: In the DWI. for all b-values are unknown, there are at least one case with different number of slices, take a look of this warning, since it can cause problems in ordering the slices (e.g. slice may belong to another b-value)
* ADCRescaleTypeMicro: Shows which ADCs have been multiplied by 1000
* SameBValueFound: At least one slice was found, whith the same b-value and image postion.
* bad_format: in ImageLoader - add_column: user should use a list or a string with the name of the columns. In the case of string, used ',' delimeter.
* select_col: column selected does not exist in parquet.
* FileNotFound: file was not found inside the working directory
* DuplicateDetected: This error it was appearing on the previous version of the dataset, it is fixed now but kept to assure no duplicates found.
* SameOriginFound: For sequences except DWI, at least one slice with same image location but not duplicate was found.
* MultiplePlanesFound: The series is Multi-Planar
* ZeroMaskFound: A given labeled segmentation is zeroes


# Authors

Kalantzopoulos Charalampos, xkalantzopoulos@gmail.com

Zaridis Dimitrios dimzaridis@gmail.com

Mylona Eugenia mylona.eugenia@gmail.com

Nikolaos Tachos ntachos@gmail.com

# Thank you

Me and my team, we will like to thank you for using AIPassport. You can send any message to xkalantzopoulos@gmail.com.
