# rnaseq-deseq2-studio

An RStudio environment for Seqera Platform Studios that runs a DESeq2 differential expression analysis on the output of [nf-core/rnaseq](https://nf-co.re/rnaseq).

The notebook `analysis/deseq2.Rmd` reads the Salmon gene-level counts that nf-core/rnaseq writes to `star_salmon/`, then:

- plots library sizes per sample,
- shows a PCA of variance-stabilised counts coloured by cell line,
- fits a DESeq2 model with design `~ cell_line`,
- runs the comparisons K562 vs GM12878 and MCF7 vs H1, with fold changes shrunk by `apeglm`,
- draws a volcano plot for each comparison with the top genes labelled by gene name,
- draws a heatmap of the 50 most variable genes,
- writes one results table per comparison to `~/results/`.

It was written for the nf-core/rnaseq `test_full` samples (GM12878, K562, MCF7 and H1, two replicates each), but it works for any run whose sample names follow the pattern `<group>_REP<n>`.

## Repository layout

```
.seqera/studio-config.yaml   Studio definition read by Seqera Platform
.seqera/environment.yaml     Conda packages added to the R-IDE image
analysis/deseq2.Rmd          The analysis notebook
```

`studio-config.yaml` uses the Seqera R-IDE image (`public.cr.seqera.io/platform/data-studio-ride:2026.01.2-0.15`) as the template. When the Studio is added, Seqera builds a new image with Wave that adds the packages in `environment.yaml` (DESeq2, apeglm, tidyverse, pheatmap, ggrepel, rmarkdown). `r-base` is pinned to 4.5.3, the version in the template image, so that the added packages don't replace the R that RStudio runs on.

## Add the Studio in Seqera Platform

You need a workspace with a container repository set under Settings > Studios > Container repository, because the image build pushes there.

1. Go to Studios and select Add Studio.
2. Choose Import from Git repository and enter `https://github.com/robsyme/rnaseq-deseq2-studio`.
3. Pick a revision. A commit SHA gives the same notebook every time the Studio starts.
4. Select a compute environment.
5. Under Mount data, select the data link that holds the nf-core/rnaseq results. Data links can't be named in the Git configuration, so this step is always manual.
6. Select Add and start.

The first start takes longer while Wave builds the image. Once the session is running, the repository is at `/workspace` and the mounted data is under `/workspace/data/<data link name>`.

## Run the analysis

Open `/workspace/analysis/deseq2.Rmd` in RStudio. Run the chunks one at a time, or select Knit to produce an HTML report.

## Point the notebook at your data

By default the notebook searches the mounted data links under `/workspace/data`, a few folders deep, and uses the first folder that contains `star_salmon/`. That folder is the nf-core/rnaseq `--outdir`.

To use a specific folder, set `data_dir` in the first chunk:

```r
data_dir <- "/workspace/data/my-data-link/results/my-run"
```

If no results folder is found, the notebook stops with a message saying so.

The comparisons and significance thresholds are in the same chunk. To compare other groups, edit `contrasts`:

```r
contrasts <- list(
  c(numerator = "K562", denominator = "H1")
)
```

## Inputs and outputs

The notebook reads `star_salmon/salmon.merged.gene.SummarizedExperiment.rds`, which holds the Salmon estimated counts, average transcript lengths and gene names. If that file is missing it reads `salmon.merged.gene_counts.tsv` and `salmon.merged.gene_lengths.tsv` from the same folder instead. Counts are rounded to integers, and the transcript lengths are passed to DESeq2 as a normalisation offset, as `DESeqDataSetFromTximport()` would do.

Results tables are written to `~/results/` inside the Studio, one tab-separated file per comparison (for example `deseq2_K562_vs_GM12878.tsv`). They live on the Studio's own disk, so they are kept when the Studio is stopped and restarted, and they are not written to the mounted bucket. Each table has the gene ID, gene name, mean normalised count, shrunken log2 fold change and its standard error, the Wald test p-value, the adjusted p-value and a direction call.
