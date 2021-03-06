---
feature_text: |
  ## Precision Medicine
title: AWS AMI Setup
categories:
    - Module-10-Appendix
feature_image: "assets/genvis-dna-bg_optimized_v1a.png"
date: 0010-03-01
---

This module is primarily for the course developers to document how the AWS AMI was developed for the course. Students will start from an AMI where some system setup will be done already, but students will still learn to install all necesary bioinformatics files.

***

### Initial AWS setup for development and testing purposes

For development purposes we started with a very large instance (overkill). Future experimentation is needed to determine the appropriate size for actual student instances.

- Launch an EC2 instance:
- Select Ubuntu Server 16.04 LTS (HVM), SSD Volume Type - ami-2581aa40
- Choose r4.16xlarge (64 vCPUs, 488 GiB Memory, 25 Gigabit Network Performance)
- Increase root storage to 10GB
- Add storage: 10,000 GiB (~10TB) EBS volume, not encrypted
- Configure security: Allow SSH access
- Login with key the usual way (e.g., ssh -i PMB.pem ubuntu@18.217.114.211)

### Formatting and mounting storage volumes
After initializing the EC2 instance we will need to mount and format the storage volume we allocated. Students will use this volume to install their own copies of tools used as well as input data and results. See [http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html) for guidance on setting up fstab records for AWS.
```bash
# start sudo shell
sudo bash

# get structure
lsblk

# make a workspace directory in root
cd /
mkdir workspace

# format the mount
mkfs /dev/xvdb

# mount the drive with the allocated space
mount /dev/xvdb /workspace
chown -R ubuntu:ubuntu /workspace

# Make ephemeral storage mounts persistent
echo -e "LABEL=cloudimg-rootfs / ext4 defaults,discard 0 0\n/dev/xvdb /workspace ext4 defaults,nofail 0 2" | tee /etc/fstab

# make symlink for convenience
cd ~
ln -s /workspace workspace

# exit sudo shell
exit

```

### Software Dependencies
Many of the tools used also have underlying dependencies, in many linux distributions these packages will already be installed and available. In this AMI setup however we start from a very basic Ubuntu distrubtion and we will have to install these dependencies. Ubuntu is based on the Debian operating system and so we can use the Debian based package manager `apt-get` for installation. Below the stand-alone dependencies required for each bioinformatic tool used in the course are supplied. In this we will use the normal linux convention where our own compiled binaries and executables are installed in `/usr/local/bin`.

#### Pre-Installation
Describes the general system wide dependencies required for downloading and decompressing source and binary files related to the tools to be installed. Also a few general use tools are listed as well.
```bash
# start sudo shell
sudo bash

# general tools for installation and use
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  wget \
  bzip2 \
  unzip \
  git \
  curl \
  tree

# exit sudo shell
exit
```

#### R 3.5.1
Describes dependencies and installation for R 3.5.1, used in this course for general file manipulation/analysis.
```bash
# start sudo shell
sudo bash

# Install R Dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  gfortran \
  libreadline-dev \
  libpcre3-dev \
  libcurl4-openssl-dev \
  build-essential \
  zlib1g-dev \
  libbz2-dev \
  liblzma-dev \
  openjdk-8-jdk

# Install R
wget https://cran.r-project.org/src/base/R-3/R-3.5.1.tar.gz
tar -zxvf R-3.5.1.tar.gz
cd R-3.5.1
./configure --prefix=/usr/local/ --with-x=no
make
make install

# devtools and BiocManager dependencies
apt-get update -y && apt-get install -y \
  libssl-dev \
  libxml2-dev

R --vanilla -e 'install.packages(c("devtools", "BiocManager"), repos="http://cran.us.r-project.org")'

# change write permissions so students can install additional packages
chown -R ubuntu:ubuntu /usr/local/lib/R/library
find /usr/local/lib/R/library -type d -exec chmod 777 {} \;
find /usr/local/lib/R/library -type f -exec chmod 664 {} \;
find /usr/local/lib/R/doc/ -type d -exec chmod 777 {} \;
find /usr/local/lib/R/doc/ -type f -exec chmod 664 {} \;

# exit sudo shell
exit
```

