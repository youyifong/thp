# TSP - The Seattle Pipeline for Deep Learning Methods for Cell and Tissue Imaging Analysis

## Pipeline steps

Pre-processing

- background reduction and optimizing brightness using the ZKW DataWizard application
- eliminating/blacking out false or artificial fluorescence signals generated by debris introduced into the chip chamber during the multiple staining cycles using Fiji (ImageJ)
- stitching (?)
- alignment using tsp
- cropping

Analysis

- cell segmentation
- compute distance to boundary
- compute region membership
- multistaining analysis
- statistical analysis


## Cell segmentation with cellpose 
> python -m tsp runcellpose --f '*.png' --l [0,0]  --model cytotrain7

Required input:
- --f is required and tells the program which image files to segment. The quotes around file name pattern are required, because otherwise it will be expanded by shell. '`' is not recognized in the path.

- --l A list of 2 numbers indicating [cytoplasm channel, nucleus channel]. Each value can be 0, 1 (red), 2 (green), and 3 (blue)
For the cytoplasm channel, 0 means grayscale; for the nucleus channel, 0 means no nuclei. 
E.g. [3,0] that means cytoplasm signal is in the blue channel and no nuclei; [0,0] means cytoplasm is taken from a grayscale image and no nuclei.

- --model Cellpose model to use, including cyto and cytotrain7. cyto is the Cellpose default, cytotrain7 is the model from Sunwoo et al.


Optional input:
- --saveimgwithmasks Without this option, _masks.png is a grayscale image with masks outlines. With this option, the intensities are also present in this file

- --saveflow If present, .npy file containing flow, diam and masks will be saved.

- --saveroi If present, mask outline roi files will be saved.

- --s If present, additional image files will be saved.

- --flow Flow threshold. Larger values => more masks. This may be the most important parameter to adjust at inference. Default 0.4. 

- --cellprob Cell probability threshold. Smaller values => fewer and larger masks, fewer because masks they are close start to merge. Default 0. 

- --d Cell diameter. Default 0. 

- --normalize100 If present, normalize image from min to max instead of 1 to 99 percentiles. This helps when there are few cells in the image.

- --min_size Post-processing parameter, min_size, is changed from 15 (default) to the specified value. If a cell consists of the number of pixels less than min_size, the cell is removed. 

- --min_avgintensity Post-processing parameter, min_average_intensity, is changed from 0 (default) to the specified value. If an average intensity of pixels in a cell is less than min_average_intensity, the cell is removed. 

- --min_totalintensity Post-processing parameter, min_total_intensity, is changed from 0 (default) to the specified value. If the total intensity of pixels in a cell is less than min_total_intensity, the cell is removed. 


Output 

All files are saved to the current working directory.

- cellpose_counts_xxx.txt: number of predicted masks for each image. xxx refers to a date string

- _masks.csv: a text file containing info about the size of each predicted mask and x-y coordinate of center pixel of each predicted mask  

- _masks.png: a grayscale image with mask indices

- _masks_outline.png: a grayscale image with masks outlines. If --saveimgwithmasks is present, the original image is saved with the masks in an RGB image

- _masks_fill.png (with --s): a grayscale file containing the solid fill of the predicted cell masks 

- _masks_point.png (with --s): an RGB file with mask centers plotted in the R channel and image copied from input if image.ndim is 3 or image in the G and B channels if image.ndim is 2

- _masks_text.png (with --s):  an RGB file with mask indices plotted in the R channel and image copied from input if image.ndim is 3 or image in the G and B channels if image.ndim is 2

- _seg.npy (with --saveflow): cellpose output file containing overall info about predicted masks and parameters used in prediction. This file can be huge.

- _cp_outlines.txt (with --saveroi): a text file that can be converted to roi by ImageJ if imagej_roi_converter.py from Cellpose is run as a macro after opening image file



## Cell phenotyping 
> python -m tsp cellphenotyping --f [file1.png,file2.png,file3.png] --m [Mask,Mask] --c [0.5,0.5] --p [True,False] --n [marker2,marker3] 

The command above looks for marker1+ marker2+ marker3- cells.  Let K be the number of markers. 

Required input:
- --f List of K file names. In this example, for the first file, the program expects to find both file1.png and file1_masks_id.png. For the following files, e.g. file2.png, the program expects to find the mask file (_masks_id.png) if the method is Mask and the image file otherwise. 

- --m List of K-1 values from Mask, Intensity_avg_all, or Intensity_total. Methods for finding multistained cells. Under mask, overlap between A and B is computed for individual B cells, not all B cells.

- --c List of K-1 cutoff values for deciding if markers are present.

- --p List of K-1 values from True or False. Marker is required to be present if True and abscent if False.

- --n List of K-1 names for markers (first marker excluded). 

Optional input:

- --l List of K-1 channels. If the images are grayscale, the argument is not needed. If the images are RBG, the argument is used to extract the channels of interest from the two images: 1 R, 2 G, 3 B, e.g. [2,2]

- --s If present, additional image files will be saved.

Output 

- _counts_multistain.txt: cell counts, one row for each additional marker

- _counts_lastcutoff.txt: cell counts for a series of cutoffs for the last marker

- _masks.csv: a text file containing info about the size of each predicted mask and x-y coordinate of center pixel of each predicted mask, last marker only

- _masks.png: K grayscale files with mask outlines 

- _masks_id.png: K grayscale files with mask ids 

- _masks_fill.png (with --s): K grayscale files with masks as filled shapes

<!--- - _point.png (with --s): K grayscale files with each mask drawn as a point. --->




## Intensity statistics 

> python -m tsp intensityanalysis --maskfile xx_masks_id.png  --f '*.png'  

Load masks in xx_seg.npy and measures the intensities of each markers in the list of image files. 

Required input

- --maskfile A _seg.npy file containing the mask info.

- --f List of file names matching the pattern.

Optional iput

- --l An integer: 1 (red), 2 (green), and 3 (blue). Not required if image files are grayscale.

Output 
- _MFI.csv: contains the x-y coordinates and MFI for each mask 


## Image alignment

> python -m tsp alignimages --ref_image xx  --image2 xx 

Optional input:

- --l Channels. If the images are grayscale, the argument is not needed. If the images are RBG, the argument is used to extract the channels of interest from the two images: 1 R, 2 G, 3 B, e.g. [2,2]

Output 

- A new image file named _aligned.png will be saved.

 

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

The program creates two png files, one mask file and one mask outline file. 

> python -m tsp roifiles2mask --f "eg1/*" --width 1392 --height 1040 --saveas fname
 
- --f is required and tells the program which image files to segment. The quotes around file name pattern are required, because otherwise it will be expanded by shell

- --saveas is required and tells the output mask png file name

- --width: dimension of the image

- --height: dimension of the image


To unzip, e.g.

> unzip CF_Les_Pos7_CD3+CD8+_RoiSet_865.zip -d CF_Les_Pos7_CD3+CD8+_RoiSet_865 

### Compare two mask files to get AP
> python -m tsp AP --mask1 testmasks/M872956_JML_Position10_CD3_test_masks.png --mask2  testmasks/M872956_JML_Position10_CD3_test_masks.png 
> 
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


