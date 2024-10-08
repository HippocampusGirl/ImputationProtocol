# Imputation Protocol (as a container)

[![build](https://github.com/HippocampusGirl/ENIGMAImputationProtocol/actions/workflows/build.yml/badge.svg)](https://github.com/HippocampusGirl/ENIGMAImputationProtocol/actions/workflows/build.yml)

> This project is currently at an experimental stage and has not been validated by an experienced geneticist. Please use at your own risk.

With the advent of stricter data privacy laws in many jurisdictions, it has become impossible for some researchers to use the [Michigan Imputation Server](https://imputationserver.readthedocs.io/en/latest/) to phase and impute genotype data. This project allows you to use the open source code behind the server on your local workstation or high-performance compute cluster, together with the [1000 Genomes Phase 3 v5](https://imputationserver.readthedocs.io/en/latest/reference-panels/#1000-genomes-phase-3-version-5) reference, and the rest of the [ENIGMA Imputation Protocol](https://enigma.ini.usc.edu/wp-content/uploads/2020/02/ENIGMA-1KGP_p3v5-Cookbook_20170713.pdf), .

## System requirements

This document assumes that you have either [`Singularity`](https://sylabs.io/guides/3.7/user-guide/quick_start.html) or [`Docker`](https://docs.docker.com/engine/install/) installed on your system.

## Usage

The container comes with all the software needed to run the imputation protocol. You can either do this manually based on the [official instructions](https://enigma.ini.usc.edu/wp-content/uploads/2020/02/ENIGMA-1KGP_p3v5-Cookbook_20170713.pdf), or use the step-by-step guide below.

The guide makes some suggestions for folder names (e.g. `mds`, `raw`, `qc`), but these can also be chosen freely. The only exceptions to this rule are the folders `cloudgene`, `hadoop`, `downloads`, `input` and `output` inside the working directory (in `/data` inside the container).

<ol>

<li>
<p>
You need to download the container file using one of the following commands. This will use approximately one gigabyte of storage.
</p>
<table>
<thead>
  <tr>
    <th><b>Container platform</b></th>
    <th><b>Version</b></th>
    <th><b>Command</b></th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>Singularity</td>
    <td>3.x</td>
    <td><code>wget <a href="http://download.gwas.science/singularity/imputation-protocol-latest.sif">http://download.gwas.science/singularity/imputation-protocol-latest.sif</code></a></td>
  </tr>
  <tr>
    <td>Docker</td>
    <td></td>
    <td><code>docker pull gwas.science/imputation-protocol:latest</code></td>
  </tr>
</tbody>
</table>
</li>

<li>
<p>
You will now need to create a working directory that can be used for intermediate files and outputs. This directory should be empty and should have sufficient space available. We will store the path of the working directory in the variable <code>working_directory</code>, and then create the new directory and some subfolders.
</p>
<p>
Usually, you will only need to do this once, as you can re-use the working directory for multiple datasets. Note that this variable will only exist for the duration of your terminal session, so you should re-define it if you exit and then resume later.
</p>

```bash
export working_directory=/mnt/scratch/imputation
mkdir -p -v ${working_directory}/{raw,mds,qc}
```

</li>

<li>
<p>
Copy your raw data to the <code>raw</code> subfolder of the working directory. If you have multiple <code>.bed</code> file sets that you want to process, copy them all.
</p>

```bash
cp -v my_sample.bed my_sample.bim my_sample.fam ${working_directory}/raw
```

</li>

<li>
<p>
Next, start an interactive shell inside the container using one of the following commands.
</p>
<p>
The <code>--bind</code> (Singularity) or <code>--volume</code> (Docker) parameters are used to make the working directory available inside the container at the path <code>/data</code>. This means that in all subsequent commands, we can use the path <code>/data</code> to refer to the working directory.
</p>
<table>
<thead>
  <tr>
    <th><b>Container platform</b></th>
    <th><b>Command</b></th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>Singularity</td>
    <td><code>singularity shell --hostname localhost --bind ${working_directory}:/data --bind /tmp imputation-protocol-latest.sif</code></td>
  </tr>
  <tr>
    <td>Docker</td>
    <td>
        <code>docker run --interactive --tty --volume ${working_directory}:/data --bind /tmp gwas.science/imputation-protocol /bin/bash</code>
    </td>
  </tr>
</tbody>
</table>
</li>

<li>
<p>
Inside the container, we will first go to <code>/data/mds</code>, and run the script <code>enigma-mds</code> for your <code>.bed</code> file set. The script creates the files <code>mdsplot.pdf</code> and <code>HM3_b37mds2R.mds.csv</code>, which are summary statistics that you will need to share with your working group as per the <a href="https://enigma.ini.usc.edu/wp-content/uploads/2020/02/ENIGMA-1KGP_p3v5-Cookbook_20170713.pdf">ENIGMA Imputation Protocol</a>.
</p>
<p>
Note that this script will create all output files in the current folder, so you should use <code>cd</code> to change to the <code>/data/mds/sample</code> folder before running it.
</p>
<p>
If you have multiple <code>.bed</code> file sets, you should run the script in a separate folder for each one. Otherwise the script may overwrite previous results when you run it again. 
</p>

<p>
If you have just one dataset:
</p>

```bash
mkdir /data/mds/sample

cd /data/mds/sample
enigma-mds --bfile /data/raw/my_sample
```

<p>
Alternatively, for multiple datasets:
</p>

```bash
mkdir /data/mds/{sample_a,sample_b}

cd /data/mds/sample_a
enigma-mds --bfile /data/raw/sample_a

cd /data/mds/sample_b 
enigma-mds --bfile /data/raw/sample_b

```

</li>

<li>
<p>
Next, we will set up our local instance of the <a href="https://imputationserver.readthedocs.io/en/latest/">Michigan Imputation Server</a>.
</p>
<p>
The <code>setup-hadoop</code> command will start a <a href="https://hadoop.apache.org/">Hadoop</a> instance on your computer, which consists of four background processes. When you are finished processing all your samples, you can stop them with the  <code>stop-hadoop</code> command. If you are using Docker, then these processes will be stopped automatically when you exit the container shell.
</p>
<p>
The <code>setup-imputationserver</code> script will then verify that the Hadoop instance works, and then install the <a href="https://imputationserver.readthedocs.io/en/latest/reference-panels/#1000-genomes-phase-3-version-5">1000 Genomes Phase 3 v5</a> genome reference that will be used for imputation (around 15 GB of data, so it may take a while).
</p>
<p>
If you are resuming analyses in an existing working directory, and do not still have the Hadoop background processes running, then you should re-run the setup commands. If they are still running, then you can skip this step.
</p>

```bash
setup-hadoop --n-cores 8
setup-imputationserver
```

<p>
If you encounter any warnings or messages while running these commands, you should consult with an expert to find out what they mean and if they may be important. However, processing will usually complete without issues, even if some warnings occur. 
</p>
<p>
If something important goes wrong, then you will usually see a clear error message that contains the word "error". Please note that if the commands take more than an hour to run the setup, then that may also indicate that an error occurred.
</p>

</li>

<li>
<p>
Next, go to <code>/data/qc</code>, and run <code>enigma-qc</code> for your <code>.bed</code> file sets. This will drop any strand ambiguous SNPs, then screen for low minor allele frequency, missingness and *Hardy-Weinberg equilibrium*, then remove duplicate SNPs (if necessary), and finally convert the data to sorted <code>.vcf.gz</code> format for imputation.
</p>
<p>
The script places intermediate files in the current folder, and the final <code>.vcf.gz</code> files in <code>/data/input/my_sample</code> where they can be accessed by the <code>imputationserver</code> script in the next step.
</p>
<p>
Note that this script will create some output files in the current folder, so you should use <code>cd</code> to change to the <code>/data/qc/sample</code> folder (or similar) before running it.
</p>
<p>
The input path is hard-coded, because the imputation server is quite strict and expects a directory with just the <code>.vcf.gz</code> files and nothing else, so to avoid any problems we create that directory automatically.
</p>
<p>
If you have just one dataset:
</p>

```bash
mkdir /data/qc/sample

cd /data/qc/sample
enigma-qc --bfile /data/raw/my_sample --study-name my_sample
```

<p>
Alternatively, for multiple datasets:
</p>

```bash
mkdir /data/qc/{sample_a,sample_b}

cd /data/qc/sample_a
enigma-qc --bfile /data/raw/sample_a --study-name sample_a

cd /data/qc/sample_b 
enigma-qc --bfile /data/raw/sample_b --study-name sample_b

```
</li>

<li>
<p>
Finally, run the <code>imputationserver</code> command for the correct <a href="https://github.com/genepi/imputationserver/blob/v1.5.8/docs/getting-started.md#population">sample population</a>. This parameter can be set to <code>mixed</code> if the sample has multiple populations or the population is unknown. 
<p>
If a specific population (not <code>mixed</code>) is specified, then the imputation server will compare <a href="https://imputationserver.readthedocs.io/en/latest/pipeline/#quality-control">the minor allele frequency of each variant in the sample to the population reference using a chi-squared test</a>, and excludes outliers. If the population is <code>mixed</code>, then the step is skipped.
</p>
<p>
Regardless of the population paramater, the imputation step always uses the entire <a
href="https://imputationserver.readthedocs.io/en/latest/reference-panels/#1000-genomes-phase-3-version-5">1000 Genomes Phase 3 v5</a> genome reference.
</p>
<p>
<b>If you see any errors (such as <code>obvious stand flips detected</code>), you may need to follow one or more of the workarounds in the <a href="#troubleshooting">troubleshooting</a> section.</b>
</p>

```bash
imputationserver --study-name my_sample --population mixed
```
<p>
This process will likely take a few hours, and once it finishes for all your <code>.bed</code> file sets, you can exit the container using the <code>exit</code> command.
</p>
<p>
All outputs can be found in the working directory created earlier. The quality control report can be found at <code>${working_directory}/output/my_sample/qcreport/qcreport.html</code> (only if the population is not <code>mixed</code>), and the imputation results at <code>${working_directory}/output/my_sample/local</code>. The <code>.zip</code> files are encrypted with the password <code>password</code>.
</p>
</li>

<li>
<p>
To merge all output files into a compact and portable <code>.zip</code> archive, the container includes the <code>make-archive</code> command. It will create a single output file at <code>${working_directory}/my_sample.zip</code> with all output files.
</p>

```bash
make-archive --study-name my_sample
```

<p>
Add the `--zstd-compress` option to the command to use a more efficient compression algorithm. This will take longer to run, but will create a smaller output file (around 70% smaller).
</p>

</li>

<li>
<p>
Once you have finished processing all your datasets, you can stop all background processes of the imputation server with the <code>stop-hadoop</code> command. Then you can exit the container, move the output files to a safe location and delete the working directory (in case you need the disk space).
</p>

```bash
stop-hadoop
exit
mv ${working_directory}/my_sample.zip /storage
rm -rI ${working_directory}
```
</li>

</ol>

## Troubleshooting

### My data is not in `.bed` file set format

You will need to convert your data before you start. Usually, this is very straight-forward with the [`plink` command](https://www.cog-genomics.org/plink/1.9/input).

#### `.ped` file set format

```bash
plink --ped my_sample.ped --map my_sample.map --make-bed --out my_sample
```

### Genome reference

> Error: No chunks passed the QC step. Imputation cannot be started!

This error happens when your input data uses a different genome reference from what the imputation server expects (`hg19`). You can do this manually with [`LiftOver`](https://genome.sph.umich.edu/wiki/LiftOver). To automatically do that, the container comes with the `check hg19` command, which is based on the [RICOPILI](https://sites.google.com/a/broadinstitute.org/ricopili/) command `buigue`.

```bash
check hg19 --bfile /data/raw/my_sample 
```

The command will create a `.bed` file set at `/data/raw/my_sample.hg19` which will have the correct genome reference. You should now use this corrected file to re-run the `enigma-qc` script, and then retry running the `imputationserver` command. 

### Strand flips

> Error: More than 100 obvious strand flips have been detected. Please check strand. Imputation cannot be started!

If the `imputationserver` command fails with this error, then you will need to resolve strand flips in your data. To automatically do that, the container comes with the `check flip` command, which is based on the [RICOPILI](https://sites.google.com/a/broadinstitute.org/ricopili/) called `checkflip4` and [check-bim](https://www.well.ox.ac.uk/~wrayner/tools/).

```bash
check flip --bfile /data/raw/my_sample
```

The command will create a `.bed` file set at `/data/raw/my_sample.check-flip` which will have all strand flips resolved. You should now use this corrected file to re-run the `enigma-qc` script, and then retry running the `imputationserver` command.

### Velocity

> Job execution failed: Velocity could not be initialized!

You likely have bad permissions in your working directory. You can either try to fix them, or start over with a fresh working directory.

### VCF violates the official specification

> Warning: At least one VCF allele code violates the official specification; other tools may not accept the file. (Valid codes must either start with a '<', only contain characters in {A,C,G,T,N,a,c,g,t,n}, be an isolated '\*', or represent a breakend.)

This message occurs for example when you have indels in your raw data. You can ignore this message, because the [Michigan Imputation Server](https://imputationserver.readthedocs.io/en/latest/) will remove these invalid variants in its quality control step.

### The commands are stuck, for example at `Init HadoopUtil null`

This suggests that your Hadoop instance may not be accepting new jobs. The fastest way to solve this is to stop and delete the instance, and then to re-run the setup.

```bash
# stop and delete
stop-hadoop
rm -rf /data/hadoop

# re-run setup
setup-hadoop --n-cores 8
setup-imputationserver
```

### Error `Application or file imputationserver not found`

Please delete the `/data/cloudgene` folder inside the container and run `setup-imputationserver` again.


