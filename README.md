# Museomics
Code for analysing 16S data from frozen museum specimens

# Import fastq files

#forward reads only 

qiime tools import \
  --type 'SampleData[SequencesWithQuality]' \
  --input-path AGRF_CAGRF220711473_KHP2R_custom/demultiplexed \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path demux-single-end.qza
  
qiime demux summarize \
  --i-data demux-single-end.qza \
  --o-visualization demux-single-end.qzv

qiime tools view demux-single-end.qzv

qiime quality-filter q-score --i-demux demux-single-end.qza --o-filtered-sequences demux-filtered.qza --o-filter-stats demux-filter-stats.qza

qiime dada2 denoise-single \
--i-demultiplexed-seqs demux-filtered.qza \
--p-trim-left 10 \
--p-trunc-len 245 \
--o-representative-sequences rep-seqs.qza \
--o-table table.qza \
--o-denoising-stats stats-dada2.qza





#paired end 

qiime tools import \
  --type 'SampleData[SequencesWithQuality]' \
  --input-path test \
  --output-path paired-end-demux.qza \
  --input-format PairedEndFastqManifestPhred33V2



qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path test \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path demux-paired-end.qza

qiime tools import \
--type 'SampleData[PairedEndSequencesWithQuality]' \
--input-path test\
--input-format CasavaOneEightSingleLanePerSampleDirFmt \
--output-path demux-paired-end.qza



#generate quality plot

qiime demux summarize \
  --i-data demux-paired-end.qza \
  --o-visualization demux-paired-end.qzv
  
qiime tools view demux-paired-end.qzv


#Quality control 
qiime quality-filter q-score --i-demux demux-paired-end.qza --o-filtered-sequences demux-filtered.qza --o-filter-stats demux-filter-stats.qza


# Denoising with DADA2

qiime dada2 denoise-paired \
--i-demultiplexed-seqs demux-paired-end.qza \
--p-trim-left-f 5 \
--p-trim-left-r 5 \
--p-trunc-len-f 240 \
--p-trunc-len-r 230 \
--p-n-threads 2 \
--o-table table.qza \
--o-representative-sequences rep-seqs.qza \
--o-denoising-stats stats-dada2.qza \
--verbose

  
qiime metadata tabulate \
--m-input-file stats-dada2.qza \
--o-visualization stats-dada2.qzv


qiime feature-table summarize \
  --i-table table.qza \
  --o-visualization table.qzv
  
qiime tools view stats-dada2.qzv
  
qiime tools view table.qzv

qiime feature-table tabulate-seqs \
  --i-data rep-seqs.qza \
  --o-visualization rep-seqs.qzv

# Taxonomy

qiime feature-classifier classify-sklearn \
--i-classifier silva-138-99-nb-classifier.qza \
--i-reads rep-seqs.qza \
--o-classification taxonomy.qza
  
qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization taxonomy.qzv

qiime taxa barplot --i-table table.qza --i-taxonomy taxonomy.qza --o-visualization taxa-bar-plots.qzv
  
qiime tools view taxa-bar-plots.qzv

# generates phylogenetic tree and distances 

qiime phylogeny align-to-tree-mafft-fasttree --i-sequences rep-seqs.qza --o-alignment aligned-rep-seqs.qza --o-masked-alignment masked-aligned-rep-seqs.qza --o-tree unrooted-tree.qza --o-rooted-tree rooted-tree.qza


#Rarefaction

qiime diversity alpha-rarefaction --i-table table.qza --i-phylogeny rooted-tree.qza --p-max-depth 5000 --o-visualization alpha-rarefaction.qzv

#how do I decide what sampling depth to use?? Use table.qzv to gauge.

qiime tools view alpha-rarefaction.qzv

#diversity analysis 

qiime diversity core-metrics-phylogenetic --i-phylogeny rooted-tree.qza --i-table table.qza  --output-dir core-metrics-results
 
cd core-metrics-results
 
qiime diversity alpha-group-significance --i-alpha-diversity observed_otus_vector.qza --m-metadata-file ../metadata.txt --o-visualization observed_otus_significance.qzv


# Export tables

qiime tools export \
  --input-path table.qza \
  --output-path exported-table
  
biom convert -i exported-table/feature-table.biom -o exported-table/table.tsv --to-tsv

#this is the final output table useful for PRIMER or R.
