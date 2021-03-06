# leafcutter

<img src="./logo.png" width="200"> **Annotation-free quantification of RNA splicing.**

*Yang I Li, David A Knowles, Jonathan K Pritchard.*

Leafcutter quantifies RNA splicing variation using short-read RNA-seq data. The core idea is to leverage spliced reads (reads that span an intron) to quantify (differential) intron usage across samples. The advantages of this approach include
* easy detection of novel introns
* modeling of more complex splicing events than exonic PSI
* avoiding the challenge of isoform abundance estimation
* simple, computationally efficient algorithms scaling to 100s or even 1000s of samples

For details please see
http://biorxiv.org/content/early/2016/03/16/044107

If you're annoyed by the RStan warnings try the `newstansyntax` branch. If you want to include covariates in your differential splicing analysis you can checkout the `covariates` branch. 

## Installation

To download the code:
```
git clone https://github.com/davidaknowles/leafcutter
```

To compile the R package to perform differential splicing analysis and make junction plots you can either...

### Option 1. Install from source
```
cd leafcutter
R CMD INSTALL --build .
```
You'll need the following R packages: `Rcpp, rstan, foreach, ggplot2, R.utils, gridExtra, reshape2, Hmisc, dplyr, doMC, optparse`. 

### Option 2. Install using devtools

This has the advantage of installing the required package dependencies for you. 
```
library(devtools)
install_github("davidaknowles/leafcutter/leafcutter")
```

## Usage

For a (hopefully) complete example of the complete pipeline, take a look at
```
example_data/worked_out_example.sh
```
:warning: The test data download is 4Gb!

LeafCutter has two main components: 

1. Python code to 
   - generate intron excision counts from `junc` files (which can be obtained easily from `.bam` files)
   - group introns into clusters
2. `R` code to 
   * perform differential splicing (here, differential intron excision) analysis
   * plot (differentially spliced) clusters
   
### Step 1. Converting `bam`s to `junc`s

I'm skipping Step 0 which would be mapping `fastq` files, e.g. using STAR, to obtain `.bam` files. 

We provide a helper script `scripts/bam2junc.sh` to (you guessed it) convert `bam` files to `junc` files. This step uses the CIGAR strings in the `bam` to quantify the usage of each intron. 

