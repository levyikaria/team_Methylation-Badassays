Exploratory2
================
Nivretta

### All output of preprocessing portion will be supressed. Please see preprocessing md file for more.

1.0 PREPROCESSING: INTRODUCTION
===============================

Minfi package separates data from annotation and annotation from array design. The annotation means how methylation loci are associated with gneomic locations and design means how probes on the array are matched with relevant color channels to produce the Meth and Unmeth signals.

Minfi has a couple of different classes to enable flexible development of preprocessing and analysis methods. Quality control and normalization methods can, therefore, be optimally customized to fit unique datasets. The starting point is the raw intensity (RGchannelset) and the end point is the GenomicRatioSet object, which can be exported into a .txt file for downstream analyses.

The following classes and their details:

**RGChannelSet**: a binary format containing the raw green and red channel intensities.

**MethylSet** represents the Meth and Unmeth measurements and are useful for preprocessing routines delivering final measurements in these channels.

**GenomicMethylSet** represents the Meth and Unmeth measurements and are useful for preprocessing routines delivering final measurements in these channels. Also, the methylation loci have been associated with genomic location.

**RatioSet** represents the data as beta values (methylation ratios) or M-values (log ratios of beta values).

**GenomicRatioSet** represents the data as beta values (methylation ratios) or M-values (log ratios of beta values). Also, the methylation loci have been associated with genomic location.

1.1 Dependencies
----------------

Installed the necessary dependencies by:

``` r
source("https://bioconductor.org/biocLite.R")
#biocLite("wateRmelon")
#biocLite('limma')
#biocLite('minfi')
#biocLite('IlluminaHumanMethylation450kmanifest') 
#biocLite('IlluminaHumanMethylation450kanno.ilmn12.hg19')
library(wateRmelon) 
library(IlluminaHumanMethylation450kmanifest)
library(IlluminaHumanMethylation450kanno.ilmn12.hg19)
library(minfi) 
library(dplyr)
library(tibble)
```

1.2 Load data
-------------

We start with reading the .IDAT files and we read in a sample sheet, and then use the sample sheet to load the data into a RGChannelSet.

``` r
#input the right base directory
getwd() #input the right base directory
setwd("../Raw Data/")
basedir <- getwd()
samplesheet <- read.metharray.sheet(basedir, recursive = TRUE) # read in sample sheet .csv file
```

    ## [read.metharray.sheet] Found the following CSV files:

``` r
Eth_rgset <- read.metharray.exp(targets = samplesheet) # read in iDAT files using sample sheet
Eth_rgset2 <- read.metharray.exp(targets = samplesheet, extended = TRUE) #extended Rgset to get bead count info
```

The Eth\_rgset class contains the intensities of the internal control probes as well and as our data were read from a data sheet experiment, the phenotype data is also stored in the Eth\_rgset and can be accessed via the accessor command pData.

``` r
pheno <- pData(Eth_rgset) # phenotype data (from sample sheet)
pheno[,1:6]
getManifest(Eth_rgset) # manifest probe design information of the array.
```

Manifest verifies that we are working with 450K data.

1.3 Create Classes - with no normalization
------------------------------------------

Generating **MethylSet**, which contains only the methylated and unmethylated signals, and **RatioSet**, which stores Beta vlues and/or M values instead of the methylated and unmethylated signals:

``` r
MSet <- preprocessRaw(Eth_rgset) # Processes raw intensity data into Methylated and Unmethylated values (can conver to Beta or 'M' values). Beta values are the estimate of methylation level at each position using the ratio of intensities between methylated and unmethylated probes. Beta values are expected to follow a bimodel distribution of roughly 0s and 1s, corresponding to unmethylated and methylated respectively.
```

preprocessRaw() converts raw intensity data (in the form of IDAT files) into Methylated and Unmethylated values. These values are called Beta or M-values. Beta values are the estimate of methylation level at each position using the ratio of intensities between methylated and unmethylated probes. Beta values are expected to follow a bimodel distribution of roughly 0s and 1s. M-values are the same information just on a log scale, which has been shown to be better for some downstream statistical analyses.

