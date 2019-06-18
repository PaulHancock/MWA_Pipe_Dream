# A semi-automated pipeline for the reduction of MWA data

This pipeline aims to provide the core functionality required to download, calibrate, and image MWA data.

The pipeline has been developed for the Pawsey system which uses a SLURM job scheduler.

## Credits
If you use this code, or incorporate it into your own workflow, please give credit to
the contributors and obey the [license](LICENSE).
Please acknowledge the use of this code by citing this repository.

## Project Structure
Central code repo:
- /bin: executable files to be run from the command line
- /lib: libraries and extensions
- /tmpl: templates for job scripts
- /data: storage for common data such as sky models

Within individual project directories:
- /queue: job scripts
- /logs: error and output files from jobs
- /db: coordination database

## scripts and templates
Templates for scripts are `tmpl/*.tmpl`, these are modified by the scripts `bin` and the completed script is then put in `queue/*.sh` and submitted to SLURM.

## track_task.py
Used to track the submission/start/finish/fail of each of the jobs.
Not intended for use outside of these scripts.

### obs_asvo.sh
Use the [ASVO-mwa](https://asvo.mwatelescope.org) service to do the cotter conversion
and then download the resulting measurement set. This replaces the operation of `obs_dl.sh` and `obs_cotter.sh`.

usage:
```
obs_asvo.sh [-d dep] [-c calid] [-n calname] [-s timeav] [-k freqav] [-t] obsnum
  -d dep     : job number for dependency (afterok)
  -c calid   : obsid for calibrator. 
               If a calibration solution exists for calid
               then it will be applied this dataset.
  -n calname : The name of the calibrator.
               Implies that this is a calibrator observation 
               and so calibration solutions will be calculated.
  -m minbad  : The minimum number of bad dipoles requried for a 
               tile to be used (not flagged), default = 2
               NOTE: Currently not supported by asvo-mwa so this is IGNORED.
  -s timeav  : time averaging in sec. default = no averaging
  -k freqav  : freq averaging in KHz. default = no averaging
  -t         : test. Don't submit job, just make the batch file
               and then return the submission command
  obsnum     : the obsid to process
```
uses templates:
- `asvo_dl_cotter.tmpl` (obsnum->OBSNUM/timeav->TRES/freqav->FRES)

### obs_peel.sh
Peel a sky model from a given observation.

Usage:
```
obs_infield_cal.sh [-d dep] [-q queue] [-M cluster] [-p model] [-n minuvm] [-x maxuvm] [-s steps] [-a] [-t] obsnum
  -d dep     : job number for dependency (afterok)
  -q queue   : job queue, default=workq
  -M cluster : cluster, default=zeus
  -p model   : model to peel, 'AO' format
  -n minuvm  : minuv distance in m
  -x maxuvm  : maxuv distance in m
  -s steps   : number of timesteps to average over, default = all
  -a         : turn ON applybeam, default=assume model has beem applied
  -t         : test. Don't submit job, just make the batch file
               and then return the submission command
  obsnum     : the obsid to process
```

uses template:
- `peel.tmpl`


### obs_infield_cal.sh
Generate calibration solutions for an observation using the sources within the
field of view.
The model is generated from the points sources within the FoV that are within
the GLEAM catalogue.
Note that GLEAM does not include all areas of sky, and has some bright sources
cropped.
This calibration is done in a two stage process as per obs_calibrate.sh

Usage:
```
obs_infield_cal.sh [-d dep] [-q queue] [-M cluster] [-c catalog] [-t] obsnum
  -d dep     : job number for dependency (afterok)
  -q queue   : job queue, default=workq
  -M cluster : cluster, default=zeus
  -c catalog : catalogue file to use, default=GLEAM_EGC.fits
  -t         : test. Don't submit job, just make the batch file
               and then return the submission command
  obsnum     : the obsid to process
```

uses templates:
- `infield_cal.tmpl`

### obs_apply_cal.sh
Apply a pre-existing calibration solution to a measurement set.

Usage:
```
obs_apply_cal.sh [-d dep] [-q queue] [-M cluster] [-c calid] [-t] obsnum
  -d dep      : job number for dependency (afterok)
  -q queue    : job queue, default=workq
  -M cluster : cluster, default=zeus
  -c calid    : obsid for calibrator.
                processing/calid/calid_*_solutions.bin will be used
                to calibrate if it exists, otherwise job will fail.
  -t          : test. Don't submit job, just make the batch file
                and then return the submission command
  obsnum      : the obsid to process
```

uses tempaltes:
- `apply_cal.tmpl`


### obs_image.sh
Image a single observation.

Usage: 
```
obs_image.sh [-d dep] [-q queue] [-M cluster] [-s imsize] [-p pixscale] [-c] [-t] obsnum
  -d dep     : job number for dependency (afterok)
  -q queue   : job queue, default=workq
  -M cluster : cluster, default=zeus
  -s imsize  : image size will be imsize x imsize pixels, default 4096
  -p pixscale: image pixel scale, default is 32asec
  -c         : clean image. Default False.
  -t         : test. Don't submit job, just make the batch file
               and then return the submission command
  obsnum     : the obsid to process
```
uses tempaltes:
- `image.tmpl`
  - make a single time/freq image and clean
  - perform primary beam correction on this image.

### obs_im05s.sh 
Image an observation once per 0.5 seconds

Usage:
```
obs_im05s.sh [-d dep] [-q queue] [-M cluster] [-s imsize] [-p pixscale] [-t] obsnum
  -d dep     : job number for dependency (afterok)
  -q queue   : job queue, default=workq
  -M cluster : cluster, default=zeus
  -s imsize  : image size will be imsize x imsize pixels, default 4096
  -p pixscale: image pixel scale, default is 32asec
  -t         : test. Don't submit job, just make the batch file
               and then return the submission command
  obsnum     : the obsid to process
```

uses tempaltes:
- `im05s.tmpl`

### obs_im28s.sh
Image an observation once pert 28 seconds

Usage:
```
obs_im28s.sh [-d dep] [-q queue] [-M cluster] [-s imsize] [-p pixscale] [-c] [-t] obsnum
  -d dep     : job number for dependency (afterok)
  -q queue   : job queue, default=workq
  -M cluster : cluster, default=zeus
  -s imsize  : image size will be imsize x imsize pixels, default 4096
  -p pixscale: image pixel scale, default is 32asec
  -c         : clean image. Default False.
  -t         : test. Don't submit job, just make the batch file
               and then return the submission command
  obsnum     : the obsid to process
```

uses tempaltes:
- `im28s.tmpl`

### obs_flag.sh
Perform flagging on a measurement set.
This consists of running `aoflagger` on the dataset.

Usage:
```
obs_flag.sh [-d dep] [-q queue] [-M cluster] [-t] obsnum
  -d dep      : job number for dependency (afterok)
  -q queue    : job queue, default=workq
  -M cluster : cluster, default=zeus
  -t          : test. Don't submit job, just make the batch file
                and then return the submission command
  obsnum      : the obsid to process
```

uses tempaltes:
- `flag.tmpl`

No job is submitted if the flagging file doesn't exist so this script is safe to include always.

### obs_flag_tiles.sh
Flags a single observation using the corresponding flag file.
The flag file should contain a list of integers being the tile numbers (all on one line, space separated).
This does *not* run `aoflagger`.


usage: 
```
obs_flag_tiles.sh [-d dep] [-q queue] [-M cluster] [-f flagfile] [-t] obsnum
  -d dep      : job number for dependency (afterok)
  -q queue    : job queue, default=workq
  -M cluster : cluster, default=zeus
  -f flagfile : file to use for flagging
                default is processing/<obsnum>_tiles_to_flag.txt
  -t          : test. Don't submit job, just make the batch file
                and then return the submission command
  obsnum      : the obsid to process
```

uses templates:
- `flag_tiles.tmpl`