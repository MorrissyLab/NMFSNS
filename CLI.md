# Command Line Interface for cNMF-SNS

Each step of a workflow can be run as a subcommand within cNMF-SNS. You can see which subcommands are available using:

```bash
cnmfsns --help
```

Easily get help for each subcommand using like this:

```bash
cnmfsns model-odg --help
```

## Part I: Factorizing individual datasets

### 1. Create AnnData object from text files with gene expression and metadata

If expression and annotation data is in text files, this utility can combine them into a .h5ad file for downstream tools. If you have counts data, then use the following command:

```bash
cnmfsns txt-to-h5ad --data_file counts.txt --metadata metadata.txt -o dataset.h5ad
```

If no counts data is available and the dataset is normalized(eg., for non-count based assays), you can also specify `--is_normalized` to prevent cNMF-SNS from performing a TPM
normalization step for the purposes of overdispersed gene selection. If `--is_normalized` is specified, the input matrix is used both fore overdispersed gene selection and for
factorization.

By default, text files are tab-separated, although other characters can be specified using `--data-delimiter` and `--metadata-delimiter`.

By default, Expression data must be indexed as follows.

  - Rows are samples/cells/spots; first column must be sample/cell/spot IDs
  - Columns are genes/features; the first row must be feature names
If your data is in the opposite orientation, specify `--transpose`.

Metadata must be indexed as follows:

  - The first column must be sample/cell/spot IDs
  - Other columns are metadata 'layers' and must be labelled. Values can be numerical, boolean, or categorical types.
      > Note: if any values in a column are not numerical, the entire column will be treated as categorical. This can have implications for annotated heatmaps where numerical data is usually presented as a continuous color scale, not a set of distinct colors. If a column is numerical with missing values, then these should be empty values (not "NA", "NaN", etc.)
  - Missing values are acceptable. For categorical data, these will be plotted in an "Other" category. For numerical data, these will be ignored.

### 2. Check existing h5ad files for minimum requirements for cNMF.

Check h5ad objects for rows or columns which have missing values, negative values, or variance of 0.

cNMF  supports input data that is sparse (i.e. with zeros), but not with missing values. When missing values are present (eg. from concatenation of datasets with partially overlapping features), the default behaviour is to subset the input matrix to shared features/genes only, but it is recommended to either run each dataset separately or use a dense, imputed data matrix. cNMF will warn the user if missing data is present.

```bash
cnmfsns check-h5ad -i file.h5ad -o file_filtered.h5ad
```

### 3. Model gene overdispersion to select genes for factorization.

Deconvolution of a gene expression dataset using cNMF requires a set of overdispersed genes/features which will be used for factorization. GEPs will include all genes after a re-fitting step, but cNMF will only optimize the fit for overdispersed genes, providing the user the opportunity to decide which genes are most informative.

Since cNMF performs variance scaling on the input matrix, it is important to remove genes whose variance could be attributable to noise. cNMF-SNS supports two methods for overdispersed gene selection:

  - `cnmf`: v-score and minimum expression threshold (cNMF method: Kotliar, et al. eLife, 2019). This method is only suitable for count data.
  - `default`: residual standard deviation after modeling mean-variance dependence. (STdeconvolve method: Miller, et al. Nat. Comm. 2022) This method makes fewer assumptions about the input data but requires a visual check since the optimal threshold depends on the data type. Differently than all other methods, this method does not make assumptions about mean/sum of expression across samples, permitting the discovery of rare cell types.

To produce plots to guide selection of overdispersed genes, run the following command:

```bash
cnmfsns model-odg --name example_run --input file_filtered.h5ad
```

This command will create a directory with the name of the run inside the output directory (defaults to current working directory).

### 4. Select overdispersed genes and parameters for factorization

Once you have decided on a method for selecting overdispersed genes and are ready for factorization, you can easily use the default parameters and select values of k as follows:

```bash
cnmfsns set-parameters --name example_run -k 2 -k 3
```

A more complex range of k values can also be set up using the `--k_range` parameter. For example, to perform cNMF using a range of k values from 5 to 60, with a step of 5 (ie.: 5, 10, 15, ... 60), you would specify `--k_range 5 60 5`.

The default behaviour is to select overdispersed genes using an od-score > 1.0. You can also choose from a number of different methods for selecting overdispersed_genes, for example:

  - `cnmfsns set-parameters --name example_run -m cnmf_topn -p 2000`: select top 2000 genes using cNMF's model and minimal mean threshold (Kotliar et al., 2019, eLife)
  - `cnmfsns set-parameters --name example_run -m default_quantile -p 0.8`: Select top 20% of genes when ranked by od-score
  - `cnmfsns set-parameters --name example_run -m genes_file -p path/to/genesfile.txt`: use a custom list of genes

### 5. Perform cNMF factorization

