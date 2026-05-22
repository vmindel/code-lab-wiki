# How to download the sequencing data after the NovaSeq run

## 1. Connect to Weizmann Network

If doing this from home first connect to the Weizmann Network via the VPN

## 2. Locate your data

Open your browser and navigate to:

```
https://stefan.weizmann.ac.il/raw_novx_incpm/
```
You will see a lot of links to the previous run.
Locate the one with the right date and side (A|B) you ran on.
This is an example of run that we did on December 12 2024, side B:

```
https://stefan.weizmann.ac.il/raw_novx_incpm/20241212_LH00211_0204_B225MJJLT1/
```

## 3. Donwload the data to WEXAC

Connect to the WEXAC server with yiur favorite way and move to the RAWSEQ folder

```bash
cd /home/labs/barkailab/LAB/data/RAWSEQ
```
After you inside just run the wget with the link you previously indetified

```bash
wget -l 7 -nH --cut-dirs=1 -r --reject='index.html*' --no-parent --no-check-certificate https://stefan.weizmann.ac.il/raw_novx_incpm/20241212_LH00211_0204_B225MJJLT1/
```

When everything is done you will see the folder inside the RAWSEQ:
```
20241212_LH00211_0204_B225MJJLT1
```