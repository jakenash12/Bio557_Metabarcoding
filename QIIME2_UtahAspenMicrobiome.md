### Download Utah aspen microbiome dataset
```
curl -L "https://www.dropbox.com/scl/fi/34gh6e7yip8qs4lmhw6y9/UtahAspenMicrobiome.tar.gz?rlkey=1ozl9c6cdzhlkbovs715zlrgr&dl=0" > UtahAspenMicrobiome.tar.gz && tar -xvzf UtahAspenMicrobiome.tar.gz
```

### Prepares a manifest file in tsv (tab separated value) format that is used to import data into QIIME2
Don't worry about understanding this too much - it is an arcane bash script
```
ls UtahAspenMicrobiome | grep "R1_001.fastq.gz" | sed 's/_R1_001.fastq.gz//g' > filelist
printf "%s\t%s\t%s\n" "sample-id" "forward-absolute-filepath" "reverse-absolute-filepath" > QIIMEManifest.tsv
Dir=$(pwd)
for i in $(cat filelist)
do
Sample=$(echo "$i" | awk -F '_' '{print $1}')
printf "%s\t%s\t%s\n" "${Sample}" "${Dir}/UtahAspenMicrobiome/${i}_R1_001.fastq.gz" "${Dir}/UtahAspenMicrobiome/${i}_R2_001.fastq.gz" >> ${Dir}/QIIMEManifest.tsv
done
```

### Imports data into QIIME2, then creates summary visualization 
QIIME2 qzv visualizations need to be downloaded to your personal computer to view at https://view.qiime2.org/. I have provided all of them in the github repo for convenience, but example code to download directly from your VM is below
```
#make sure to activate the QIIME2 environment if its not already activated
conda activate qiime2-2024.2

qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path QIIMEManifest.tsv \
  --output-path AspenMicro.qza \
  --input-format PairedEndFastqManifestPhred33V2

qiime demux summarize \
  --i-data AspenMicro.qza \
  --o-visualization AspenMicro.qzv

#optional: download qzv (or get from github)
#Log out of VM by typing "exit" and then type following code into shell
scp NETID@HOSTNAME:./AspenMicro.qzv /local/path/
```

### Cutadapt to trim primers and adapter sequences
Cutadapt can trim primers/adapters from both the 5' (i.e beginning) and 3' (i.e. end) of the forward and reverse reads. Primers/adapters may occur on the 3' end in the event of read through (i.e. the 300 bp read extends to the end of the DNA fragment).
```
qiime cutadapt trim-paired \
	--i-demultiplexed-sequences AspenMicro.qza \
	--p-adapter-f ATTAGAWACCCBDGTAGTCC \
	--p-adapter-f AGATCGGAAGAGCACACGTCTGAACTCCAGTCAC \
	--p-front-f GTGCCAGCMGCCGCGGTAA \
	--p-front-f GTGCCAGCMGCWGCGGTAA \
	--p-front-f GTGCCAGCMGCCGCGGTCA \
	--p-front-f GTGKCAGCMGCCGCGGTAA \
	--p-front-f GCCTCCCTCGCGCCATCAGAGATGTGTATAAGAGACAG \
	--p-adapter-r TTACCGCGGCKGCTGMCAC \
	--p-adapter-r TGACCGCGGCKGCTGGCAC \
	--p-adapter-r TTACCGCWGCKGCTGGCAC \
	--p-adapter-r TTACCGCGGCKGCTGGCAC \
	--p-adapter-r CTGTCTCTTATACACATCTCTGATGGCGCGAGGGAGGC \
	--p-front-r GGACTACHVGGGTWTCTAAT \
	--p-front-r GTGACTGGAGTTCAGACGTGTGCTCTTCCGATCT \
	--p-error-rate 0.15 \
	--output-dir AspenMicro_cutadapt

qiime demux summarize \
  --i-data ./AspenMicro_cutadapt/trimmed_sequences.qza \
  --o-visualization AspenMicro_cutadapt.qzv
```