#### Samtools 1.7
Describes dependencies and installation for samtools 1.7, used in this course for general bam file manipulation.
```bash
# start sudo shell
sudo bash

# samtools dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  build-essential \
  libncurses5-dev \
  zlib1g-dev \
  libbz2-dev \
  liblzma-dev

# samtools installation
wget https://github.com/samtools/samtools/releases/download/1.7/samtools-1.7.tar.bz2
tar --bzip2 -xvf samtools-1.7.tar.bz2
cd samtools-1.7
./configure --prefix=/usr/local/
make
make install

# exit sudo shell
exit
```

#### PICARD 2.18.14
Describes dependencies and installation for PICARD 2.18.14, used in this course for general bam file manipulation and QC.
```bash
# start sudo shell
sudo bash

# picard dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  openjdk-8-jdk

# picard installation
wget https://github.com/broadinstitute/picard/releases/download/2.18.14/picard.jar
export PICARD='/usr/local/bin/picard.jar'

# exit sudo shell
exit
```

#### BWA 0.7.17
Describes dependencies and installation for BWA 0.7.17, used in this course for DNA alignment.
```bash
# start sudo shell
sudo bash

# bwa dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  build-essential \
  libz-dev

# bwa installation
wget https://cytranet.dl.sourceforge.net/project/bio-bwa/bwa-0.7.17.tar.bz2
tar --bzip2 -xvf bwa-0.7.17.tar.bz2
cd  bwa-0.7.17
make
ln -s /usr/local/bin/bwa-0.7.17/bwa /usr/local/bin/bwa

# exit sudo shell
exit
```

#### GATK 4.0.2.1
Describes dependencies and installation for GATK 4.0.2.1, used in this course for .....
```bash
# start sudo shell
sudo bash

# install miniconda dependency
cd /usr/local/bin
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh # choose /usr/local/bin/miniconda as install location
source ~/.bashrc

# install additional dependencies
apt-get update -y && apt-get install -y \
  openjdk-8-jdk

# install R dependencies
R --vanilla -e 'install.packages(c("gsalib", "ggplot2","reshape", "gplots"), repos="http://cran.us.r-project.org")'

# install GATK environment
wget https://github.com/broadinstitute/gatk/releases/download/4.0.2.1/gatk-4.0.2.1.zip
unzip gatk-4.0.2.1.zip
cd /usr/local/bin/gatk-4.0.2.1
conda env create -n gatk -f gatkcondaenv.yml
# to use gatk: source activate gatk
# note that we told the installer to add conda to the paths
# added by Miniconda3 installer
# export PATH="/usr/local/bin/miniconda/bin:$PATH"

# symlink gatk executable
ln -s /usr/local/bin/gatk-4.0.2.1/gatk /usr/local/bin/gatk

# exit sudo shell
exit
```

#### VEP 93.4
Describes dependencies for VEP 93.4, used in this course for variant annotation.
- Note: VEP natively supports gnomad allele frequencies but it is unclear if this works for all variants or only for dbSNP subset of variants.
See: http://useast.ensembl.org/info/docs/tools/vep/script/vep_other.html#assembly

- To access the full gnomAD data set, it is possible to use VEP's custom annotation feature to retrieve the frequency data directly from the gnomAD VCF files. See: http://useast.ensembl.org/info/docs/tools/vep/script/vep_example.html#gnomad

```bash
# start sudo shell
sudo bash

# Install VEP dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  libdbi-perl \
  libdbd-mysql-perl \
  build-essential \
  zlib1g-dev \
  libmodule-build-perl \
  cpanminus
cpanm -i Bio::Root::Version

# install vep with the various plugins
mkdir -p /opt/vep_cache
wget https://github.com/Ensembl/ensembl-vep/archive/release/93.5.zip
unzip 93.5.zip
cd ensembl-vep-release-93.5/
perl INSTALL.pl --CACHEDIR /opt/vep_cache # install cache, hg38:refseq,vep,merged

# make a symlink
ln -s /usr/local/bin/ensembl-vep-release-93.5/vep /usr/local/bin/vep

# Install required data for plugins
mkdir -p /opt/vep_cache/data
cd /opt/vep_cache/data
wget -c ftp://ftp.ensembl.org/pub/data_files/homo_sapiens/GRCh38/variation_genotype/gnomad.exomes.r2.0.1.sites.GRCh38.noVEP.vcf.gz
wget -c ftp://ftp.ensembl.org/pub/data_files/homo_sapiens/GRCh38/variation_genotype/gnomad.exomes.r2.0.1.sites.GRCh38.noVEP.vcf.gz.tbi

# unlock permissions for downloaded cache
find /opt/vep_cache -type d -exec chmod 777 {} \;
find /opt/vep_cache -type f -exec chmod 664 {} \;

# exit sudo shell
exit
```

