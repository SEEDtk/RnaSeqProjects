# Important Note

This document was prepared using Mycobacterium tuberculosis as an example, at a time when there were roughly 1500 samples available. For other reference genomes, the time estimates given will vary depending on the number of samples involved.

In a production environment, the log files should probably be given date-based names, and all be kept in the appropriate reference-genome subdirectory. To keep from getting too bogged down in minutiae, that was not done in these examples.

# Creating the Database

Currently, we are using a single SQLite database in **/vol/mg-binning/RnaSeqProject**, but we have endless options. The underlying software is an ERDB system written in Java which can rest just as easily on MySQL, or any other SQL-based system if we add support for it. We could have smaller databases—one for each genus, one for each reference genome, or whatever—so long as each reference genome's data is kept together. Basically, the data per genome is fairly isolated and can be kept separate from the data for other genomes.

- Change to the database directory. Currently this is **/vol/mg-binning/RnaSeqProject**. It is important that the **rnaseqdb.sql** file be in this directory. If not, the full path should be used in the following commands.
- To create the database itself, run the following command.

erdb.utils init --dbfile rnaseq.db rnqseqdb.sql

- - This creates the database as an SQLite in the file **rnqseq.db**.
    - To document the database, run the following command.

erdb.utils display --dbfile rnaseq.db --title "RNA Seq Database Diagram" > rnaseqdb.html

- - This creates a web page in the file **rnaseqdb.html**.
    - Assuming we stick with the single-database approach, this only needs to be done once.

# Setting up a Genome

Once you start working with a reference genome, you need to prime the database with the genome information. We need the list of CDS features, the genome name and length, and for each feature we need to know its common name, its primary alias, and most importantly, which groups it belongs in.

We have solid subsystem information for all the genomes in PATRIC; however, for some genomes, we might also want to include information on operons, modulons, and regulons. If this information is known, it can be loaded in this step.

We will need a full-scale GTO for the reference genome. The **genome.download patric** command creates this. After that, we need to create a subsystem map and a group file. The group file is created by the **rna.erdb groups** command and the subsystem map by the **genome.download subMap** command. Finally, the **rna.erdb metaLoad** command fills in the database.

The **rna.erdb groups** command needs to know which features are in biologically significant groupings. Subsystem groups can be identified easily from the GTO, but other groups-- such as operons, regulons, and modulons-- are not in the BV-BRC database and must be computed by examining the literature and filling in a file by hand. The input file for the non-subsystem groups is tab-delimited with headers and three columns. The first column contains the group type ("operon", "regulon", "modulon", or whatever). The second column contains the group name. In the literature, operon, regulon, and modulon names are generally short, so the database ID for the group will be the reference genome ID prefixed to the name you specify here and separated by a colon. (For example, atomic regulon 6 in the E coli reference genome is _511145.183:AR6_.) Finally, the third column contains a comma-delimited list of the features in the group, either using the FIG ID suffix (e.g. **peg.132**) or a known alias (e.g. **dnaK**). The following table shows how this would look for three typical groups.

| **type** | **name** | **Features** |
| --- | --- | --- |
| modulon | Zinc | peg.2032, peg.300, peg.2033, peg.1918, peg.1197, peg.1920, peg.3551, peg.1919 |
| regulon | AR81 | moaB, ybhO, narH |
| operon | hcaEFCBD | b2538, b2539, b2540, b2541, b2542 |

This example is for E coli. Note that both Blattner numbers and gene names are used. If the aliases are in BV-BRC, they will work. If an alias is not found, the command will throw an error so you have a chance to fix it before the database is updated. The ultimate goal was to make the file easy to build from what tends to be found in papers on the subject. You can create as many of these files as you want for a particular genome. They will all be reformatted and rolled into a single **groups.tbl** file for the metadata loader.

Once you have your literature-based groups files (and it is perfectly acceptable for there to be zero such files), you next need to pick a preferred alias. Each feature is stored with both the BV-BRC fig-id and its gene name, but you can also pick a third type of alias to be kept in the database. The alias is identified by a PERL regular expression pattern that matches it. So, for example, to include Blattner numbers for E Coli, you would use **b\\d+**. To use GenInfo identifiers, you would use **gi\\|\\d+**. To bypass the alternate alias entirely, use a pattern that will never match, such as ~.

