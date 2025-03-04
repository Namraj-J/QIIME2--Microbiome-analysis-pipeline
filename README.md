# QIIME2--Microbiome-analysis-pipeline
"Pipeline for microbiome analysis using QIIME 2"
## 1. Directory Management
# Change to the home directory to ensure correct file handling.
cd ~

## 2. File Handling
# Verify the presence of the manifest file and inspect its contents.
cat manifestt.csv

## 3. Activate QIIME 2 Environment
# Activate the QIIME 2 environment to ensure you're using the appropriate software version.
conda activate qiime2-amplicon-2024.10

## 4. QIIME Data Import
# Import paired-end sequences into QIIME 2 using a manifest file.
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path manifest.tsv \
  --output-path paired-end-demux.qza \
  --input-format PairedEndFastqManifestPhred33V2

## 5. Primer Removal
# Trim primers from the sequences using `qiime cutadapt`.
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences paired-end-demux.qza \
  --p-front-f TTGTACACACCGCCC \
  --p-front-r CCTTCYGCAGGTTCACCTAC \
  --p-match-adapter-wildcards \
  --p-discard-untrimmed \
  --o-trimmed-sequences trimmed-seqs-18S.qza \
  --verbose

## 6. Visualize Imported Data
# Summarize and visualize the imported data to inspect quality.
qiime demux summarize \
  --i-data trimmed-seqs-18S.qza \
  --o-visualization trimmed-seqs-18S.qzv

## 7. Denoise with DADA2
# Denoise paired-end reads using DADA2.
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs trimmed-seqs-18S.qza \
  --p-trunc-len-f 193 \
  --p-trunc-len-r 158 \
  --o-table table193158.qza \
  --o-representative-sequences rep-seqs193158.qza \
  --o-denoising-stats denoising-stats193158.qza

# View DADA2 denoising stats.
qiime metadata tabulate \
  --m-input-file denoising-stats193158.qza \
  --o-visualization denoising-stats193158.qzv

## 8. Train Classifier Using RESCRIPt
# Train a classifier using the RESCRIPt plugin and the SILVA database.
# First, process, filter, and evaluate the SILVA database.

## 9. Assign Taxonomy
# Assign taxonomy to sequences using the trained classifier.
qiime feature-classifier classify-sklearn \
    --i-classifier silva-138.2-18S-classifier.qza \
    --i-reads rep-seqs193158.qza \
    --o-classification taxonomy.qza

## 10. Filter for Eukaryota
# Filter the feature table and sequences to retain only features assigned to `d__Eukaryota`.

# For Feature Table:
qiime taxa filter-table \
  --i-table table193158.qza \
  --i-taxonomy taxonomy.qza \
  --p-include "d__Eukaryota" \
  --o-filtered-table eukaryota-table.qza

# For Representative Sequences:
qiime taxa filter-seqs \
  --i-sequences rep-seqs193158.qza \
  --i-taxonomy taxonomy.qza \
  --p-include "d__Eukaryota" \
  --o-filtered-sequences eukaryota-rep-seqs.qza

## 11. Filter for Specific Phyla
# Further filter to focus on specific phyla.

# For Feature Table:
qiime taxa filter-table \
  --i-table eukaryota-table.qza \
  --i-taxonomy taxonomy.qza \
  --p-include "p__" \
  --o-filtered-table filtered-phylum-table.qza

# For Representative Sequences:
qiime taxa filter-seqs \
  --i-sequences eukaryota-rep-seqs.qza \
  --i-taxonomy taxonomy.qza \
  --p-include "p__" \
  --o-filtered-sequences filtered-phylum-rep-seqs.qza

## 12. Feature Table Summary
# Summarize the filtered feature table to examine the sequencing depth and feature counts.
qiime feature-table summarize \
  --i-table filtered-phylum-table.qza \
  --o-visualization filtered-phylum-table-summary.qzv

## 13. Generate Taxa Bar Plot
# Create a bar plot to visualize taxa distribution.
qiime taxa barplot \
  --i-table filtered-phylum-table.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization taxa-bar-plots.qzv

## 14. Phylogenetic Tree Construction
# Use the filtered representative sequences for phylogenetic tree generation.
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences filtered-phylum-rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza

## 15. Core Diversity Analysis
# Perform core diversity analysis using the rooted phylogenetic tree and the filtered feature table.
qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table filtered-phylum-table.qza \
  --p-sampling-depth 9500 \
  --m-metadata-file metadata.tsv \
  --output-dir core-metrics-results

# Alpha Diversity Visualization:
qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/faith_pd_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization core-metrics-results/faith-pd-group-significance.qzv

qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/observed_features_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization core-metrics-results/observed-otus-group-significance.qzv

# Additional Alpha Diversity Metrics:
qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/shannon_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization core-metrics-results/shannon-group-significance.qzv

qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/evenness_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization core-metrics-results/evenness-group-significance.qzv

## 16. Beta Diversity Visualization
# Visualize beta diversity using PCoA plots.
qiime emperor plot \
  --i-pcoa core-metrics-results/unweighted_unifrac_pcoa_results.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization core-metrics-results/unweighted-unifrac-emperor.qzv

## 17. Alpha Rarefaction Curve
# Generate an alpha rarefaction curve to evaluate sequencing depth.
qiime diversity alpha-rarefaction \
  --i-table filtered-phylum-table.qza \
  --p-max-depth 9500 \
  --m-metadata-file metadata.tsv \
  --o-visualization alpha-rarefaction9500.qzv

## 18. Adonis Test Results for Environmental Factors
# Perform the Adonis test to evaluate the influence of environmental factors on community composition.
qiime diversity adonis \
  --i-distance-matrix bray_curtis_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --p-formula "Position + pH + EC + Nitrate + Total_N + Total_P + Arsenic + Cadmium + Chromium + Lead + Zinc" \
  --o-visualization adonis_bray_curtis_all_factorsall.qzv
