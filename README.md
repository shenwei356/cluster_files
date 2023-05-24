# cluster_files

`cluster_files` clusters files into multiple directories by creating symbolic links or moving files.

It is helpful in bioinformatic analyses, where multiple datasets are analysed in a series of steps, each with one or more methods.

## Features

- **Safe**. Creating symbolic links keeps the original files untouched.
- **Convenient** for parallel processing of multiple datasets with the same input file structure.
    -  You can use [rush](https://github.com/shenwei356/rush) or [parallel](https://www.gnu.org/software/parallel/) for local batch processing,
    -  and [easy_qsub](https://github.com/shenwei356/easy_qsub) or [easy_sbatch](https://github.com/shenwei356/easy_sbatch) for batch submitting jobs to a computer cluster.
- Each analysis step can be separately performed in its directory
    - **Clear** organization
    - Avoid conflicts
    - Supporting simultaneous analyses with multiple methods

## Best practice

1. Raw data.

        $ tree data/
        data/
        ├── A-1_R1_001.fastq.gz
        ├── A-1_R2_001.fastq.gz
        ├── A-2_R1_001.fastq.gz
        ├── A-2_R2_001.fastq.gz
        └── MD5
            ├── A-1_R1_001.fastq.gz.md5
            ├── A-1_R2_001.fastq.gz.md5
            ├── A-2_R1_001.fastq.gz.md5
            └── A-2_R2_001.fastq.gz.md5

    Make them read-only for safety, and keep the original file names.

        chmod -R a-w data/

1. Create another directory and create symbolic links.

        mkdir raw; cd raw;
        find ../data/ -name "*.fastq.gz" \
            | while read f; do ln -s $f; done
        cd ..

        $ tree raw
        raw/
        ├── A-1_R1_001.fastq.gz -> ../data/A-1_R1_001.fastq.gz
        ├── A-1_R2_001.fastq.gz -> ../data/A-1_R2_001.fastq.gz
        ├── A-2_R1_001.fastq.gz -> ../data/A-2_R1_001.fastq.gz
        └── A-2_R2_001.fastq.gz -> ../data/A-2_R2_001.fastq.gz

    Rename the symbolic links with [brename](https://github.com/shenwei356/brename):

        brename -p '_R(\d)_.+' -r '_${1}.fq.gz' raw/

        $ tree raw
        raw
        ├── A-1_1.fq.gz -> ../data/A-1_R1_001.fastq.gz
        ├── A-1_2.fq.gz -> ../data/A-1_R2_001.fastq.gz
        ├── A-2_1.fq.gz -> ../data/A-2_R1_001.fastq.gz
        └── A-2_2.fq.gz -> ../data/A-2_R2_001.fastq.gz

1. Cluster files.

        cluster_files -p '(.+?)_[12]\.fq\.gz$' raw/ -o raw.cluster

        $ tree raw.cluster
        raw.cluster
        ├── A-1
        │   ├── A-1_1.fq.gz -> ../../raw/A-1_1.fq.gz
        │   └── A-1_2.fq.gz -> ../../raw/A-1_2.fq.gz
        └── A-2
            ├── A-2_1.fq.gz -> ../../raw/A-2_1.fq.gz
            └── A-2_2.fq.gz -> ../../raw/A-2_2.fq.gz

1. QC with fastp, [rush](https://github.com/shenwei356/rush) is used for batch processing.
   In this step, we do not use `cluster_files`.

        s=raw.cluster
        t=raw.cluster.fastp

        mkdir -p $t
        ls -d $s/* | rush -v t=$t 'mkdir -p {t}/{%}'

        minlen=70
        j=8
        J=16
        minq=25
        ls -d $s/* \
            | rush -j $j -v t=$t -v l=$minlen -v j=$J -v q=$minq  \
                -v 'p={}/{%}'  -v 'op={t}/{%}/{%}' \
                '{ time fastp -i {p}_1.fq.gz -I {p}_2.fq.gz -o {op}_1.fq.gz -O {op}_2.fq.gz \
                        --unpaired1 {op}_1.unpaired.fq.gz --unpaired2 {op}_2.unpaired.fq.gz \
                        -l {l} -q {q} -W 2 -M {q} -3 {q} --thread {j} \
                        --trim_poly_g --poly_g_min_len 5 --low_complexity_filter \
                        --html {op}.fastp.html --json {op}.fastp.json ; } &> {op}.fastp.log' \
                -c -C fastp.rush --eta

        $ tree raw.cluster.fastp/
        raw.cluster.fastp/
        ├── A-1
        │   ├── A-1_1.fq.gz
        │   ├── A-1_1.unpaired.fq.gz
        │   ├── A-1_2.fq.gz
        │   ├── A-1_2.unpaired.fq.gz
        │   ├── A-1.fastp.html
        │   ├── A-1.fastp.json
        │   └── A-1.fastp.log
        └── A-2
            ├── A-2_1.fq.gz
            ├── A-2_1.unpaired.fq.gz
            ├── A-2_2.fq.gz
            ├── A-2_2.unpaired.fq.gz
            ├── A-2.fastp.html
            ├── A-2.fastp.json
            └── A-2.fastp.log

1. Assemble with megahit.

        s=raw.cluster.fastp
        t=raw.cluster.fastp.megahit

        # link the paired reads
        cluster_files -p '(.+)_[12].fq.gz$'          $s -o $t
        # link the unpaired reads
        cluster_files -p '(.+)_[12].unpaired.fq.gz$' $s -o $t

        $ tree raw.cluster.fastp.megahit
        raw.cluster.fastp.megahit
        ├── A-1
        │   ├── A-1_1.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_1.fq.gz
        │   ├── A-1_1.unpaired.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_1.unpaired.fq.gz
        │   ├── A-1_2.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_2.fq.gz
        │   └── A-1_2.unpaired.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_2.unpaired.fq.gz
        └── A-2
            ├── A-2_1.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_1.fq.gz
            ├── A-2_1.unpaired.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_1.unpaired.fq.gz
            ├── A-2_2.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_2.fq.gz
            └── A-2_2.unpaired.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_2.unpaired.fq.gz


        # -------------------------------------------

        conda activate megahit

        ls -d $t/* \
            | rush -j 4 -v 'p={}/{%}' \
                '{ time megahit -1 {p}_1.fq.gz -2 {p}_2.fq.gz -r {p}_1.unpaired.fq.gz,{p}_2.unpaired.fq.gz -o {}/megahit \
                    --presets meta-sensitive -t 40 -m 0.4 ; } &> {}/megahit.log' \
                -c -C megahit.rush --verbose --eta

        $ tree raw.cluster.fastp.megahit
        raw.cluster.fastp.megahit
        ├── A-1
        │   ├── A-1_1.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_1.fq.gz
        │   ├── A-1_1.unpaired.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_1.unpaired.fq.gz
        │   ├── A-1_2.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_2.fq.gz
        │   ├── A-1_2.unpaired.fq.gz -> ../../raw.cluster.fastp/A-1/A-1_2.unpaired.fq.gz
        │   ├── megahit
        │   │   └── files omitted
        │   └── megahit.log
        └── A-2
            ├── A-2_1.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_1.fq.gz
            ├── A-2_1.unpaired.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_1.unpaired.fq.gz
            ├── A-2_2.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_2.fq.gz
            ├── A-2_2.unpaired.fq.gz -> ../../raw.cluster.fastp/A-2/A-2_2.unpaired.fq.gz
            ├── megahit
            │   └── files omitted
            └── megahit.log

1. Assemble with another tool.

        s=raw.cluster.fastp
        t=raw.cluster.fastp.xxxx

        # link the paired reads
        cluster_files -p '(.+)_[12].fq.gz$'          $s -o $t
        # link the unpaired reads
        cluster_files -p '(.+)_[12].unpaired.fq.gz$' $s -o $t

        # do somethings


1. All directories.

        data
        raw
        raw.cluster
        raw.cluster.fastp
        raw.cluster.fastp.megahit
        raw.cluster.fastp.xxxx


## Installation

`cluster_files` is a single script written in Python using standard libraries.
It's Python 2/3 compatible, and version 2.7 or a later version is needed.

You can simply save the [script](https://raw.githubusercontent.com/shenwei356/cluster_files/master/cluster_files)
into any directory included in environment `PATH`, e.g `/usr/local/bin`.

Or

    git clone https://github.com/shenwei356/cluster_files.git
    cd cluster_files

    mkdir -p $HOME/bin; cp cluster_files /usr/local/bin

    # sudo cp cluster_files /usr/local/bin

## Usage

```
usage: cluster_files [-h] [-o OUTDIR] [-p PATTERN] [-k] [-m] [-f] indir

clustering files by regular expression (V4.0.0)

positional arguments:
  indir                 source directory

options:
  -h, --help            show this help message and exit
  -o OUTDIR, --outdir OUTDIR
                        out directory [<indir>.cluster]
  -p PATTERN, --pattern PATTERN
                        pattern (regular expression) of files in indir. if not given, it will be the longest common substring of the
                        files.GROUP (parenthese) should be in the regular expression. Captured group will be the cluster name. e.g.
                        "(.+?)_\d\.fq\.gz"
  -k, --keep            keep original dir structure
  -m, --mv              moving files instead of creating symbolic links
  -f, --force           force file overwriting, i.e. deleting existed out directory

https://github.com/shenwei356/cluster_files
```

## Support

Please [open an issue](https://github.com/shenwei356/cluster_files/issues) to report bugs,
propose new functions, or ask for help.

## License

[MIT License](https://github.com/shenwei356/cluster_files/blob/master/LICENSE)