With all this as background, the procedure is as follows.

- Change to the main directory (**/vol/mg-binning/RnaSeqProject**).
- Run the following command to create or update the subsystem map.

nohup genome.download subMap subMap.tbl >submap.log &

- - This command automatically creates a backup of the old subsystem map if one exists. If the map does not exist, it is created in the named file (**subMap.tbl**). This command generally takes less than a minute.
- To download the genome from PATRIC, run the following commands.

p3-echo _genome-id_ \> temp.txt

nohup genome.download patric --subsystems -c 1 _genome-id_ &lt; temp.txt &gt; genome.log &

- - The first command stashes the genome ID in a tiny input file.
    - The second command downloads the genome from BV-BRC in GTO format and populates the subsystems. It will be common to see messages about invalid subsystem names and roles in the log. This reflects the general sloppiness of the subsystem data. The GTO will be stored in the reference genome subdirectory with a name consisting of the genome ID itself followed by the suffix **.gto**.
    - This process generally takes less than a minute.
    - Now you can create the master group file from the literature-based group files.

nohup rna.erdb groups _genome-id_/_genome-id_.gto _litFile1 litFile2 …_ > _genome-id/_groups.tbl 2> groups.log &

- - Here _genome-id_ is used to identify both the directory for the reference genome's files and the GTO file of the genome itself.
    - _litFile1_ and _litFile2_ represent the various literature-group file names. There can be none or there can be many.
    - Note that the group names must be unique. Do not give two groups of different types the same name, or you will get an error.
    - This process usually takes less than a minute.
    - Finally, to upload the metadata to the database, run the following command.

nohup rna.erdb metaLoad --alias=_pattern_ --dbfile rnaseq.db _genome-id_/_genome-id_.gto _genome-id_/groups.tbl subMap.tbl > metaLoad.log &

- - Here _pattern_ is the alias pattern described above, and _genome-id_ is the ID of the reference genome, which identifies the directory containing most of the necessary files as well as the name of the GTO file.
    - This process usually takes less than a minute.

# Procedure for Processing RNA Seq Samples

- Change to the directory for the base genome. This has the same name as the genome ID under the **/vol/mg-binning/RnaSeqProject** directory. So, if the base genome is 83332.12, the directory would be **/vol/mg-binning/RnaSeqProject/83332.12**.
- Make sure the workspace exists on BV-BRC. This will be [**/parrello@patricbrc.org/BVBRC_Transcriptomics/Bacteria/_genus_/_species_/_genome-id_**](mailto:/parrello@patricbrc.org/BVBRC_Transcriptomics/Bacteria/genus/species/genome-id), where _genus_ is the genus, _species_ is the species name with spaces converted to underscores, and _genome-id_ is the reference genome ID.
- To get a list of the samples to process, run the following command.

nohup ncbi.utils query --sinceFile lastSra.tbl SAMPLE STRA RNA-Seq ORGN "_taxonomic name_" > samples._date_.tbl 2> ncbi.log &

- - In the above command, _taxonomic name_ is the taxonomic name string for the strain of the base genome, thus "**Mycobacterium tuberculosis H37Rv**" for 83332.12.
    - Here also _date_ is the current date, in YYYY-MM-DD format. This helps to organize the sample list files. The **samples._date_.tbl** file will contain a list of all the samples found along with various other useful bits of information.
    - The **lastSra.tbl** file will be updated with the current time so that future queries will only find new samples. If it does not exist, every matching sample will be retrieved, and a new file will be created.
    - The **ncbi.log** file will contain messages about the process.
    - The command usually takes a few minutes. For 83332.12 on 4/16/2023, it found 1508 samples in 70 seconds.
- To run the samples through the BV/BRC analysis process, run the following command:

nohup rna.erdb tpm --source SRA_FILE --maxIter 1000 _genome­-id_ samples._date_.tbl /parrello@patricbrc.org/BVBRC_Transcriptomics/Bacteria/_genus_/_species_/_genome-id_ _login_ > rna.log &

