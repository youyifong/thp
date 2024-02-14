# TSP - The Seattle Pipeline for Deep Learning Methods for Cell and Tissue Imaging Analysis

## Pipeline steps

Pre-processing

- background reduction and optimizing brightness (ZKW DataWizard)
- eliminating/blacking out false or artificial fluorescence signals generated by debris introduced into the chip chamber during the multiple staining cycles using Fiji (ImageJ)
- alignment of individual position images
- stitching
- cropping

Analysis

- cell segmentation
- spatial analysis for single markers
- multiplexed analysis
- statistical analysis


## Cell segmentation with cellpose 
> python -m tsp runcellpose --f '*.png' --l [0,0]  --model cytotrain7

Required input:
- --f A list of file names separated by comma, containing no spaces, e.g., [xx1.png,xx2.png], or a pattern to match file names, e.g. 'xx*.png'. The quotes around file name pattern are required, because otherwise it will be expanded by shell. 

- --l A list of 2 numbers indicating [cytoplasm channel, nucleus channel]. Each value can be 0, 1 (red), 2 (green), and 3 (blue)
For the cytoplasm channel, 0 means grayscale; for the nucleus channel, 0 means no nuclei. 
E.g. [3,0] that means cytoplasm signal is in the blue channel and no nuclei; [0,0] means cytoplasm is taken from a grayscale image and no nuclei.

- --model Cellpose model to use, e.g. cyto, cytotrain7/c7, c7s, c7s2, c7s3, cs. cyto is the Cellpose default, cytotrain7 is the model from Sunwoo et al. c7s is c7-shrinked, cs is cyto-shrinked, c7s2 is trained starting from c7s with more data, c7s3 is trained starting from c7s2 with more data


Optional input:
- --saveimgwithmasks Without this option, _masks.png is a grayscale image with masks outlines. With this option, the intensities are also present in this file

- --saveflow If present, .npy file containing flow, diam and masks will be saved.

- --saveroi If present, mask outline roi files will be saved.

- --s If present, additional image files will be saved.

- --flow Flow threshold. Larger values => more masks. This may be the most important parameter to adjust at inference. Default 0.4. 

- --cellprob Cell probability threshold. Smaller values => fewer and larger masks, fewer because masks they are close start to merge. Default 0. 

- --d Cell diameter. Default 0. 

- --normalize99 If present, normalize image from 1 to 99 percentiles as is the default behavior in Cellpose. If not present, normalize with reference to min to max instead, which helps when there are a lot of black pixels in the image.

- --min_size Post-processing parameter, min_size, is changed from 15 (default) to the specified value. If a cell consists of the number of pixels less than min_size, the cell is removed. 

- --min_avgintensity Post-processing parameter, min_average_intensity, is changed from 0 (default) to the specified value. If an average intensity of pixels in a cell is less than min_average_intensity, the cell is removed. 

- --min_totalintensity Post-processing parameter, min_total_intensity, is changed from 0 (default) to the specified value. If the total intensity of pixels in a cell is less than min_total_intensity, the cell is removed. 


Output 

All files are saved to the current working directory, which may or may not be the same directory as the directory containing the input image files.

- cellpose_counts_xxx.txt: number of predicted masks for each image. xxx refers to a date string

- _masks.csv: a text file containing info about the size of each predicted mask and x-y coordinate of center pixel of each predicted mask  

- _m_{model}.png: a grayscale image with mask indices

- _o_{model}.png: a grayscale image with masks outlines. If --saveimgwithmasks is present, the original image is saved with the masks in an RGB image

- _masks_fill.png (with --s): a grayscale file containing the solid fill of the predicted cell masks 

- _masks_point.png (with --s): an RGB file with mask centers plotted in the R channel and image copied from input if image.ndim is 3 or image in the G and B channels if image.ndim is 2

- _masks_text.png (with --s):  an RGB file with mask indices plotted in the R channel and image copied from input if image.ndim is 3 or image in the G and B channels if image.ndim is 2

- _seg.npy (with --saveflow): cellpose output file containing overall info about predicted masks and parameters used in prediction. This file can be huge.

- _cp_outlines.txt (with --saveroi): a text file that can be converted to roi by ImageJ if imagej_roi_converter.py from Cellpose is run as a macro after opening image file



## Cell phenotyping 
> python -m tsp cellphenotyping --f [file1.png,file2.png,file3.png] --m [Mask,Mask] --c [0.5,0.5] --p [True,False] --n [marker2,marker3] --c2 [0.5,0.5]

The command above looks for marker1+ marker2+ marker3- cells.  Let K be the number of markers. 

Required input:
- --f List of K file names. For the first file, it is the mask file name. For the rest, it depends on the method. For mask-mask analyses, it is the mask file name; **for mask-intensity analyses, it is the image file name**. 

- --m List of K-1 values for methods for finding multistained cells.
    * Mask:: Overlap between A and B is computed for individual B cells, not all B cells.
    * Intensity_mean
    * Intensity_median
    * Intensity_total
    * Intensity_pos: pixels above a threshold, default 100, are called positive. A cell with over a default 50% positive pixels is called positive.

- --c List of K-1 cutoff values for deciding if markers are present.

- --c2 List of K-1 cutoff values for deciding if markers are present. 

- --p List of K-1 values from True or False. Marker is required to be present if True and abscent if False.

- --n List of K-1 names for markers (first marker excluded). 

Optional input:

- --l List of K-1 channels. If the images are grayscale, the argument is not needed. If the images are RBG, the argument is used to extract the channels of interest from the two images: 1 R, 2 G, 3 B, e.g. [2,2]

- --s If present, additional image files will be saved.

- --mask_dilations A list of integers. When doing mask-intensity analyses, a positive/negative mask_dilation causes the masks the be dilated/eroded by that number of pixels on all sides. E.g., [1,-1]. 

Output 

- _counts_multistain.txt: cell counts, one row for each additional marker

- _counts_lastcutoff.txt: cell counts for a series of cutoffs for the last marker

- _masks.csv: a text file containing info about the size of each predicted mask and x-y coordinate of center pixel of each predicted mask, last marker only

- _o.png: K grayscale files with mask outlines 

- _m.png: K grayscale files with mask ids 

- _masks_fill.png (with --s): K grayscale files with masks as filled shapes

<!--- - _point.png (with --s): K grayscale files with each mask drawn as a point. --->




## Intensity statistics 

> python -m tsp intensityanalysis --maskfile xx_masks_id.png  --f '*.png'  

Load masks in xx_seg.npy and measures the intensities of each markers in the list of image files. 

Required input

- --maskfile A _seg.npy file containing the mask info.

- --f A list of file names separated by comma, containing no spaces, e.g., [xx1.png,xx2.png], or a pattern to match file names, e.g. 'xx*.png'. The quotes around file name pattern are required, because otherwise it will be expanded by shell. 

Optional iput

- --l An integer: 1 (red), 2 (green), and 3 (blue). Not required if image files are grayscale.

Output 
- _MFI.csv: contains the x-y coordinates and MFI for each mask 


## Image stitching

Run command in the directory where the json file resides.

> python -m tsp stitchimages --json AS_Les_layout_by_percentage.json --imgfolder CD3

- --json A configuration file 

- --imgfolder Name of the folder containing images at all positions to be stitched together

Output 

- A new image file named *wholeslide*.png will be saved in the current directory.

 

## Image alignment

> python -m tsp alignimages --ref_image xx  --f '*.png'

- --f A list of file names separated by comma, containing no spaces, e.g., [xx1.png,xx2.png], or a pattern to match file names, e.g. 'xx*.png'. The quotes around file name pattern are required, because otherwise it will be expanded by shell. If the list of files contain ref_image, ref_image will be removed

Optional input:

- --l Channels. If the images are grayscale, the argument is not needed. If the images are RBG, the argument is used to extract the channels of interest from the two images: 1 R, 2 G, 3 B, e.g. [2,2]

Output 

- A new image file named _aligned.png will be saved.

 

## Collapse images

> python -m tsp collapseimages --f 'xx*.png'  --saveas newfilename.png

Required input

- --f A list of file names separated by comma, containing no spaces, e.g., [xx1.png,xx2.png], or a pattern to match file names, e.g. 'xx*.png'. The quotes around file name pattern are required, because otherwise it will be expanded by shell. 

- --saveas Output file name.

Optional input:

- --mode Default to max, other allowed values: avg.

Output 

- A new image file with will be saved.

 

## Compute distance from cell center to a boundary polyline

Appends shortest distance from cell centers to boundary lines as new columns and save as a new _d2b.csv file.

> python -m tsp dist2boundary --cells masks.csv  --boundaryroi boundary.roi  --saveas col_name

- --cells: a csv file containing the cell center coordinates

- --boundaryroi: roi files defining the boundary, can be either [file1.roi,file2.roi] or 'file*.roi'




## Compute region membership

Appends region membership as new columns and save as a new _regmem.csv file.

> python -m tsp regionmembership --cells masks.csv  --regionroi [region1.roi,region2.roi]

- --cells: a csv file containing the cell center coordinates

- --boundaryroi: roi files defining the boundary, can be either [file1.roi,file2.roi] or 'file*.roi'

 

## Working with masks

### Convert roi files into mask png files

First, unzip the roi zip file, e.g.

> unzip CF_Les_Pos7_CD3+CD8+_RoiSet_865.zip -d CF_Les_Pos7_CD3+CD8+_RoiSet_865 

The program creates two png files, one mask file and one mask outline file. 

> python -m tsp roifiles2mask --f "eg1/*" --width 1392 --height 1040 --saveas fname.png
 
- --f A list of file names separated by comma, containing no spaces, e.g., [xx1.png,xx2.png], or a pattern to match file names, e.g. 'xx*.png'. The quotes around file name pattern are required, because otherwise it will be expanded by shell. 

- --saveas Output mask file name

Optional input:

- --width: dimension of the image

- --height: dimension of the image

- --imagefile: can also be provided in lieu of width and height to provide dimension info


### Compare two mask files to get AP
> python -m tsp AP --mask1 mask1.png --mask2 mask2.png 
> python -m tsp AP --mask1 M926910_Position1_CD45+CD56+_seg.npz --mask2 M926910_Position1_CD45+CD56+CD3-CD271-_seg.npz 

### Compare two folders of masks
> python -m tsp checkprediction --metric   --predfolder   --gtfolder   --min_size

### Add mask1 in red, mask2 in green (optional), and overlap in yellow, all on top of images
> python -m tsp overlaymasks

### Add mask2 in green and highlight tp (based on comparing with mask1) in yellow, on top of images
> python -m tsp colortp

### Make outlines from masks
> python -m tsp mask2outline --f '*.png' 
> python -m tsp mask2outline --f '*.png' --color [green,red]


