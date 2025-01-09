# 16SrRNA- Concepts and Data Processing Steps 

## Create Manifest file 
````
echo "sample-id,absolute-filepath,direction" > manifest.txt
for file in /scratch/user/zahra/Spring_2024/16S_3rdrun/trimmed_16S2024/*_out.1.fastq.gz; do
  base=$(basename $file _out.1.fastq.gz)
  echo "$base,$file,forward" >> manifest.txt
  echo "$base,${file/_out.1.fastq.gz/_out.2.fastq.gz},reverse" >> manifest.txt
done
````

## 1. **Demultiplexing** (Reverse + Forward reads)
Raw fastq files are demultiplexed to assign seqeunces to their respective samples.<br>
```{code-cell}
qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' --input-path Spring_manifest.txt  --output-path Spring2023_16S_Pairedend_demux.qza --input-format PairedEndFastqManifestPhred33
```
**Visualization**
```{code_cell}
qiime demux summarize --i-data Spring2023_16S_Pairedend_demux.qza  --o-visualization Spring2023_16S_Pairedend_demux.qzv
```


## 2. **Denoising Step**
Removal of low qualtiy bases after analyzing the QC scores from demultiplexed output.  
Data filtering for quality trimming i;e adapters/primers removal following DADA2 pipeline - in case it was not done during pre-processing steps[optional].
DADA2 Script

```{code_cell} #!/bin/bash
#SBATCH --export=NONE               # do not export current env to the job
#SBATCH --job-name=dada2           # job name
#SBATCH --time=0-04:00:00           # max job run time dd-hh:mm:ss
#SBATCH --ntasks-per-node=1         # tasks (commands) per compute node
#SBATCH --cpus-per-task=48          # CPUs (threads) per command
#SBATCH --mem=360G                  # total memory per node
#SBATCH --output=stdout.%j          # save stdout to file
#SBATCH --error=stderr.%j           # save stderr to file

module purge
module load Anaconda3/2023.07-2
source activate /sw/hprc/sw/Anaconda3/2023.07-2/envs/qiime2-2024.2-amplicon

qiime dada2 denoise-paired --p-n-threads 48 --i-demultiplexed-seqs newtrimmed_test_paired-end-demux.qza --o-table newtrimmed_test_table --o-representative-sequences newtrimmed_test_rep-seqs --p-trim-left-f 0 --p-trim-left-r 0 --p-trunc-len-f 270 --p-trunc-len-r 220 --o-denoising-stats newtrimmed_test_denoising_stats.qza

```
**Visualization**
```{code_cell}
qiime feature-table summarize \
  --i-table newtrimmed_test_table.qza \
  --o-visualization newtrimmed_test_table.qzv
qiime feature-table tabulate-seqs \
  --i-data newtrimmed_test_rep-seqs.qza \
  --o-visualization newtrimmed_test_rep-seqs.qzv
qiime metadata tabulate \
  --m-input-file newtrimmed_test_denoising_stats.qza \
  --o-visualization newtrimmed_test_denoising_stats.qzv
```
## 3. **Filtering** 
The nubmer of featuers (ASVs) initially obtained from 16S Fall-2023 dataset were 21308 with a total frequency of ~1.8 million reads. 2-3 samples of each dataset had low reads frequency i;e less than 10K.<br>

During another round of filtering, non-representative data was removed i;e in 16S sequences, singletons and sequences from cholorplast and mitochondira were removed. 
The second filtered data contained total number of features 9235 (ASVs) with a total frequency of ~ 1.7 million.

## 4. **Data Nomalization** 
Rarefaction was performed on 16S rRNA datasets to normalize the sequencing depth across diffrent samples.

However, on fungal ITS datasets, data normalization resulted in significant amount of data loss. To keep all the samples, ITS dataset was processed without normalization.

## 5. **Taxonomy Assignment** 
For 16S rRNA sequences, SILVA database was used for genus to genus rank taxonomy assigment. 

While, for ITS, UNITE database was emploed for genus to Species level taxonomic classification. 

A consus taxonomy was obtained for each ASV which were aggregated into abundances and high level taxa , and plotted relative abundace graphs and top microbial groups present in each sample. 

## 5. Diversity Analysis 
**1. Alpha Diveristy**
Kruskal_Wallis tets was used to compare the alpha diversity indices (Shanon index) across multiple groups within a dataset to identifyt the differences in diversity within the groups. 