- - In the above command, _genome-id_ is the BV/BRC id for the reference genome. Note that it is used twice.
    - Here also _date_ is the current date, and must match the date used in the previous step.
    - _genus_ and _species_ should be the genus and species names, and are used to make the workspace easier to navigate. The entire workspace folder structure must already exist, down to the bottom level. Note that I use underscores instead of spaces everywhere to avoid a very obscure bug in the workspace software. This applies to the species name (for example, _Mycobacterium_tuberculosis_ instead of _Mycobacterium tuberculosis_).
    - Finally, _login_ must be a login with write access to the workspace, and you MUST be logged into that ID in your current session.
    - The **rna.log** file will contain messages about the process.
    - The command can take days. It may stop before it is finished, even if there are no errors, in which case you simply restart it. (This is simply a precaution to keep it from running away if NCBI or BV-BRC goes down.) The stopping is controlled by the -**\-maxIter** parameter. A good rule of thumb is that 100 iterations takes 12 hours (7 to 8 minutes per cycle) and processes 200 samples, so the above invocation will take 5 days and process up to 2000 samples. This is way more than you need, especially if it's not the first download for the genome.
    - There are some glitches. The command likes to retry failed tasks. If BV-BRC crashes and the jobs all fail to start it will burn through them all in a desperate attempt to get back on track. Even if it gave up on a job during a previous run, it may try again when you restart. (The exception is when there is a **Job Failed** file in the result folder.) If you know a sample is hopeless (as would be the case if it were deleted from NCBI or simply too low-quality to process), you should delete it from the input file to avoid wasting time during restarts.
    - This process took a week for the 1500 samples found for Mycobacterium tuberculosis.
    - Now we need to get the project and PUBMED information from NCBI. We will put this into a file named **srr._date_.tbl**, where _date_ is the date on the sample file produced above. Run the following command.

nohup ncbi.utils list PROJTABLE < samples._date_.tbl > srr._date_.tbl 2> ncbi.log &

- - We are still in the reference-genome subdirectory here. _date_ is the date on which the sample file was generated. The output will be a three-column table with sample ID, project ID, and pubmed ID for each sample that has a valid project. The pubmed ID will be blank in about a third of the cases.
    - This command takes a few minutes.
    - The next step downloads the sample data files from BV-BRC to the local hard drive. This makes the database update faster and more restartable. Run the following command.

nohup rna.erdb download --clear --filter samples._date_.tbl /parrello@patricbrc.org/BVBRC_Transcriptomics/Bacteria/_genus_/_species_/_genome-id_ _login_ . > download.log &

- - Here _date_ is the date from the original NCBI query to get the sample list. It is used to identify the sample list so we only download the samples from the current run. _genome-id_ is the reference genome ID, and _login_ is the appropriate BV-BRC login email.
    - The samples will be downloaded to the subdirectory **Output** of the current directory. This will be cleared before the download begins, or created if it does not exist yet.
    - This operation is restartable. If you restart it, remove the **\--clear** option.
    - The BV-BRC workspace is checked, and if all the samples in it are to be downloaded, it does a mass copy. Otherwise, it does a much slower file-by-file copy. In general, the mass copy will happen the first time you set up a genome, and the slower copy after that, when there are many fewer files to download.
    - If a full copy is used, the job will not produce any significant progress messages during the download; your only gauge for progress will be the number of files in the output directory.
    - Now we are finally ready to load the samples into the database. Run the following command.

nohup rna.erdb upload --ncbi srr._date_.tbl --dbfile ../rnaseq.db _genome-id_ Output > upload.log &

- - Here _date_ is the date of the original NCBI query, and it is used to identify the file with the project and pubmed information, and _genome-Id_ is the ID of the reference genome.
    - The sample data is taken from the files in the subdirectory **Output**, downloaded in the previous step.
    - The database file is identified by the **\--dbfile** option, and points to the master database in the parent directory.
    - The trace messages will identify one or more suspicious samples for various reasons. This is normal. A lot of bad data gets put in SRA. We keep the data, but we mark it for easy exclusion from reports.

# Clustering and Computing Baselines

With all the samples loaded, we cluster the expression data into feature groups and sample groups. The sample-group clustering is used to compute the baseline level for each feature.

The sample-group clustering should be redone each time new data is added for the reference genome. We expect the feature-group clustering and baseline data to remain fairly stable, so these are only needed after major updates.

- Change to the directory containing the RNA SEQ database, which is currently **/vol/mg-binning/RnaSeqProject**.
- Run the following command to generate the feature clusters.