``` r
RSet <- ratioConvert(MSet, what = "both", keepCN = TRUE) #CN is the sum of the methylated and unmethylated signals
```

ratioConvert() consolidates methylated and unmethylated values per CpG site into one value (a ratio of Methylated / Unmethylated).

Get **GenomicRatioSet**:

``` r
GRset <- mapToGenome(RSet) 
```

The function mapToGenome() applied to a RatioSet object adds genomic coordinates to each probe together with some additional annotation information. granges(GRset) can be used to return the probe locations as a genomic ranges.

2.0 Quality Control (QC) - before normalization
===============================================

Before we do normalization we should remove samples that are globally much different than the rest. If a sample has too many failed probes or a very different distribution of methylation measurements, then this is an indication that something went technically wrong with the sample, and therefore carries unreliable data.

In this section we will use some tools from the Minfi package to see if there are any 'bad' samples, obvious outliers, and to remove non-useful probes. What is considered a 'bad' sample? Samples that have been processed poorly due to human error can have global unpredictable affects on DNAm readings.Most of these biases we hope to identify in this section and correct for.

Samples with lots of bad detection p value probes will appear as outliers in the following QC plots. Bad detection p value probes are basically probes that fail to be statistically different (p value &gt; 0.01) from background intensity. Illumina has a set a of negative control probes that give us the 'background intensity value'.

``` r
qc <- getQC(MSet)
plotQC(qc)
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-4-1.png) &gt; The plot estimates samples-specific probe intensities from the MethylSet, comparing methylation probes (M) against unmethylation probes (UM). Intensities for each M and UM are plotted to ensure that they have similar intensities and that the intensities stronger than 10. Bad samples (with lots of bad detection p value probes) will cluster together and lower on the plot. We can see that our samples seem to cluster all together in the 1st quadrant with no obvious outliers.

``` r
densityPlot(MSet, sampGroups = pheno$Ethnicity) #bit hard to see
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-5-1.png) &gt; This is a density plot of the beta values for all samples annotated by ethnicity. We can see that there are no obvious outliers as each sample follows a relatively similar average beta value distribution across all probes.

``` r
densityBeanPlot(MSet, sampGroups = pheno$Ethnicity)
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-6-1.png) &gt; Similar to the density plot above, but plotted as a bean plot, color coded by ethnicity. Again, no obvious outliers.

``` r
controlStripPlot(Eth_rgset, controls="BISULFITE CONVERSION II") #Red is type 1, green is type 2
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-7-1.png) &gt; The 450k array contains several internal control probes that can be used to assess the quality control of different sample preparation steps, in this case, bisulfite conversion. Each control probe is plotted as a strip plot here, showing the consistency of each control probe, suggesting that the changes in our probe sets are not due to errors in preparation.