#### VarScan 2.4.2
Describes dependencies and installation for VarScan 2.4.2, used in this course for variant calling
```bash
# start sudo shell
sudo bash

# varscan dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  openjdk-8-jdk

# install varscan
curl -L -k -o VarScan.v2.4.2.jar https://github.com/dkoboldt/varscan/releases/download/2.4.2/VarScan.v2.4.2.jar

# exit sudo shell
exit
```

#### BCFtools 1.3.1
Describes dependencies and installation for BCFtools 1.3.1, used in this course for manipulating VCF files.
```bash
# start sudo shell
sudo bash

# BCFtools dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  build-essential \
  libz-dev

# install BCFtools
curl -L -k -o bcftools-1.3.1.tar.bz2                https://github.com/samtools/bcftools/releases/download/1.3.1/bcftools-1.3.1.tar.bz2
tar --bzip2 -xvf bcftools-1.3.1.tar.bz2
cd bcftools-1.3.1
make -j
make prefix=/usr/local/ install

# exit sudo shell
exit
```

#### Strelka 2.7.1
Describes dependencies and installation for strelka, used in this course for variant calling. Note that Strelka requies python 2.
```bash
# start sudo shell
sudo bash

# strelka dependencies
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  python-dev

# install strelka
curl -L -k -o strelka-2.7.1.centos5_x86_64.tar.bz2 https://github.com/Illumina/strelka/releases/download/v2.7.1/strelka-2.7.1.centos5_x86_64.tar.bz2
tar --bzip2 -xvf strelka-2.7.1.centos5_x86_64.tar.bz2 # note uses python2

# exit sudo shell
exit
```

#### Sambamba 0.6.4
Describes dependencies and installation for Sambamba 0.6.4, used in this course for ....
```bash
# start sudo shell
sudo bash

# install sambamba
cd /usr/local/bin
curl -L -k -o sambamba_v0.6.4_linux.tar.bz2 https://github.com/lomereiter/sambamba/releases/download/v0.6.4/sambamba_v0.6.4_linux.tar.bz2
tar --bzip2 -xvf sambamba_v0.6.4_linux.tar.bz2
ln -s /usr/local/bin/sambamba_v0.6.4 /usr/local/bin/sambamba

# exit sudo shell
exit
```

#### HISAT 2.0.4
Describes dependencies and installation for HISAT 2.0.4, used in this course for RNA alignment.
```bash
# start sudo shell
sudo bash

# install hisat2
cd /usr/local/bin
wget ftp://ftp.ccb.jhu.edu/pub/infphilo/hisat2/downloads/hisat2-2.0.4-Linux_x86_64.zip
unzip hisat2-2.0.4-Linux_x86_64.zip
ln -s /usr/local/bin/hisat2-2.0.4/hisat2 /usr/local/bin/hisat2

# exit sudo shell
exit
```

#### StringTie 1.3.0
Describes dependencies and installation for StringTie 1.3.0, used in this course for transcript abundance estimates.
```bash
# start sudo shell
sudo bash

# install stringtie
cd /usr/local/bin
wget http://ccb.jhu.edu/software/stringtie/dl/stringtie-1.3.0.Linux_x86_64.tar.gz
tar -xzvf stringtie-1.3.0.Linux_x86_64.tar.gz
ln -s /usr/local/bin/stringtie-1.3.0.Linux_x86_64/stringtie /usr/local/bin/stringtie

# exit sudo shell
exit
```

#### Gffcompare 0.9.8
Describes dependencies and installation for Gffcompare 0.9.8, used in this course for ....
```bash
# start sudo shell
sudo bash

# intall Gff compare
cd /usr/local/bin
wget http://ccb.jhu.edu/software/stringtie/dl/gffcompare-0.9.8.Linux_x86_64.tar.gz
tar -xzvf gffcompare-0.9.8.Linux_x86_64.tar.gz
ln -s /usr/local/bin/gffcompare-0.9.8.Linux_x86_64/gffcompare /usr/local/bin/gffcompare

# exit sudo shell
exit
```

