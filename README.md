# fritz-classification
This tool pulls data from Fritz, uses SNID to classify transients, and uploads classification data to TNS.

## Requirements

* [Anaconda for Python 3.7+](https://www.anaconda.com/products/individual)
* [SNID](https://people.lam.fr/blondin.stephane/software/snid/index.html)
* [superfit](https://github.com/samanthagoldwasser25/superfit)
* [ztfiaenv](https://github.com/MatSmithAstro/ztfiaenv/tree/main/ztfiaenv) (contact [Mat Smith](mailto:m.smith@ipnl.in2p3.fr) for access)
* [pymage](https://github.com/MickaelRigault/pymage) (don't use pip, it will install a different package)

## Installation

Install the required Python packages with `pip install -r requirements.txt`.

Installation of SNID is detailed on the above link. Ensure that its dependencies (PGPLOT) are installed. **You must download the correct templates for this application**. As they are too large to upload here, ask me for them. Installation of superfit is also detailed above, and the correct templates will also need to be installed. Enter in the location the SNID executable and Superfit Python files in the generated info file.

Note about Superfit, I have changed a `run.py` in my installation such that the procces in the script are confined within a `run()` method. To make it work with my scripts, just enclose everything after the imports into a `def run():`.

Generate your unique Fritz API token by going to your [profile](https://fritz.science/profile), finding "Generate New Token for Command-Line Authentication", entering in a name (this is not important), checking all the boxes, and clicking "generate token". Running the program for the first time will generate a file called `info.info`, enter in the API token where appropriate in the file. **Let the code generate the file, the string must be parsed correctly.**

You will also need the ID and API key for the bot that submits information to TNS. Email mrchu@caltech.edu for them and enter them in `info.info` when generated. Enter this into `info.info` also.

You will also need to create a Zooniverse account with edit privileges for the Zwicky Chemical Factory project. The generated info file will have a space to enter in your Zooniverse credentials.

Ensure that after ztfiaenv is installed, you set the appropriate global variables.

## Usage

This script should be run about daily. Run `python master.py` in terminal.

The code will then prompt to enter in a date:

```
Enter in the earliest date you want to check classifications or saves (YYYY-MM-DD) or 'y' for yesterday at midnight:
```

In most cases, you will want to look for newly classified or saved transients from the previous day, so entering `y` is generally acceptable (if more than one day since last running, such as Monday, enter in the date of last use). If for whatever reason you want to look at objects saved earlier, you can enter in this date in the appropriate format.

### -1. Zooniverse Classifications

The code will then ask whether you would like to pull classifications from Zooniverse. If you select yes, all objects with more than 11 classifications retired from the inputted date will be pulled. If those objects have a classification that a majority agree with, it will prompt the user to input the classification. **Enter it as it appears on the image displayed**. If it is a Type II and the rlap score from when SNID ran on it is >5, Superfit will run. If the identified type on Superfit agrees, the user will indicate that this is the case and the object will be formally classified on Fritz. If not, a comment will be submitted with this information.

This is before the rest of the procedure because if you submit classifications to Fritz, you want it to be updated before you submit reports to TNS.

### 0. Data Download

The script will prompt the user to download an ASCII file containing a list of sources saved since the inputted date:

```
Download new list of RCF sources? ([y]/n)
```

Each source is saved with its TNS name, save date, classification (if available), classification date (if available), and redshift (if available). By default, the last 180 days of saved sources will be saved. This is necessary because some sources are saved much earlier and potentially classified recently. This process can take a while, so be patient. This data will be saved as `RCF_sources.ASCII`. If the code fails for whatever reason, you can run from the beginning but skip downloading the ASCII file by entering `n` to the prompt.

### 1. SNID Analysis on Unclassified Sources

The script can select all unclassified sources that have been saved since the inputted date to run classification. The user can proceed to run SNID on unclassified transients, the process of which is the same as with the redshifts. Select the spectrum to use carefully using the following critera:

* **The spectra themselves**: check the plots on Fritz. Generally, SEDM spectra which were taken most recently will have lower SNR, and "constep" spectra will generally also have lower SNR than "robot" spectra. If there are spectra from more powerful telescopes than SEDM (such as LRIS or NOT), use these.
* **Comments**: Check comments for information on the source, sometimes there will be discussion on several potential classifications. If this is the case, SNID will likely not converge with a high `rlap` on any specific template.

The spectral flux data will be downloaded from Fritz as an ASCII file and saved in `/data` (this directory will be generated if it does not exist). SNID will then run on the downloaded spectrum and pull this spectrum from the folder. If SNID converges, the script will open up ten plots, those with the highest `rlap` score. Check the rlap score of the first few, if they are greater than 10 and all converge on the same classification, then consider submitting a classification to Fritz. Check the plots to ensure that the spectrum closely matches that of the templates. The code will also run fitting on the light curve using `sncosmo`. This will pull the photometry data from Fritz and attempt to fit it to an SN Ia light curve. This is helpful if there is a <10 rlap score or noisy spectrum, but many templates that fit an SN Ia. The fit parameters converged on by `sncosmo` and many standard deviations they are from a sample of ~500 Fritz supernovae will also be displayed.

If it is unclear, you can submit the object to Zooniverse. Enter in `n` to not save the classification, and you will be prompted to submit.

The results of SNID will be saved in `/outfiles/<ZTFname>` (this directory also will be generated if it does not exist). These include the `.output` file, which includes the converged templates and their `rlap` scores, plus the redshift and redshift error from the fit of that spectrum. The spectral flux data of the ten templates will also be saved, along with plots of their spectra plotted over the source spectrum. The light curve fit will also be saved as an image.

Again, at the end of the source list the user will be prompted to upload the classifications to Fritz. It will also upload redshifts for the transients if they are not already on Fritz, but will not overwrite if it does already have them. The API responses for each upload will again be returned. In the case that a SNID classification is not an option in Fritz, it will return a failure code, but in this situation just ignore the source and move on. The successfully uploaded classifications and redshifts will be updated in the source ASCII file, so it does not need to be downloaded again to include the updated classifications.

### 2. Light Curve Submission

Users can choose to fit light curves to transients' photometry for unclassified transients saved since the user's inputted date or Type Ia saved/classified since said date. This will pull photometry for each of the transients from Fritz. For those with less than five unique nights of photometry, a plot of the fitted light curve will appear and the user can observe it and determine whether it is meaningful enough to be uploaded to Fritz.

The rest will automatically be uploaded or, if there is already a light curve comment, updated if new photometry has changed the fit.

### 3. Host Association

You can also use ztfiaenv (developed by Mat Smith) to associate transients with potential hosts. This option will run through the objects and use the fitting code to find the most likely fit. In the occasion where this does not work, an error message will appear and will skip the object. This algorithm has relatively high accuracy, but has been known to fail with low-redshift (closer) and/or larger hosts.

### 4. TNS Upload Links

This will post a comment on the Fritz source page with a link to upload to TNS. Anyone with access to the link can choose a spectrum(if more than one) to send to TNS along with the most recent classification.  Currently these comment links are only posted to RCF, RCFDeep, and RCFDeep Partnership groups. If using a different group, either change to public by using RCF_only=False, or change the group numbers. 

### 5. TNS Submission

The script can also submit to TNS any classified transients that have not been previously submitted by ZTF.

The code will run through the sources labeled as "classified" in the ASCII file. Some may not have spectra on Fritz, if this is the case then they were classified externally from publications or by other groups. If this is the case, there is no need to submit and the code will skip it automatically. However, if there is a spectrum, it will first check the comments to ensure the classification did not come from TNS. If it did, enter 0 to proceed onto the next source. **We do not want to send classification reports if our classification came from TNS**. We send reports if we independently classify a transient.

The code will also check TNS to see if the same classification is present. If it is, we tend to not send redundant reports, so the code will skip the object. Occasionally, however, we identify a classification that contradicts that on TNS. If this is the case, usually there will be a comment on the Fritz page with the reason for why we have a different classification. If this is the case, ours usually has a higher quality spectrum so we still want to send our own spectrum. The code will ask whether you want to submit another report, so in this instance you would enter `y`.

It will then prompt the user to enter in a spectrum. It should be relatively obvious which spectrum led to classification, as the uploaded classification tends to follow only several hours after an uploaded spectrum. You can check the time of upload and quality of spectra on the Fritz page. If it is ambiguous, e.g. there is a SEDM and DBSP spectrum around the same time, the higher resolution ones (DBSP, LRIS, etc.) tend to be favored. Among the SEDM spectra, the "auto" ones are of lower quality than the "robot"/"redo", but check the quality anyway before uploading.

Once you select the correct spectrum, the code will generate a TNS report, which will be submitted to TNS through the API. The script might indicate that the source has already been classified and submitted to TNS by a different group. You can check whether the classification is the same, but regardless we want to send our own reports with our own classification analyses to TNS. The code will print out the report as a dictionary, check to make sure that things look correct. If the Fritz classification has no corresponding classification on TNS or there is no redshift available in Fritz, it will return an error in the API response. Do not worry, and move on to the next source. If the submission was successful, it will print a 200 success message in green. The "Uploaded to TNS" comment will then be posted on the source's Fritz page, and the code will move on to the next source.

After completing all in the list, the script will indicate that the submission process is complete.

## Changelog

### 2024-08-09

-Separated TNS upload link comments from host comments

### 2024-06-28

-Made comment links clickable
-Upload to TNS links are now private to RCF groups
-Minor fixes

### 2024-05-10

-Signifcant speed improvement when downloading data file from Fritz

### 2024-04-30 

-Fixed SNID error when spectra contained NaNs
-Now pulls TNS name from source page instead of alerts, reducing errors
-Removed function that adds redshift, as it was previosuly broken

### 2024-03-14

-Added Deveney+LMI to possible instruments

### 2024-02-12

-After uploading to TNS, re-query TNS name in Fritz to match

### 2024-01-09

-Fixed error preventing SPRAT spectra from being read
-Minor improvements

### 2022-06-15

- Minor fixes to author lists for TNS reports.

### 2022-05-20

- Fixes for reducing strain on Fritz by requesting in batches of 50 rather than 500.
- Changes TNS reports such that classifiers are credited first.

### 2022-04-03

- Added in rlap cutoff of 5 for uploading classifications.
- Minor improvements.

### 2022-02-07

- Added in functionality to download sources from other groups.
- Updated for changed Fritz API.
- Changed `get_IAUname` function for objects without alerts.

### 2021-11-20

- Added in host association algorithm.
- Improved classification submission logic, only submit if differing classification than on TNS.
- Corrected SNID templates.
- Added in Zooniverse functionality.

### 2021-08-11

- Added in a check that more than 5 days of photometry exists.
- Calculates peak absolute magnitude using SALT2 parameters and comments on Fritz.

### 2021-08-09

- Fixed negative sigmas.

### 2021-08-04

- Added in light-curve fitting functionality.
- Added in menu for selecting various options rather than running through successively.
- Fixed UTC incompatability with local timekeeping from earlier script - now uses UTC in datetime as it is saved as such on Fritz.

### 2021-08-02

- Moved API and SNID location to `.info` file which will be generated by the code if not present.
- Added in `timeout` of 3 seconds to API function - if timeout occurs the request will be repeated.
- Added in `params` to API function for future use.
- Removed "last date" functionality - rarely used.

## Acknowledgements

A majority of the TNS submission code has been provided for use by Aishwarya Dahiwale from her own [repository](https://github.com/Aishwarya113/TNS-Classification).
