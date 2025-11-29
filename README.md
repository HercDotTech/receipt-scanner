# Receipt Scanner

This is a simple script to automate receipt scanning: it will connect to the scanner, split the image into multiple
receipts if more than one is scanned at the time, then use AI to generate a CSV. Please see below for dependencies and
usage.

## Dependencies

### Things to have
- A scanner that outputs an image file when scanned
- A machine running MacOS

### Things to install
In order for this script to work, you will need to install the following dependencies. Some of the scripts might have to be added to your PATH variable.

| Dependency    | Required | Link                                                                     | Add to path?           |
|---------------|----------|--------------------------------------------------------------------------|------------------------|
| Image Magick  | ✅        | https://imagemagick.org/script/download.php                              | ❌ (Added by installer) |
| Tesseract OCR | ✅        | https://tesseract-ocr.github.io/tessdoc/Installation.html                | ❌ (Added by installer) |
| Fabric        | ✅        | https://github.com/danielmiessler/Fabric?tab=readme-ov-file#installation | ❌ (Added by installer) |
| MultiCrop2    | ✅        | http://www.fmwconcepts.com/imagemagick/multicrop2/index.php              | ✅                      |

### Things to configure
#### Devices
You will need to make sure the machine can access the scanner. Obvious, I know, but it's a good idea to check.
#### Fabric
Fabric requires configuration to work properly. Run `fabric --setup` to start the setup process. This will allow you to configure multiple vendors and models.

A default vendor and model needs to be set and the default patterns, including the `export_data_as_csv` pattern, need to be installed from the setup screen.

## Usage

The script can be run from the command line with the following syntax:
`./scan_receipts '<OUTPUT_DIRECTORY>' '<BG_COLOR_HEX>'`

The background color is the color of the scanner's backplate. This should be a hex value, e.g. `#b98e6e` for a sort of beige. The reason for this parameter is that, generally speaking, receipts and bills have a white background and the scanner's backplate is also white. So what I have done is glue a beige cardboard to it so that it provides a lot of contrast to the receipts. I then did an empty scan using the scanner's software and took the average value of the resulting colour image and converted that to a HEX value. This value is then passed to the script so that it can quickly remove that background and convert it to black.

## Algorithm
The algorithm is based on the following steps:

1. If the output CSV file `<OUTPUT_DIRECTORY>/data.csv` does ***NOT*** exist:
   1. Create the file
   2. Output the header row `merchant,date,summary,total`
2. Prompt the user for the scan loop (enter to continue, `s` to stop)
3. While the user has not stopped the loop:
   1. Create a temporary directory `<OUTPUT_DIRECTORY>/tmp`
   2. Scan the receipt
   3. Write the file to `<OUTPUT_DIRECTORY>/tmp/scan.jpg`
   4. Shave 20 pixels off the edges of the image (helps with some scanner artifacts)
   5. Convert the image background to black to help contrast between the receipts and the background of the image
   6. Add a 50-pixel black border around the image (helps to identify A4 documents in scans)
   7. Convert the image to binary
   8. Split the image into multiple sections using `MultiCrop2`, discard any sections that have an area smaller than 200 pixels
   9. Write the split sections to `<OUTPUT_DIRECTORY>/tmp/split-<index>.jpg`
   10. Remove the original scanned image (helps loop only over the split sections)
   11. For each section:
       1. Run Tesseract OCR on the image to extract the text
       2. Write the extracted text to `<OUTPUT_DIRECTORY>/tmp/split-<index>.txt`
       3. Prepend the AI prompt text to `<OUTPUT_DIRECTORY>/tmp/split-<index>.txt`
       4. Read the contents of `<OUTPUT_DIRECTORY>/tmp/split-<index>.txt` and pass to the AI agent using `Fabric`
       5. Save the last line of the AI response to a local variable
       6. Convert the response to lowercase
       7. Append the line to `<OUTPUT_DIRECTORY>/data.csv`
       8. Parse the line to extract the merchant and date
       9. If the `<OUTPUT_DIRECTORY>/<merchant>` directory does ***NOT*** exist: Create the directory
       10. Copy the section image to `<OUTPUT_DIRECTORY>/<merchant>/<merchant>_<date>.jpg`
   12. Remove the `<OUTPUT_DIRECTORY>/tmp` directory