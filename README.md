# Aim #
This is a simple workflow that aims to help create a BugBase-compatible biom file from within [QIIME 2](https://qiime2.org/). The BugBase pre-print can be found [here](https://www.biorxiv.org/content/10.1101/133462v1). For additional details see its [documentation page](https://bugbase.cs.umn.edu/documentation.html). As BugBase utilizes PICRUSt for its functional category predictions, the output table here should therefore also be PICRUSt-compatiable. Note that there is a [PICRUSt2](https://github.com/picrust/picrust2/wiki) which users should consider if they are not looking to use BugBase. I was/am not involved with BugBase or PICRUSt in any capacity, this is just a crude workaround that I found useful and thought others might benefit from.

Here I outline my approach for use with 16S data, not shotgun metagenomic data. I've tested this with QIIME 2 version 2021.8, but this should work with any older version as well. First activate your QIIME 2 environment:

```
conda activate qiime2-2021.8
```

Since BugBase requires as its input, an OTU table picked against the Greengenes database, I chose to cluster the DADA2-denoised reads from the Moving Pictures tutorial using `vsearch`.

You can use dereplicated sequences without denoising but I personally believe it’s better to utilise denoised reads even if OTU picking methods will ultimately be used.

First let's grab the required files:  
We'll first need the [feature table](https://docs.qiime2.org/2018.8/data/tutorials/moving-pictures/table-dada2.qza) and [representative sequences](https://docs.qiime2.org/2021.8/data/tutorials/moving-pictures/rep-seqs-dada2.qza) which are the output of DADA2 from the [Moving Pictures tutorial]( https://docs.qiime2.org/2021.8/tutorials/moving-pictures/#option-1-dada2).  

You can download these manually from the links above or simply with `wget`.  
The represenative sequences:  

```
wget https://docs.qiime2.org/2021.8/data/tutorials/moving-pictures/rep-seqs-dada2.qza 
```

The DADA2 processed feature-table:  

 ```
 https://docs.qiime2.org/2021.8/data/tutorials/moving-pictures/table-dada2.qza
 ```
 
 We also need the sample metadata file:  
 ```
 wget -O "metadata.tsv" https://data.qiime2.org/2021.8/tutorials/moving-pictures/sample_metadata.tsv
 ```

Finally, we need the Greengenes reference database, which we can download from the [resource page](https://docs.qiime2.org/2021.8/data-resources/#greengenes-16s-rrna). I downloaded the latest 13_8 version here:  

```
wget ftp://greengenes.microbio.me/greengenes_release/gg_13_5/gg_13_8_otus.tar.gz
```

You can extract the entire content of this tarball but since we only need a couple of items from it, I'm going to just extract the 2 files we need.  
The first item is the unaligned representative sequences. Here I am using 97% clustered OTUs for demonstration purposes, but you can use whatever % you want, most likely 99%.  

```
tar -xzvf gg_13_8_otus.tar.gz gg_13_8_otus/rep_set/97_otus.fasta
```

Next we'll also extract the corresponding taxonomies. Note, make sure your taxonomy % matches your OTU rep-set %.  
```
tar -xzvf gg_13_8_otus.tar.gz gg_13_8_otus/taxonomy/97_otu_taxonomy.txt
```

We’re going to need to import the 97% OTU fasta file (from the rep-set folder) into QIIME 2.

```
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path gg_13_8_otus/rep_set/97_otus.fasta \
  --output-path gg_97_otus.qza
```
Now I use `vsearch` to cluster the sequences at 97% similarity threshold using a closed-reference approach. You can use whatever % identity you want here.

```
qiime vsearch cluster-features-closed-reference \
  --i-sequences rep-seqs-dada2.qza \
  --i-table table-dada2.qza \
  --i-reference-sequences gg_97_otus.qza \
  --p-perc-identity 0.97 \
  --o-clustered-table table-cr-97.qza \
  --o-clustered-sequences rep-seqs-cr-97.qza \
  --o-unmatched-sequences unmatched-seqs \
  --verbose
```


BugBase requires that the biom table be in version 1.0 JSON format, and have taxonomic annotations instead of OTU IDs, so we need to do some adjustments first.

To do this we’ll need the underlying biom table within the `table-cr-97.qza` artifact.

```
qiime tools export \
  --input-path table-cr-97.qza \
  --output-path $PWD
```

This saves the exported biom table and calls it `feature-table.biom`.

<!--**When I try to validate the biom table here:
`biom validate-table -i feature-table.biom`
I get:
"Unknown table type, however that is likely okay.
The input file is not a valid BIOM-formatted file." 
#Any idea why this is? If I convert it to json format this goes away. Not really an issue, just seemed odd to me**-->

Right now, our biom table has OTU ID# inherited from our reference database and not taxonomic annotations. So we’ll go ahead and add those taxonomies in. For this we need the `97_otu_taxonomy.txt` file we extracted earlier.

In order to add taxonomy to our biom file first we need to add a new header to the `97_otu_taxonomy.txt` file. We need to add `#OTUID` and `taxonomy` to our first line.

```
echo -e "#OTUID\ttaxonomy" | cat - gg_13_8_otus/taxonomy/97_otu_taxonomy.txt > 97_otu_taxonomy.txt
```
Now we're ready to add our taxonomy.
```
biom add-metadata -i feature-table.biom -o feature-table-tax.biom --observation-metadata-fp 97_otu_taxonomy.txt --sc-separated taxonomy
```
Finally, to convert our biom file to an older version (V1.0 JSON). I found a crude but simple solution to this which is to just convert our biom file to a .txt file then reconvert back using an older biom version. There is likely a more elegant way of doing this, but it works.  

First, convert this to a .txt file:
```
biom convert --table-type="OTU table" -i feature-table-tax.biom -o feature-table-tax.txt --to-tsv --header-key taxonomy
```
Then re-convert back to the older biom version we need.  

```
biom convert -i feature-table-tax.txt -o feature-table-tax-biom1.biom --table-type="OTU table" --to-json --process-obs-metadata taxonomy
```

This biom table is now compatible with BugBase. I validated this by uploading it on the [web-based version of BugBase](https://bugbase.cs.umn.edu/upload.html). Optionally you can upload a metadata file though a couple of minor notes on this. You will have to manually rename the first column to `#SampleID` as per BugBase’s requirement, and also delete the second row which describes the column categories (if you have these in your QIIME 2 metadata, as is the case with the Moving Pictures tutorial metadata). You'll also want to save this file as a `.txt` file.  
The metadata file for this tutorial already has `#SampleID` as the first column header, but we do still need to delete the second row and convert to a `.txt` extension which BugBase is oddly picky about.  

This command will accomplish both these requirements:
```
sed '2d' metadata.tsv > metadata.txt
```
You can now run this through BugBase.
The image below was obtained using the web version of BugBase with all default settings and groups set to BodySite. It shows the predicted phenotypic trait (Aerobic) of the different body sites.

![](Aerobic.jpg)
