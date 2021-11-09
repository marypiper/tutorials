# Cell Hashing

'Cell Hashing, where oligo-tagged antibodies against ubiquitously expressed surface proteins uniquely label cells from distinct samples, which can be subsequently pooled. By sequencing these tags alongside the cellular transcriptome, we can assign each cell to its original sample, robustly identify cross-sample multiplets, and “super-load” commercial droplet-based systems for significant cost reduction.' [Stoeckius, M, et. al.](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-018-1603-1)  

'Cell hashtags allow for robust sample multiplexing, confident multiplet identification, and discrimination of low-quality cells from ambient RNA. In addition to enabling “super-loading” of commercial scRNA-seq platforms to substantially reduce costs, this strategy represents a generalizable approach for multiplet identification and multiplexing that can be tailored to any biological sample or experimental design'. [Stoeckius, M, et. al.](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-018-1603-1)

Therefore, the benefits of cell hashing include:

- Reduction in batch effects since library preparation between conditions occurs at the same time.
- Reliable identification of multiplets (allows for removal of 'cells' containing more than one cell.
- Detection of low quality cells that contain only ambient RNA
- Super-loading allows for reduction of costs for high number of cells


## Generate FASTQ files from BCL files

Using the 10X Cell Ranger workflow, we can efficiently demultiplex our samples. The first step is to generate the FASTQ files as we normally would using `cellranger mkfastq`. However, for this step we need to **attain the indices that correspond to the gene expression reads and those that correspond to the antibody barcode reads from the sequencing facility**. We supply the indices using the standard samplesheet as described in the [documentation](https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/mkfastq?src=pr&lss=none&cnm=&cid=NULL&src=pr&lss=none&cnm=&cid=NULL#simple_csv). An example samplesheet for cell hashing with reads distributed across all lanes (`*`) is given below.

```
Lane,Sample,Index
*,gene,SI-GA-F1
*,barcode,SI-GA-F2
```

By running the `cellranger mkfastq` command with this samplesheet, we will generate separate FASTQ files for our gene expression reads and our antibody barcode reads. An example script for creating the FASTQ files is given below (the run_folder contains the `Data` directory which has the `Intensities` folder inside):

```
#!/bin/bash

#SBATCH -p priority             # partition name
#SBATCH -t 0-12:00              # hours:minutes runlimit after which job will be killed
#SBATCH --mem 32G
#SBATCH --job-name mkfastq             # Job name
#SBATCH -o %j.out                       # File to which standard out will be written
#SBATCH -e %j.err               # File to which standard err will be written
#SBATCH --mail-type=ALL
#SBATCH --mail-user=piper@hsph.harvard.edu

# Load modules
module load cellranger/6.1.0 bcl2fastq/2.20.0.422

cellranger mkfastq --id=mycellranger_mkfastq --run=path/to/run_folder --samplesheet=samplesheet_mkfastq.csv
```

## Perform alignment and counting

The next step is to perform the alignment and counting of the reads. The `cellranger count` pipline allows for the quantification of the gene expression and feature barcode for each cell barcode. Cell ranger expects the gene expression and feature barcodes to be in separate FASTQ files, which we generated in the `cellranger mkfastq` command previously. The [documentation](https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/feature-bc-analysis) is quite good and thorough. Basically, we need to create provide the following information to `cellranger count`:

- **Library CSV file**: This file assigns each cell to a condition. You need the following columns:
  - `fastqs`: path to the directory containing the demultiplexed FASTQ files
  - `sample`: same as the `Sample` given in the samplesheet for the `cellranger mkfastq` command
  - `library_type`: Should be 'Antibody Capture' for the antibody barcode FASTQ files for cell hashing experiments, and 'Gene Expression' for your gene expression FASTQ files
  - Below is an example `library.csv` file
    
    ```
    fastqs, sample, library_type
    cellranger_mkfastq/outs/fastq_path/HGFCGBGXK/, barcode, Antibody Capture
    cellranger_mkfastq/outs/fastq_path/HGFCGBGXK/, gene, Gene Expression
    ```
- **Feature Reference CSV file**: This file provides information for parsing the antibody capture barcode information for each sample and assigning it to a sample. **You will need to acquire this information from the group preparing the libraries.** The following information is needed:
  - Are the hashing antibodies from Biolegend? If so, then the parsing information can be found in the [documention](https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/feature-bc-analysis#feature-ref)
  - What antibodies did you use and what are their barcodes?
  - What are the corresponding samples? 

  Upon asking, the client returned the following, which was sufficient for proceeding:
  
  > •WT
  > TotalSeq™-B0301 anti-mouse Hashtag 1 Antibody
  > https://www.biolegend.com/en-us/search-results/totalseq-b0301-anti-mouse-hashtag-1-antibody-17771
  > Barcode Sequence: ACCCACCAGTAAGAC
  > 
  > •KO
  > TotalSeq™-B0302 anti-mouse Hashtag 2 Antibody
  > https://www.biolegend.com/en-us/search-results/totalseq-b0302-anti-mouse-hashtag-2-antibody-17772
  > Barcode Sequence: GGTCGAGAGCATTCA
  
  To construct the feature reference file, the client's barcoding information needs to be provided in a specific format. The `feature_ref.csv` file should have the following columns:
    - `id`: name for the output folder
    - `name`: name for feature
    - `read`: which read the antibody capture barcode is present within
    - `pattern`: the pattern used to parse the barcode
    - `sequence`: the barcode sequence for each sample
    - `feature_type`: for cell hashing, this should be 'Antibody Capture'
  
  An example `feature_ref.csv` file is given below:
  
  ```
  id, name, read, pattern, sequence, feature_type
  WT, WT_TotalSeqB0301, R2, 5PNNNNNNNNNN(BC)NNNNNNNNN, ACCCACCAGTAAGAC, Antibody Capture
  KO, KO_TotalSeqB0302, R2,5PNNNNNNNNNN(BC)NNNNNNNNN, GGTCGAGAGCATTCA, Antibody Capture
  ```
  
Now to run the `cellranger count` command, we can include this information:

  ```
  cellranger count --id=name_for_output_folder_in_feature_ref.csv\
                   --libraries=library.csv \
                   --transcriptome=/n/shared_db/mm10/uk/cellranger/6.0.0/6.0.0/refdata-gex-mm10-2020-A/ \ # change for experiment
                   --feature-ref=feature_ref.csv \
                   --expect-cells=15000 \ # change for experiment
                   --localcores 6 \
                   --localmem 64 
  ```