### Dada2 for Actual Sequence Variant (ASV) calling (and quality control, read merging, chimera detection)
```
qiime dada2 denoise-paired \
	--i-demultiplexed-seqs ~/AspenMicro_cutadapt/trimmed_sequences.qza \
	--p-trunc-len-f 220 \
	--p-trunc-len-r 120 \
	--p-trim-left-f 0 \
	--p-trim-left-r 0 \
	--output-dir ~/AspenMicro_dada2

qiime metadata tabulate \
	--m-input-file ~/AspenMicro_dada2/denoising_stats.qza \
	--o-visualization ~/denoising_stats.qzv

qiime feature-table summarize \
	--i-table ~/AspenMicro_dada2/table.qza \
	--o-visualization ~/table_summary.qzv
```

### Downloads SILVA taxonomic classifier and uses it to classify Dada2 ASVs
This code is too computationally intensive to run on our virtual machines, so download the taxonomic classification file AspenMicro_dada2_taxonomy.qza from github instead
```
wget https://data.qiime2.org/2023.9/common/silva-138-99-515-806-nb-classifier.qza

qiime feature-classifier classify-sklearn \
	--i-reads ~/AspenMicro_dada2/representative_sequences.qza \
	--i-classifier ~/silva-138-99-515-806-nb-classifier.qza \
	--o-classification ~/AspenMicro_dada2_taxonomy.qza

#download taxonomic classifications here instead of doing it on your VM
wget https://github.com/jakenash12/Bio557_Metabarcoding/raw/main/AspenMicro_dada2_taxonomy.qza
```

### Download metadata file then 
```
wget https://github.com/jakenash12/Bio557_Metabarcoding/raw/main/UtahAspenMicrobiomeMetadata.tsv

qiime diversity core-metrics \
	--i-table ~/AspenMicro_dada2/table.qza \
	--p-sampling-depth 2016 \
	--m-metadata-file ~/UtahAspenMicrobiomeMetadata.tsv \
	--output-dir ~/CoreMetrics
```

### Tests for differences in ASV richness (i.e. diversity, # of ASVs) and beta diversity
```
qiime diversity alpha-group-significance \
  --i-alpha-diversity ~/CoreMetrics/observed_features_vector.qza \
  --m-metadata-file ~/UtahAspenMicrobiomeMetadata.tsv \
  --o-visualization ~/CoreMetrics/richness_significance.qzv

qiime diversity beta-group-significance \
  --i-distance-matrix ~/CoreMetrics/bray_curtis_distance_matrix.qza \
  --m-metadata-file ~/UtahAspenMicrobiomeMetadata.tsv \
  --m-metadata-column SiteType \
  --o-visualization ~/CoreMetrics/bray-curtis-SiteType-significance.qzv \
  --p-pairwise
```

### Generates a taxonomy barplot of data
```
qiime taxa barplot \
  --i-table ~/AspenMicro_dada2/table.qza \
  --i-taxonomy ~/AspenMicro_dada2_taxonomy.qza \
  --m-metadata-file ~/UtahAspenMicrobiomeMetadata.tsv \
  --o-visualization ~/taxa-bar-plots.qzv
```

### Tests for differential abundance of Genus between Site types
```
qiime taxa collapse \
  --i-table ~/AspenMicro_dada2/table.qza \
  --i-taxonomy ~/AspenMicro_dada2_taxonomy.qza \
  --p-level 6 \
  --o-collapsed-table ~/AspenMicro_dada2/table_genus.qza

qiime composition ancombc \
  --i-table ~/AspenMicro_dada2/table_genus.qza  \
  --m-metadata-file ~/UtahAspenMicrobiomeMetadata.tsv \
  --p-formula 'SiteType' \
  --o-differentials ~/ancombc-SiteType_genus.qza

qiime composition da-barplot \
  --i-data ~/ancombc-SiteType_genus.qza \
  --p-significance-threshold 0.001 \
  --o-visualization ~/ancombc-SiteType_genus.qzv
```
scp jan58@vcm-39234.vm.duke.edu:~/ancombc-SiteType_genus.qzv "C:\Users\akeja\OneDrive - Duke University\Documents\Github\Bio557_Metabarcoding"