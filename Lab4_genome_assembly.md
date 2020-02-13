## Setting up your jetstream project and your key

Navigate to https://use.jetstream-cloud.org/

1. Click the login button at the top right of the screen.
2. Log in using the xsede account you set up.
3. Click on "Projects" (second tab).
4. Click "Create New Project".
5. Give it an informative name and description, then click "Create".
6. **In your terminal** type `ssh-keygen -t rsa -f $HOME/jetkey`.
7. It will ask you for a password - make sure you make it something you will remember.
8. It will generate a random picture for you - this is normal - and then type `more $HOME/jetkey.pub | cut -d " " -f1-2 | pbcopy`. The last step of this command copies the newly made key, so we can paste it later.
9. **Back on the jetstream page** click on your username in the top right corner and select "Settings".
10. Scroll down and click "Show More".
11. Click the plus sign at the bottom and a window should pop up titled "Add a public SSH key".
12. Name the key "jetkey" and paste (that key we copied in step 8) into the public key area. Click "Confirm".

**Steps 1-12 you *should* only have to do once, provided you are working on the same computer for each lab**


**The steps below are for launching an instance and connecting to it, which you will do every week.**

## Launching a jetstream instance and connecting to it via the terminal

1. Go back to the "Projects" tab and select the project you made earlier.
2. Click the pink "NEW" button on the left and select "Instance".
3. Search for (or scroll down to find) the correct type of machine for your instance (Ubuntu 18_04 Devel and Docker) and click it.
4. The third thing down on the right is "Instance Size" - adjust this to whatever size the lab protocol says. For this lab, we want an m1.medium instance.
5. Click "Launch Instance"
6. The instance will take a second to activate and launch fully. Sometimes it takes longer than other times. Be patient.
7. Copy the IP address of the instance once it is available.
8. **Back in the terminal** type `ssh -i $HOME/jetkey username@xxx.xxx.xxx.xxx` to connect to your instance. Replace "username" with your jetstream username, and "xxx.xxx.xxx.xxx" with your IP Address that you just copied.
9. When it asks you if you want to continue connecting, type "yes" and hit enter. 
10. Type in your password when prompted.
11. Should be connected!



## Getting things set up.

Your instance is a bit of a blank slate, so we'll need to install the programs we'll use in lab.

The first few commands we will use start with "sudo" - this is telling the computer that we have administrator privileges. They also have `apt-get`, which is just a program used for managing other software packages more easily.

**Question 1:** What happens when you run the first command without the "sudo" at the beginning?

Be careful with `sudo`! Having privileges on a computer means lots of opportunities to mess things up or delete things that are important. It is pretty rare that you will have these privileges on a computer, or you could get into trouble for trying them, so don't just go around sudoing things. Best to use it only on these instances where we can always terminate the instance and start over.

First we need to check the server for updates to existing software
> sudo apt-get update

Then, we install any required updates
> sudo apt-get -y upgrade

Now, we need to install some basic software that will allow us to do a lot more. Same structure as previous commands, but `install` instead of `update` or `upgrade`. See how this works?
Everything after the `install` part is a list of things we are installing.
> sudo apt-get -y install ruby build-essential python python-pip gdebi-core r-base git

Next, we'll install Conda. Conda also manages software packages, but specifically scientific ones - we will use this all the time.
> mkdir anaconda
> cd anaconda
> curl -LO https://repo.anaconda.com/archive/Anaconda3-5.1.0-Linux-x86_64.sh
> bash Anaconda3-5.1.0-Linux-x86_64.sh -b -p $HOME/anaconda/install/
> echo ". $HOME/anaconda/install/etc/profile.d/conda.sh" >> ~/.bashrc
> source ~/.bashrc
> conda update -n base conda

Now we'll make a "conda environment" that will hold all of our installations, activate it, and install the programs we'll need for today's lab. Recognize any of them?
While we will use a very similar process during each lab, the individual programs we will install will change.
> conda create -y --name gen711
> conda activate gen711
> conda install -y -c bioconda spades quast
> conda install -y -c bioconda -c conda-forge busco=4.0.2
> cd

