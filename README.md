# msru

Investigating maximum short-read utility for structural variation

This repository contains the workflow and notes for creating most of the data for this research

# Setup
## Requirements

Assembly mapping and variant calling
- [bcftools](http://www.htslib.org/)
- [vcftools](https://vcftools.github.io/index.html)
- [truvari](https://github.com/spiralgenetics/truvari)
- [minimap2/paftools](https://github.com/lh3/minimap2)

Short-read SV discovery and genotyping
- [biograph](https://github.com/spiralgenetics/biograph)
- [manta](https://github.com/Illumina/manta)
- [paragraph](https://github.com/ACEnglish/paragraph)

## Repo structure
- scripts - Steps of the pipeline
- notebooks - Analysis notebooks for plot generation
- metadata - Mainly the sample metadata file
- parallel_helpers - Scripts that helped me parallelize the steps
- stats - Quick analysis joblib dataframes

## Genome References

Any reference genome can be used. We chose to subset references to only Autosomes and X/Y.
References need to be indexed using [`bwa index`](https://github.com/lh3/bwa). We have run
on hg19, grch38, chm13, and pr1. PR1 is not presented in the publication.

# Pipeline

Below are the steps of the pipeline to create the data and generate summaries.

## Download haplotype fastas

The haplotype fasta files are pulled from their public locations using 
```bash
bash eichler_download.sh
bash li_download.sh
```

## Assembly Stats:

Calculate the basic assembly stats with [calN50](https://github.com/lh3/calN50). This summary is inside the repository's `stats/asm_fastastat.jl`. An example `files.txt` is available in `metadata/assembly_fasta_meta.txt`
```bash
python n50_summary.py files.txt asm_fastastat.jl
```  

## Haplotype mapping and variant calling
Map each haplotype to a reference and call variants using mapping.sh

```bash
bash mapping.sh haplotype.1.fasta sample.hap1 reference.fasta
```

This creates an alignment file `sample.hap1.paf`, a vcf file `sample.hap1.var.vcf.gz`, and an index for that vcf.
A helper script `parallel_helpers/make_all_mapping_jobs.sh` can make per-sample bash scripts.

## Mapping Stats
If you collect per-haplotype mapping logs from `mapping.sh`, you can build a table from all the stats generated.
See consolidate_mapping_stats.py for details. This summary is also inside `stats/asm_mapstat.jl`

## Exact Intra-merge
Merge the two haplotype vcf files together using
  
```bash
bash merge_haps.sh sample.hap1.var.vcf.gz sample.hap2.var.vcf.gz sample output/path/var.vcf.gz
```

This creates a diploid vcf file name `sample.vcf.gz` and its index. This is the start of
our 'exact' merge VCFs. At this point, the files should be orgainzed in the below file
structure, thus the 4th argument 'output/path/var.vcf.gz'

## File structure

The per-sample vcfs are organized into sub-directories starting with the exact vcf path
`data/reference/project/sample/merge_strategy.vcf.gz` (e.g. `data/intra_merge/chm13/li/NA12878/exact.vcf.gz`)
Be sure to create the index for the VCFs, also

## Strict/Loose Intra-merge
Run truvari to collapse the exact intra-merge vcfs 

```bash
bash single_sample_collapse.sh $exact_vcf_path reference.fa
```
Where `$exact_vcf_path` is described above in File Structure.

This will run `truvari collapse` for strict and for loose merging and place them
in the appropriate directory. For example `data/chm13/li/NA12878/strict.vcf.gz` 

Each collapse also produces `removed.strict.vcf.gz`, vcf indexes, and logs in `collapse.strict.log`

## Make single sample summary stats

Beside a VCF, create a joblib DataFrame of just SVs >= 50bp using, for example

```bash
bash make_stats.sh data/chm13/exact.vcf.gz
```

Once these are all created, run
```bash
python consolidate_stats.py data/intra_merge single_sample_stats.jl
```

This will look for all subdirectories that have subdirectories that have subdirectories
containing '.jl' files. (a.k.a. reference/project/sample/merge_strategy.jl)
So that it can append columns of those path metadata information to each row, drop the
index, and make a usable dataframe for the SVCharacteristics notebook.

## Inter-sample merging

We now merge between samples. This needs to have a specific file-structure, also.
In a directory,  (e.g. `data/inter_merge`), we will create all the combinations of 
merges. For a single reference, we need to take all the exact 
merges and then do an exact merge to create `exact.exact.vcf.gz`

Do this by calling
```bash
python multi_merge.py data/intra_merge data/inter_merge/chm13 data/references/chm13.fa > the_merge_script.sh
```

This will create all the commands needed to make all combinations of merges for a single reference.
Repeat for each reference.

## Inter-merge stats

Once all of the samples have been processed, create the stats via
```bash
bash make_stats.sh grch38/exact/exact.vcf.gz
```

Consolidate the stats with 
```bash
python merge_stats.py *.jl
```
where  `*.jl` are all from the `intra_merge` made during the multi-sample merge step. Note that 
this assumes that `out_dir` is the name of a reference.

## Build the paragraph multi-sample VCFs

Given one of the multi-sample merges, create a paragraph reference out of only the svs using
```bash
 bash run_paragraphs.sh data/multi_merge/hg19/strict/strict.vcf.gz \
 						data/reference/hg19/hg19.fa \
						data/paragraph/hg19
```

This will try to run paragraph just enough to make the reference inputs needed to run a
modified version of paragraph https://github.com/ACEnglish/paragraph

## Other merges

To compare other merging methods, we made a custom script `naive_50.py` to do 50%
reciprocal overlap merging. We then downloaded three other SV merging tools to compare.

- [SURVIVOR v1.0.6](https://github.com/fritzsedlazeck/SURVIVOR)
- [Jasmine v1.1](https://github.com/mkirsche/Jasmine)

Using the strict intra-sample merge SVs, we ran each program with default parameters.  
First, collect the full paths to each of the vcfs you want to merge via
```bash
find /full/path/to/data/intra_merge/chm13/ -name "strict.vcf.gz"  > input_files.txt
```

Next, edit `perform_other_merges.sh` so that each of the programs' executable are pointed to
in their correct variable. 
Run this script inside of the working directory e.g. `data/other_merges/grch38/`.
```bash
bash perform_other_merges.sh input_files.txt
```
Note this creates a lot of temporay files to reformat things to accommodate these programs.
All the temporary files are stored in the `pwd`/temp and can be removed without affecting downstream
steps of this pipeline.

Additionally, to facilitate stats generation of these files, you can soft-link the joblibs of the truvari
and exact runs from the earlier inter_merge: e.g. `ln -s data/inter_merge/grch38/strict/strict.jl truvari.S.jl`

## Other merges stats consolidation
I'll make a script to pull it all and make a dataframe which will be input for the notebook

## Short-read SV discovery
!!Document how we made the short read discovery

## Short-read SV genoyping
!! Document how we made the short read genotyping

## Benchmarking short-read discovery
ABCDE

## Benchmarking short-read genotyping

1) Create a single VCF per sample:
vcf-concat grch38.chr*.vcf.gz | bgzip > grch38.wg.vcf.gz

2) run gtstats.sh on the base/comparison files to make the files