Factorize the input data. While parameters can be provided which allow for custom parallelization, by default cnmfsns uses a single CPU:

```bash
cnmfsns factorize --name example_run
```

For submitting jobs to the SLURM job scheduler, you can download a sample job submission script [here](https://github.com/MorrissyLab/cNMF-SNS/tree/main/scripts/slurm.sh).

After editing the script to ensure it is suitable for your HPC cluster, cNMF-SNS will submit jobs using SLURM's `sbatch` command to parallelize factorization.

```bash
cnmfsns factorize --name example_run --slurm_script /path/to/slurm.sh
```

### 6. Postprocessing

This step will check to ensure that all factorizations completed successfully, and then will create consensus GEPs and usages, updating the `.h5ad` file with the cNMF solution.

```bash
cnmfsns postprocess --name example_run
```

For downstream analyses, the output AnnData object is in `./example_run/example_run.h5ad`.

### 7. Created annotated heatmaps of GEP usages

This step will create annotated heatmaps of GEP usages from cNMF-SNS outputs:

```bash
cnmfsns annotated-heatmap --output_dir ./example_run/ -i ./example_run/example_run.h5ad
```

To provide custom colors for the metadata layers, you can specify a TOML-formatted file with a `metadata_colors` section (see `scripts/example_config.toml`) 

## Part II: Integration of multiple datasets

### 1. Identify datasets for integration

Specify an output directory for your integration.

A [TOML](https://toml.io/en/) configuration file is the most flexible way to configure cNMF-SNS. An example is found in `scripts/example_config.toml`. The minimal data to include in a TOML file is the paths to each of the datasets. Default values for other parameters will be used if not specified. 

```bash
cnmfsns integrate -c config.toml -o output_directory
```

Alternatively, you can can also initialize a cNMF-SNS integration by providing a set of h5ad files from [`cnmfsns postprocess`](#6.-postprocessing) to integrate:

```bash
cnmfsns integrate -i file1.h5ad -i file2.h5ad -i file3.h5ad -o output_directory
```

Once the command has completed, outputs are located in `<output_directory>/integrate/`.

- UpSet plot of OD Genes between datasets
- plot correlation between cohorts
- rank reduction plots to exclude high ranks with highly correlated GEPs
- `node_stats.txt`: number of nodes before and after node and edge filters, per dataset.

### 2. Network-based analyses

To run integrative analyses on multiple datasets, you can run the following command using the same output directory you specified for `cnmfsns integrate`.

```bash
cnmfsns create-network -o output_directory -n name
```

Once the command has completed, outputs are located in `<output_directory>/sns_networks/<name>/`. Plots include:

- GEP network plots:
  - `gep_network.pdf`: GEPs are labelled
  - `gep_network_communities.pdf`: Colored by community
  - `gep_network_datasets.pdf`: Colored by dataset
  - `gep_network_rank.pdf`: Colored by dataset, size inversely proportional to rank (k)
  - `npatients_discrete.pdf`: Node size is proportional to the number of patients with this GEP as its maximum
  - `nsamples_continuous.pdf`: Node size is proportional to the sum of the usage across samples
  - `nsamples_discrete.pdf`: Node size is proportional to the number of samples with this GEP as its maximum
  - `annotated_geps/overrepresentation_network/<dataset>/<annotation_layer>.pdf`: plots overrepresentation of each category on the network for categorical metadata
  - `annotated_geps/correlation_network/<dataset>/<annotation_layer>.pdf`: plots correlation of numerical metadata to GEP usage across samples on the network

- GEP bar plots:
  - `annotated_geps/overrepresentation_bar/<dataset>/<annotation_layer>.pdf`: plots overrepresentation of each category for each GEP for categorical metadata
  - `annotated_geps/correlation_bar/<dataset>/<annotation_layer>.pdf`: plots correlation of numerical metadata to GEP usage across samples for each GEP

- Community network plots:
  - `community_network.pdf`: Each community is represented as a single node located in the centroid of the community in the GEP network plots. Nodes are colored by community, and the edges are proportional to the number of edges in the GEP networks.
  - `annotated_communities/overrepresentation_network/<dataset>/<annotation_layer>.pdf`: plots overrepresentation of each category on the network for categorical metadata
  - `annotated_communities/correlation_network/<dataset>/<annotation_layer>.pdf`: plots correlation of numerical metadata to GEP usage across samples on the network
  - `annotated_communities/patient_network/<annotation_layer>/<n>samplesperpatient.pdf`: plots the community usage for each sample within a patient, averaged and colored by sample category for categorical metadata.

- Community bar plots:
  - `annotated_communities/overrepresentation_bar/<dataset>/<annotation_layer>.pdf`: plots overrepresentation of each category for each community for categorical metadata
  - `annotated_communities/correlation_bar/<dataset>/<annotation_layer>.pdf`: plots correlation of numerical metadata to GEP usage across samples for each community