nohup rna.erdb featCorr --dbfile rnaseq.db --save _genome-id_/corrFile.tbl _genome-id_ > _genome-id_/features.log &

- - Here _genome-id_ is the ID of the reference genome.
    - The individual feature-to-feature correlation coefficients are stored in the **corrFile.tbl** file in the reference-genome directory.
    - The cluster groups will have names of the form _CLXXX_ where _XXX_ is a number, and they will be stored in the database. The clustering is based on significant Pearson correlation. The level that indicates significance has been determined to be 0.7 by statistical analysis.
    - The command takes a few minutes.
    - At this point, it is useful to get a summary of the clustering information. Run the following command to generate a web page about the feature clusters now in the database.

nohup rna.erdb dbReport --dbfile rnaseq.db _genome-id_ FEATURE_CLUSTER > _genome-id_/fidClusters.html 2> report.log &

- - Here _genome-id_ is the ID of the reference genome.
    - The web page will be generated in the file **fidClusters.html** in the reference-genome subdirectory. The clusters will be in individual sections of the report, sorted from the largest to the smallest. Below the main table of each cluster is a summary of the feature groupings represented. For most genomes, this will only be subsystems. In the summary, the name of the grouping, the number of cluster members in it, and the percentage of the grouping covered by the cluster (coverage) will be displayed. (So, for example, _25% coverage_ means that the cluster includes a fourth of the features in the group.
    - This command takes a few seconds.
    - Now we need to correlate and cluster the samples. This is important, because sample clustering determines the weight of a sample when computing baseline expression levels. Run the following command.

nohup rna.erdb sampleCorr --dbfile rnaseq.db _genome_id_ > _genome-id_/clusters.tbl 2> clusters.log &

- - Here _genome-id_ is the ID of the reference genome. On completion, the **clusters.tbl** file in the reference-genome directory will contain a similarity measurement for each pair of samples.
    - This command is heavily parallelized, and usually takes a minute. The time is quadratic to the number of samples.
    - With the pair similarities computed, we can now form the sample clusters and put them in the database. This always involves a complete rebuild of the sample-clustering data. Run the following command.

nohup rna.erdb clusterLoad --dbfile rnaseq.db _genome-id_ _genome-id_/clusters.tbl > clusters.log &

- - Here _genome-id_ is the ID of the reference genome. The **clusters.tbl** file in the reference-genome directory was produced in the previous step.
    - This command takes less than a minute, but the time is quadratic with the total number of samples for the reference genome.
    - At this point it is useful to pause and write out a short report about the non-trivial sample clusters. Among other things, this helps to verify that the clustering worked. Run the following command.

nohup rna.erdb dbReport --dbfile rnaseq.db --sCorr _genome-id_/clusters.tbl _genome-id_ SAMPLE_CLUSTER_SUMMARY > _genome-id_/sampleCorrReport.tbl 2> clusters.log &

- - Here _genome-id_ is the ID of the reference genome. The **clusters.tbl** file in the reference-genome directory was produced in the above, and the report will appear in **sampleCorrReport.tbl**.
    - The minimum similarity score for clustering is 0.94, based on a statistical analysis of what is meaningful. The report contains mean, min, max, and standard deviation for everything in the cluster, and you can expect the standard deviation to be very close to 0 and everything else to be very close to 1.
    - Now we can compute the baselines for each feature. Run the following command.

nohup rna.erdb baseline --dbfile rnaseq.db _genome-id_ > baseline.log &

- - Here _genome-id_ is the ID of the reference genome.
    - The baseline is computed so that each sample cluster is weighted equally. This helps to remove sample bias.
    - The command usually takes a few seconds.
    - With the baselines computed, we can now run the main feature report, which tells us what the baseline levels are, and how good our data is for each feature. This should be done any time new samples are added, even if we skip the preceding step. Run the following command.

nohup rna.erdb dbReport --dbfile rnaseq.db _genome-id_ NORMAL_CHECK > _genome-id_/geneReport.tbl 2> geneReport.log &

- - Here _genome-id_ is the ID of the reference genome. The report will be produced in **geneReport.tbl** in the reference-genome subdirectory.
    - The **KS_p_value** column tells you how good the distribution is for a feature. This is a number between 1 and 0 with 0 being the best result.