#### copyCat 1.6.12
Describes dependencies and installation for copyCat 1.6.12, used in this course for WGS copy number calling.
```bash
# start sudo shell
sudo bash

# Install copyCat dependencies (see R section for installing R)
cd /usr/local/bin
R --vanilla -e 'BiocManager::install(c("IRanges", "DNAcopy"))'

# Install copyCat
R --vanilla -e 'devtools::install_github("chrisamiller/copycat")'

# exit sudo shell
exit
```

#### CNVnator
Describes dependencies and installation for CNVnator, used in this course for exome copy number calling.
```bash
# start sudo shell
sudo bash

# Install CNVnator dependency root
cd /usr/local/bin
apt-get update -y && apt-get install -y \
  build-essential \
  libncurses5-dev \
  zlib1g-dev \
  libbz2-dev \
  liblzma-dev \
  build-essential \
  libxpm4
wget https://root.cern.ch/download/root_v6.14.04.Linux-ubuntu16-x86_64-gcc5.4.tar.gz
tar -xzvf root_v6.14.04.Linux-ubuntu16-x86_64-gcc5.4.tar.gz
export ROOTSYS=/usr/local/bin/root
export PATH=$ROOTSYS/bin:$PATH
export LD_LIBRARY_PATH=$ROOTSYS/lib:$LD_LIBRARY_PATH

# Install CNVnator dependency yeppp
wget http://bitbucket.org/MDukhan/yeppp/downloads/yeppp-1.0.0.tar.bz2
tar -xvjf yeppp-1.0.0.tar.bz2
export YEPPPLIBDIR=/usr/local/bin/yeppp-1.0.0/binaries/linux/x86_64
export YEPPPINCLUDEDIR=/usr/local/bin/yeppp-1.0.0/library/headers
export LD_LIBRARY_PATH=$YEPPPLIBDIR:$LD_LIBRARY_PATH

# install CNVnator
wget https://github.com/abyzovlab/CNVnator/releases/download/v0.3.3/CNVnator_v0.3.3.zip
unzip CNVnator_v0.3.3.zip
cd CNVnator_v0.3.3/src/samtools
make
cd ../
make

# make sylink
ln -s /usr/local/bin/CNVnator_v0.3.3/src/cnvnator /usr/local/bin/cnvnator

# exit sudo shell
exit
```

#### cnvkit
Describes dependencies and installation for cnvkit, used in this course for ...
```bash
# start sudo shell
sudo bash

# install cnvkit
cd /usr/local/bin
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
conda config --add channels defaults
conda config --add channels conda-forge
conda config --add channels bioconda
conda create -n cnvkit cnvkit
# source activate cnvkit

# exit sudo shell
exit
```

#### Kallisto 0.44.0
Describes dependencies and installation of Kallisto 0.44.0, used in this course for abundance estimation.
```bash
# start sudo shell
sudo bash

# install kallisto
cd /usr/local/bin
wget https://github.com/pachterlab/kallisto/releases/download/v0.44.0/kallisto_linux-v0.44.0.tar.gz
tar -zxvf kallisto_linux-v0.44.0.tar.gz
ln -s /usr/local/bin/kallisto_linux-v0.44.0/kallisto /usr/local/bin/kallisto

# exit sudo shell
exit
```

#### Pizzly 0.37.3
Describes dependencies and installation of Pizzly, used in this course for fusion detection.
```bash
# start sudo shell
sudo bash

# install Pizzly
cd /usr/local/bin
wget https://github.com/pmelsted/pizzly/releases/download/v0.37.3/pizzly_linux.tar.gz
tar -zxvf pizzly_linux.tar.gz

# exit sudo shell
exit
```

#### Manta 1.4.0
Describes dependencies and installation of Manta, used in this course for SV datection.
```bash
# download and extract
cd /usr/local/bin
wget https://github.com/Illumina/manta/releases/download/v1.4.0/manta-1.4.0.centos6_x86_64.tar.bz2
tar --bzip2 -xvf manta-1.4.0.centos6_x86_64.tar.bz2

# test installation
python2 /usr/local/bin/manta-1.4.0.centos6_x86_64/bin/configManta.py --help

# exit sudo shell
exit
```

#### mosdepth 0.2.3
Describes dependencies and installation of mosdepth, used in this course for dpeth caluclations.
```bash
# start sudo shell
sudo bash

# install mosdepth
cd /usr/local/bin
conda install mosdepth

# exit sudo shell
exit
```