There is one more program to install (this is an assembler that specializes in long reads) that needs to be installed differently.
> git clone https://github.com/ruanjue/wtdbg2
> cd wtdbg2 && make

Navigate back to your home directory.


## Lab 4 - Genome Assembly

Today in lab we are going to be assembling a bacterial genome (*Escherichia coli*) in two different ways. First, with long reads from PacBio, and then with short reads from Illumina.

We will then evaluate each genome assembly using BUSCO and Quast, using outputs from these programs to compare the two assemblies.

First let's download the data we'll be using for this lab.
Reference genome:
> curl -LO ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/005/845/GCF_000005845.2_ASM584v2/GCF_000005845.2_ASM584v2_genomic.fna.gz

PacBio data:
> curl -L -o pacbio.fastq http://gembox.cbcb.umd.edu/mhap/raw/ecoli_p6_25x.filtered.fastq

The Illumina data has a number of files associated with it (instead of just one like the PacBio data), so we download it as a "tarball" which is just a group of multiple files that get compressed together. We've used gzipped and zipped files in the past, and this is similar. The next command is to decompress those files so that we can see them as individuals.
Illumina data:
> curl -LO https://s3.amazonaws.com/gen711/ecoli_data.tar.gz
> tar -zxf ecoli_data.tar.gz

### Assembling the PacBio reads

This program uses what it called a "fuzzy" De Bruijn graph to make it's assemblies. You can read about it here: https://github.com/ruanjue/wtdbg2

There are two steps, one for the actual assembly and one for making the final consensus sequence. Neither take very long! Make sure you are paying attention to your output and input files for these programs.

Assembler:
The first part of this is the relative path to the command we are going to run (you could also give it an absolute path here and it would work great).
Next there are some options to choose and parameters to set. 
	- The `-x rs` is specifying the sequencing tech that was used to produce the reads - this is the code for PacBio (but there are others on the website above).
	- **Question 2:** Head to the website above (that github site) and find where it tells you what some of the options are for running this command. What does the `-g 4.6m` mean in this command?
	- The `-t 24` is the number of "threads" we are using for this command. It's sort of the number of simultaneous processes we want going at the same time, but be careful! More is not always better! It will depend on the type of computer you are working on.
	- The `-i` is where the input file goes, which in this case is our PacBio reads that we downloaded.
	- Finally the `-fo` is where you put the prefix of the output file name you want. Remember last week when we made the blast database and we just gave it the first part of the filename, and then it created three different files that all had different file extensions? This will be exactly like that. The "f" part means that it will force the program to overwrite any files that already have that name, which is great if you end up needing to run it multiple times, but less great if something is already named that - be careful!
> ./wtdbg2/wtdbg2 -x rs -g 4.6m -t 16 -i pacbio.fastq -fo long

Consenser (this is their word, it's a new one for me):
You'll notice that the path is the same, but the name of the command is different for this one.
**Question 3:** list off the different parts of this command and say what information they give the program.
> ./wtdbg2/wtpoa-cns -t 24 -i long.ctg.lay.gz -fo long.ctg.fa


Now we can evaluate the genome assembly we just made in two different ways. **Important** The output file from your last command (above) is going to be the input file of both of these two programs.

Run quast on the finished assembly. 
> quast long.ctg.fa -R GCF_000005845.2_ASM584v2_genomic.fna.gz -o long_reads_output --threads 24 --gene-finding

Run busco on the finished assembly.
> busco -m genome -i long.ctg.fa -o busco_long -l bacteria_odb10


Now assemble the short read data
> spades.py -t 24 -m 55 --mp1-rf -k 95  --pe1-1 ecoli_pe.1.fq  --pe1-2 ecoli_pe.2.fq  --mp1-1 nextera.1.fq  --mp1-2 nextera.2.fq  -o short_all_data

Run quast on the short reads
> quast short_all_data/scaffolds.fasta -R GCF_000005845.2_ASM584v2_genomic.fna.gz -o short_reads_output --threads 24 --gene-finding

Run busco on the short reads
> busco -m genome -i short_all_data/scaffolds.fasta -o busco_short -l bacteria_odb10





