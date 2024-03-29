# A script to create an array to run MitoFinder in Hydra using Spades assemblies 
How to use the script:

### Step 1
Run "MvSpadesContigs.sh" (command below) from the working directory to rename Spades contig output files (contigs.fasta) with unique individual samples names. This command will find all "*contigs.fasta" files in the subdirectories of the working directory and copy them to the working directory. During the copy it will prepend said fasta file with the name of it's immediate parent directory.

After this completes, a new directory called contigs will be created in the working directory into which the renamed fasta files will be moved into.

```
find . -type f -exec sh -c 'for f do x=${f#./}; if [[ $x == *"/contigs.fasta" ]]; then cp "$x" "${x////_}"; fi done' {} + && mkdir contigs/ && mv *_contigs.fasta contigs/;
```

### Step 2
Run array builder. Copy the large script below and run in Hydra. Script puts the output files in a directory you specify:

Example:
bash mf_array_builder_output.sh 
-i : path to input spades contains
-o: path to output directory
-r: path to reference genome
-g: genetic translation table number (1-25, see MitoFinder Manual). Common code examples, Cnidarians 5, most Invertebrates 6, Echinoderms and Flatworms 10.

Commands for running script:
```
bash mf_array_builder_output.sh -i /example/input/path/contigs -o /example/output/path/directory/ -g (number 1 - 25) -r /example/path/reference_genome.gb
```
This is the script to copy and run on Hydra. Save it as "mf_array_builder_output.sh"
```
#!bin/bash
############################################################
# Help menu                                                #
############################################################
Help(){
   # Display Help
   echo "Creates the .sou and .sh files needed to run mitofinder with spades contigs as a job array."
   echo
   echo "Syntax: $(basename "$0") [-i INPUT|-o OUTPUT|-h HELP|-g TRANSLATION TABLE|-r REFERENCE GENOME]"
   echo "options:"
   echo "-g     [Required] Enter integer corresponding to your organism's genetic translation table."
   echo "-h     Print this help."
   echo "-i     [Required] INPUT directory path of spades contig fasta files."
   echo "-o     [Required] OUTPUT directory path."
   echo "-r     [Required] Enter path to reference genome file."
   echo
}
############################################################
############################################################
# Main program                                             #
############################################################
############################################################
# Define the arguments
while getopts ":hi:o:r:g:" option; do
   case $option in
      h|-help) # Display help menu
         Help
         exit 1;;
      i) # Enter input directory path
         DIR=${OPTARG};;
      o) # Enter output directory path
		 OUT=${OPTARG};;
	  r) # Enter path to reference genome file
		 REF=${OPTARG};;
	  g) # Enter Organism genetic code following NCBI table
		 GC=${OPTARG};;
     \?) # Invalid option
         echo "Error: Invalid option"
		 echo "Use -h to see valid argument options"
         exit 1;;
   esac
done

shift "$(( OPTIND - 1 ))"

# Check for required arguments
if [ -z "$DIR" ] || [ -z "$OUT" ] || [ -z "$REF" ] || [ -z "$GC" ]; then
	echo "Arguments -i, -o, -r, and -g are required"
	echo "Use -h to access the help menu" >&2
  exit 1
fi

# Check if input directory path is formatted correctly
if [[ $DIR == */ ]]; then
	:
else
	DIR=${DIR}"/"
fi

# Check if input directory path is formatted correctly
if [[ $OUT == */ ]]; then
	:
else
	OUT=${OUT}"/"
fi

# Check if -o is an integer in the correct range
# Coming soon to a script near you

# Set task ID max value
TMAX=`ls $DIR | wc -l`

# Check to see if fasta_list exists
FLIST="fasta_list.txt"
if [ -f "$FLIST" ]
then
	rm $FLIST
else
	:
fi

# Create qsub parameters file
cat > qsub_mf_spades_array.sou << EOF
qsub \
 -pe mthread 8 \
 -q mThM.q \
 -l mres=96G,h_data=12G,h_vmem=12G,himem \
 -cwd \
 -j y \
 -N mf_spades_array.job \
 -o ../logs/'mf_spades_array_\$TASK_ID.log' \
 -t 1-$TMAX \
 -tc 50 \
 -b y \$PWD/mf_spades.sh
EOF

# Create gzip bash file
cat > mf_spades.sh << EOF
#!/bin/sh
# ----------------Modules------------------- #
source /etc/profile.d/modules.sh
module load bioinformatics/mitofinder
# ----------------Commands------------------- #
echo + \`date\` job \$JOB_NAME started in \$QUEUE with jobID=\$JOB_ID on \$HOSTNAME
echo + NSLOTS = \$NSLOTS
#
# Input path, genetic code number, reference genome path
DIR=$DIR
OUT=$OUT
GC=$GC
REF=$REF

# Check to see if list file containing the names of all the fastq.gz files
# exists. If not, create it. Is so, do nothing.
FLIST="fasta_list.txt"
if [ -f "\$FLIST" ]
then
	:
else
	ls \$DIR > "fasta_list.txt"
fi
# This converts the SGE_TASK_ID into a useful parameter. Here it is stored in "i" and
# then passed to awk. Awk then iterates through filenames in "FLIST" and stores the
# filename in "P" which is then passed through the commands below before moving on
# to the next filename and repeating the process until all enteries have been passed
i=\$SGE_TASK_ID
P=\`awk "NR==\$i" \$FLIST\`
SAMP=\${P%_L00*}

mitofinder \
-j \${SAMP} \
-o \${GC} \
-r \${REF} \
-a \${DIR}\${P}

mv \${SAMP}* \${OUT}
#
echo = \`date\` job \$JOB_NAME done
EOF

# Check if files were generated
if [ -f "mf_spades.sh" ] && [ -f "qsub_mf_spades_array.sou" ]
then
	echo "Successfully generated job array files:"
	echo "mf_spades.sh"
	echo "qsub_mf_spades_array.sou"
	echo "Before using comfirm these files generated correctly"
	echo "Launch the job array using 'source qsub_mf_spades_array.sou'"
else
	echo "Job files not generated"
	echo "Use -h to access the help menu"
	exit 1
fi

# Make the resulting bash file executable
chmod +x mf_spades.sh
```


### Step 3
Launch the array:
```
source qsub_mf_spades_array.sou
```