#### extra utilities
Describes installation of extra software helpfull to instructors but not necessarily used by Students
```bash
# start sudo shell
sudo bash

# install faSplit
conda install ucsc-fasplit

# exit sudo shell
exit
```

### apache web serve setup
Set up apache web server for convenient access to files. This will allow students to easily download generated data from the `/workspace` directory. This directory is served from the IPv4 Public IP, which will be different for each user. This IP address can be viewed from the AWS EC2 instance site.
```bash
# start sudo shell
sudo bash

# install apache
apt-get update -y && apt-get install -y \
  apache2

# add the following to the config
#<Directory /workspace/>
#       Options Indexes FollowSymLinks
#       AllowOverride None
#       Require all granted
#</Directory>
sed -ie '/#<\/Directory>/a <Directory /workspace/>\n        Options Indexes FollowSymLinks\n        AllowOverride None\n        Require all granted\n</Directory>' /etc/apache2/apache2.conf

# change the document root in vhost file (000-default.conf)
sed -i 's/DocumentRoot \/var\/www\/html/DocumentRoot \/workspace/' /etc/apache2/sites-available/000-default.conf

# restart apache
service apache2 restart

# exit sudo shell
exit
```

### Environment Variables
We need to set some environment variables so that they are available to the studens when they start up the instance. Importantly we se variables for CNVnator dependencies so students can install the software more easily. these are all added to the '~/.bashrc' file.
```bash
# variables for ROOT
export ROOTSYS=/usr/local/bin/root
export PATH=$ROOTSYS/bin:$PATH
export LD_LIBRARY_PATH=$ROOTSYS/lib:$LD_LIBRARY_PATH

# variables for YEPP
export YEPPPLIBDIR=/usr/local/bin/yeppp-1.0.0/binaries/linux/x86_64
export YEPPPINCLUDEDIR=/usr/local/bin/yeppp-1.0.0/library/headers
export LD_LIBRARY_PATH=$YEPPPLIBDIR:$LD_LIBRARY_PATH
```

### Results Directory structure
Here we create the directory structure for holding results as well as downloading some of the larger raw_data files for the students.
```bash
# make directory structure
mkdir -p /workspace/data/raw_data/fastqs/all
mkdir -p /workspace/data/raw_data/fastqs/chr6
mkdir -p /workspace/data/raw_data/references/
mkdir -p /workspace/data/results/setup
mkdir -p /workspace/data/results/inputs
mkdir -p /workspace/data/results/align
mkdir -p /workspace/data/results/somatic
mkdir -p /workspace/data/results/germline
mkdir -p /workspace/data/results/rnaseq
mkdir -p /workspace/data/results/clinical
mkdir -p /workspace/data/results/immune
mkdir -p /workspace/data/results/cwl
mkdir -p /workspace/data/results/appendix

# change permissions for Students
find /workspace/data/ -type d -exec chmod 775 {} \;
chown -R ubuntu:ubuntu /workspace/data

# download the raw data
wget -c -P /workspace/data/raw_data/fastqs/chr6 http://genomedata.org/pmbio-workshop/fastqs/chr6/Exome_Norm.tar
wget -c -P /workspace/data/raw_data/fastqs/chr6 http://genomedata.org/pmbio-workshop/fastqs/chr6/Exome_Tumor.tar
wget -c -P /workspace/data/raw_data/references http://genomedata.org/pmbio-workshop/references/NimbleGenExome_v3.interval_list
wget -c -P /workspace/data/results/align http://genomedata.org/pmbio-workshop/results/all/alignments/Exome_Norm_sorted_mrkdup.bam
wget -c -P /workspace/data/results/align http://genomedata.org/pmbio-workshop/results/all/alignments/Exome_Tumor_sorted_mrkdup.bam
wget -c -P /workspace/data/results/align http://genomedata.org/pmbio-workshop/results/all/alignments/WGS_Norm_merged_sorted_mrkdup.bam
wget -c -P /workspace/data/results/align http://genomedata.org/pmbio-workshop/results/all/alignments/WGS_Tumor_merged_sorted_mrkdup.bam

# view directory structure
tree -d /workspace/data
```

### Final Cleanup
To finnish up clean out the downloaded compressed binary files
```bash
# start sudo shell
sudo bash

# clean things out
cd /usr/local/bin
rm -f *.tar.gz
rm -f *.zip
rm -f *.bz2

# exit sudo shell
exit
```