``` r
mdsPlot(Eth_rgset, sampNames = pheno$Sample_Name, sampGroups = pheno$Ethnicity)
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-8-1.png)

> The Multi-Dimensional Scaling (MDS) plot shows a 2D projection of beta values. The distance between samples show their similarity to each other. It is used as a mean to visually conceptualize the data without making claims to its significance.

> So far all of our QC checks have shown no evidence that there are any obvious outliers or samples that might need to be removed. Here is our first piece of evidence that some samples might be more *interesting*.

> PM58 and PM29 are from different ethnic groups but cluster closer together than they do to the rest of the samples in their group. PM130 and PM158 are also noted.

3.0 Normalization
=================

There are a couple different normalization methods used in DNA methylation analysis. There isn't a consensus on which method is the best. And metrics to evaluate how good normalization methods perform are vague and unclear. So we will try the different available normalization methods:

-   **Noob**: "Implements the noob background subtraction method with dye-bias normalization"
-   **Functional Normalization**: "This function applies the preprocessNoob function as a first step for background substraction, and uses the first two principal components of the control probes to infer the unwanted variation"
-   **Quantile Normalization**: "Implements stratified quantile normalization preprocessing"

Minfi's preprocessing functions all exclusively take an RGset and convert into a downstream object (MSet, GRset).

3.1 Comparing normalization methodology
---------------------------------------

**preprocessNoob** First, we use preprocessNoob function to implement the noob background subtraction method with dye-bias normalization. In this background subtraction method, background noise is estimated from the out-of-band probes and is removed from each sample separately, while the dye-bias normalization utilizes a subset of the control probes to estimate the dye bias (red and green dyes have certain hybridization biases that need to be corrected for).

``` r
MSet.noob <- preprocessNoob(Eth_rgset)
MSet.noob <- MSet.noob[order(featureNames(MSet.noob)), ]
```

**preprocessFunnorm** We use preprocessFunnorm function to implement the functional normalization alogrithm which uses the internal control probes present on the array to infer between-array technical variation.

``` r
GRset_funnorm <- preprocessFunnorm(Eth_rgset, nPCs = 2, sex = NULL, bgCorr = TRUE, dyeCorr = TRUE, verbose = TRUE)
```

    ## [preprocessFunnorm] Background and dye bias correction with noob

    ## [preprocessFunnorm] Mapping to genome

    ## [preprocessFunnorm] Quantile extraction

    ## [preprocessFunnorm] Normalization

``` r
GRset_funnorm <- GRset_funnorm[order(featureNames(GRset_funnorm)), ]
```

**preprocessQuantile**

``` r
GRset_quant <- preprocessQuantile(Eth_rgset, fixOutliers = TRUE, removeBadSamples = FALSE, quantileNormalize = TRUE, stratified = TRUE, mergeManifest = FALSE, sex = NULL)
```

    ## [preprocessQuantile] Mapping to genome.

    ## [preprocessQuantile] Fixing outliers.

    ## [preprocessQuantile] Quantile normalizing.

``` r
GRset_quant <- GRset_quant[order(featureNames(GRset_quant)),]
```

Okay, now we have all of our data into objects that correspond to each normalization method. We will assess how they normalized based on the distribution of Type I and Type II probes after normalization.

``` r
probeTypes <- data.frame(Name = featureNames(MSet),
                         Type = getProbeType(MSet)) #legendpos = "btm" is used to generate an error to remove the legend all together. 
plotBetasByType(MSet[,1], main = "Raw")
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-12-1.png) &gt; preprocessRaw does no normalization so we can see that the type I and type II (infinium I & II) probes have a distinct distribution. As we try different normalization methods we will assess how they perform by looking how close the distributions shift towards overlapping.

``` r
plotBetasByType(MSet.noob[,1], main = "Noob")
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-13-1.png)

``` r
plotBetasByType(getBeta(GRset_funnorm[,1]), probeTypes = probeTypes, main = "funNorm_noob")
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-13-2.png)

``` r
plotBetasByType(getBeta(GRset_quant[,1]), probeTypes = probeTypes, main = "Quantile")
```

![](Exploratory2_files/figure-markdown_github/unnamed-chunk-13-3.png) &gt; A good preprocessing method should make the peaks of type 1 & 2 probe distributions close together, so functional normalization and quantile normalization appears to be better at this task with our dataset. &gt; Hansen et al. 2014 suggests quantile normalization will be better for datasets where we are looking for small differences. FunNorm is more appropriate for when global changes are expected.

3.2 Sex check
-------------

Here we use Minfi's preprocessFunnorm's built-in sex prediction function to verify that there were no sample mixups:

``` r
GRset_funnorm@colData$sex
predicted_sex <- GRset_funnorm@colData$predictedSex
for (i in 1:length(predicted_sex)){
  if (predicted_sex[i] == "F") {predicted_sex[i] = "FEMALE"}
  else {predicted_sex[i] = "MALE"} }
predicted_sex == GRset_funnorm@colData$sex
```