`example_data/worked_out_example.sh` gives you an example of how to do this in batch, assuming your data is in `run/geuvadis/`
```
for bamfile in `ls run/geuvadis/*.bam`
do
    echo Converting $bamfile to $bamfile.junc
    sh ../scripts/bam2junc.sh $bamfile $bamfile.junc
    echo $bamfile.junc >> test_juncfiles.txt
done
```

This step is pretty fast (e.g. a couple of minutes per bam) but if you have samples numbering in the 100s you might want to do this on a cluster. Note that we also make a list of the generated `junc` files in `test_juncfiles.txt`. 

### Step 2. Intron clustering

Next we need to define intron clusters using the `leafcutter_cluster.py` script. For example: 

```
python ../clustering/leafcutter_cluster.py -j test_juncfiles.txt -m 50 -o testYRIvsEU -l 500000
```

This will cluster together the introns fond in the `junc` files listed in `test_juncfiles.txt`, requiring 50 split reads supporting each cluster and allowing introns of up to 500kb. The predix `testYRIvsEU` means the output will be called `testYRIvsEU_perind_numers.counts.gz` (perind meaning these are the *per individual* counts). 

You can quickly check what's in that file with 
```
zcat testYRIvsEU_perind_numers.counts.gz | more 
```
which should look something like this: 
```
RNA.NA06986_CEU.chr1.bam RNA.NA06994_CEU.chr1.bam RNA.NA18486_YRI.chr1.bam RNA.NA06985_CEU.chr1.bam RNA.NA18487_YRI.chr1.bam RNA.NA06989_CEU.chr1.bam RNA.NA06984_CEU.chr1.bam RNA.NA18488_YRI.chr1.bam RNA.NA18489_YRI.chr1.bam RNA.NA18498_YRI.chr1.bam
chr1:17055:17233:clu_1 21 13 18 20 17 12 11 8 15 25
chr1:17055:17606:clu_1 4 11 12 7 2 0 5 2 4 4
chr1:17368:17606:clu_1 127 132 128 55 93 90 68 43 112 137
chr1:668593:668687:clu_2 3 11 1 3 4 4 8 1 5 16
chr1:668593:672093:clu_2 11 16 23 10 3 20 9 6 23 31
```

Each column corresponds to a different sample (original bam file) and each row to an intron, which are identified as chromosome:intron_start:intron_end:cluster_id. 

### Step 3. Differential intron excision analysis

We can now use our nice intron count file to do differential splicing (DS) analysis, but first we need to make a file to specify which samples go in each group. In `worked_out_example.sh` there's some code to generate this file, but you can just make it yourself: it's just a two column tab-separated file where the first column is the sample name (i.e. the filename of the 'bam') and the second is the column (what you call these is arbitrary, but note :bug: the command line interface currently only supports two groups). 

For the worked example this file, named `test_diff_introns.txt` looks like this: 
```
RNA.NA18486_YRI.chr1.bam YRI
RNA.NA18487_YRI.chr1.bam YRI
RNA.NA18488_YRI.chr1.bam YRI
RNA.NA18489_YRI.chr1.bam YRI
RNA.NA18498_YRI.chr1.bam YRI
RNA.NA06984_CEU.chr1.bam CEU
RNA.NA06985_CEU.chr1.bam CEU
RNA.NA06986_CEU.chr1.bam CEU
RNA.NA06989_CEU.chr1.bam CEU
RNA.NA06994_CEU.chr1.bam CEU
```

Having made that file we can run DS (this assumes you have successfully installed the `leafcutter` R package as described under Installation above) 
```
../scripts/leafcutter_ds.R --num_threads 4 ../example_data/testYRIvsEU_perind_numers.counts.gz ../example_data/test_diff_intron.txt
```

Running `../scripts/leafcutter_ds.R -h` will give usage info for this script. 

Two tab-separated text files are output:

1. `leafcutter_ds_cluster_significance.txt`. This shows per cluster `p`-values for there being differential intron excision between the two groups tested. The columns are
 1. cluster: the cluster id
 2. Status: whether this cluster was a) successfully tested b) not tested for some reason (e.g. too many introns) c) there was an error during testing - this should be rare. 
 3. loglr: log likelihood ratio between the null model (no difference between the groups) and alternative (there is a difference) 
 4. df: degrees of freedom, equal to the number of introns in the cluster minus one (assuming two groups)
 5. p: the resulting (unadjusted!) p-value under the asymptotic Chi-squared distribution. We just use `p.adjust( ..., method="fdr")` in R to control FDR based on these. 

2. `leafcutter_ds_effect_sizes.txt`. This shows per intron effect sizes between the groups, with columns:
 1. intron: this has the form chromosome:intron_start:intron_end:cluster_id
 2. log effect size (as fitted by LeafCutter).
 3. Fitted usage proportion in condition 1. 
 4. Fitted usage proportion in condition 2. 
 5. DeltaPSI: the difference in usage proprotion (condition 2 - condition 1). Note that in general this will have the same sign as the log effect size but in some cases the sign may change as a result of larger changes for other introns in the cluster. 

### Step 4. Plotting splice junctions

This will make a pdf with plots of the differentially spliced clusters detected at an FDR of 5%. 
```
../scripts/ds_plots.R -e ../leafcutter/data/gencode19_exons.txt.gz ../example_data/testYRIvsEU_perind_numers.counts.gz ../example_data/test_diff_intron.txt leafcutter_ds_cluster_significance.txt -f 0.05
```

## Splicing QTL

Splicing QTL analysis is a little more involved than differential splicing analysis, but we provide a script `scripts/prepare_phenotype_table.py` intended to make this process a little easier. We assume you will use `FastQTL` (http://fastqtl.sourceforge.net/) for the sQTL mapping itself, but reformatting the output if you want to use another tool (e.g. MatrixEQTL http://www.bios.unc.edu/research/genomic_software/Matrix_eQTL/) should be reasonably straightforward. The script is pretty simple: a) calculate intron excision ratios b) filter out introns used in less than 40% of individuals or with almost no variation c) output these ratios as gzipped txt files along with a user-specified number of PCs. 

You'll need the `sklearn` and `scipy` Python packages installed, e.g. 
```
pip install sklearn
```
for the PCA calculation. 

Usage is e.g.
```
python scripts/prepare_phenotype_table.py example_data/testYRIvsEU_perind.counts.gz -p 10
```
where `-p 10` specifies you want to calculate 10 PCs for FastQTL to use as covariates. 

FastQTL needs `tabix` indices. To generate these you'll need `tabix` and `bgzip` which you may have as part of `samtools`, if not they're now part of `htslib`, see https://github.com/samtools/htslib for installation instructions (alternatively `apt-get install tabix` worked for me in Ubuntu 14.04). With these dependencies installed you can run the script created by and pointed to by the output of `prepare_phenotype_table.py`, e.g. `example_data/testYRIvsEU_perind.counts.gz_prepare.sh`. 

We assume you'll run FastQTL separately for each chromosome: the files you'll need will have names like `testYRIvsEU_perind.counts.gz.qqnorm_chr21.gz`. The PC file will be e.g. testYRIvsEU_perind.counts.gz.PCs. 