> No sex-specific sample mix ups.

3.3 Probe filtering
-------------------

> Next we check for the presence of SNPs inside the probe body or CpG or at the nucleotide extension. Such probes will be removed.

``` r
# check presence of SNPs inside probe body or single nucleotide extensions
snps <- getSnpInfo(GRset_funnorm)
str(snps@listData$Probe_rs)

GRset_funnorm <- addSnpInfo(GRset_funnorm)
# drop the probes that contain either a SNP at the CpG interrogation or at the single nucleotide extension
GRset_funnorm2 <- dropLociWithSnps(GRset_funnorm, snps=c("SBE","CpG"), maf=0)
GRset_funnorm2
```

``` r
MSet2 <- pfilter(Eth_rgset2, pnthresh = 0.01) #removes all bad detection p value probes and bead count <3.
```

> We used the pfilter() function from wateRmelon package to remove bad detection p value probes and probes with bead count &lt; 3. We started with 'r nrow(MSet)' to 'r nrow(MSet2)' number of probes.

> Now to remove these probes from our genomic ranges object (normalized object)

``` r
GRset_funnorm2B <- getBeta(GRset_funnorm2) #take out beta values
GRset_funnorm2B <- rownames_to_column(as.data.frame(GRset_funnorm2B), 'cpg') #add rownames to a column
gset <- mapToGenome(MSet2) #maps filtered data to genome so 
gsetB <- getBeta(gset) #gsetB will act as filtering index to remove additional probes from the normalized data
gsetB <- rownames_to_column(as.data.frame(gsetB), 'cpg') #dplyr requires rownames to be in colun for joining dataframes
dim(gsetB)
dim(GRset_funnorm2B)

gsetFin2 <- semi_join(GRset_funnorm2B, gsetB, by = 'cpg')
dim(gsetFin2)
#464923 CpGs left
gsetFin2 <- column_to_rownames(gsetFin2, 'cpg') #add cpg row names back (I found this reduces the file size significantly - this is kind of weird)
head(gsetFin2) #yay
```

4.0 Exploratory Analysis
========================

``` r
design <- read.csv("des.txt", sep="\t", header=TRUE)

colnames(gsetFin2) <- c(as.character(design$Samplename))

full <- cbind(design, t(gsetFin2))


#random cpg site, choose number between 1 and 464928
probe_row <- 2000

#get the site name 
probe_name <- colnames(full)[probe_row]

#scatter plot of a random CpG site for all samples, colored by ethnicity
ggplot(full, aes(x = as.factor(ga), y = full[probe_row], colour = Ethnicity)) + 
  geom_jitter(width = 0.5) + 
  facet_wrap(~Sample_Group) + 
  xlab("Gestational Age") + 
  ylab("Beta values") +
  ggtitle(paste("Beta values for CpG site", probe_name)) 


#box plot of a random CpG site for all samples, colored by ethnicity. Red dot is mean.
ggplot(full, aes(x = Ethnicity, y = full[probe_row])) + 
  geom_boxplot(aes(fill=Ethnicity), show.legend = TRUE) + 
  geom_jitter(width = 0.3) + 
  xlab("Ethnicity") + 
  ylab("Beta values") +
  ggtitle(paste("Beta values for CpG site", probe_name)) +
  stat_summary(fun.y = mean, geom="point", colour="darkred", size= 3)

#box plot of a random CpG site
ggplot(full, aes(x = Ethnicity, y = full[probe_row])) + 
  geom_boxplot(aes(fill=Ethnicity), show.legend = FALSE) + 
  geom_jitter(width = 0.3) + 
  facet_wrap(~sex) +
  xlab("Ethnicity") + 
  ylab("Beta values") +
  ggtitle(paste("Beta values for CpG site", probe_name)) +
  stat_summary(fun.y = mean, geom="point", colour="darkred", size= 3)
```