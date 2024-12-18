# blast-tips-and-tricks
A (mostly) empirically acquired collection of BLAST+ tricks and things you should know

Notes:
- If unspecified, these pertain to `blastn`.
- Most are actually obvious if you think them through.

---

# Tips 

## Know the defaults

Look at the `-help`. No seriously look at it. Know your defaults and formats.

## Use default or extended formats

If your output is going to be used by other programs, consider the fields that the program requires. I like using the default `-outfmt 6` format. From the `blastn -help`:

```
*** Formatting options
 -outfmt <String>
   alignment view options:
     0 = Pairwise,
     1 = Query-anchored showing identities,
     2 = Query-anchored no identities,
     3 = Flat query-anchored showing identities,
     4 = Flat query-anchored no identities,
     5 = BLAST XML,
     6 = Tabular,

...

   Options 6, 7, 10 and 17 can be additionally configured to produce
   a custom format specified by space delimited format specifiers,
   or in the case of options 6, 7, and 10, by a token specified
   by the delim keyword. E.g.: "17 delim=@ qacc sacc score".
   The delim keyword must appear after the numeric output format
   specification.
   The supported format specifiers for options 6, 7 and 10 are:
            qseqid means Query Seq-id
               qgi means Query GI
...

   When not provided, the default value is:
   'qaccver saccver pident length mismatch gapopen qstart qend sstart send
   evalue bitscore', which is equivalent to the keyword 'std'
 
```

This includes the main relevant fields, if you want to add more fields, you can add them at the end (after `send`). Many programs will easily take the default format and you will avoid having to mess with the output:

- [BASTA](https://github.com/timkahlke/BASTA) for consensus-based Last Common Ancestor sequence taxonomy annotation
- [QIIME2](https://docs.qiime2.org/) you can [import the results into a `search-results.qza` artifact](#loading-a-blast-file-into-qiime2).
- [blast2taxonomy](https://github.com/tmaruy/blast2taxonomy)

## Pre-made DBs

- [NCBI BLAST web service DBs](https://ftp.ncbi.nlm.nih.gov/blast/db/)
- [NCBI refseq/TargetedLoci for specific genes](https://ftp.ncbi.nlm.nih.gov/refseq/TargetedLoci/)

If you were wondering how to get the `fasta` files for those DBs, a note from NCBI:

> In April 2024, the BLAST FASTA files in this directory will no longer be
> available. You can easily generate FASTA files yourself from the formatted
>BLAST databases by using the BLAST utility blastdbcmd that comes with the
> standalone BLAST programs. See NCBI Insights for more details
> https://ncbiinsights.ncbi.nlm.nih.gov/2024/01/25/blast-fasta-unavailable-on-ftp/

Some tips on how to dump the DBs [below](#dump-the-db)


## Missing hits of interest

Depending on the parameters and your application, BLAST can miss hits that may be interesting to you.

The `-max_target_seqs` option controls the maximum number of matched sequences that will be kept in the output. This means that aligned sequences beyond this number, will not be included in the output even if they pass the thresholds. 

In this [blog post from Peter Cock](https://blastedbio.blogspot.com/2024/02/blast-max-target-seq-meets-metabarcoding.html), he shows how BLAST may *"miss"* 100% identity hits in favour of longer alignments despite containing more mismatches (I quote *"miss"* because those hits do have better scores, but it's a miss relative to the current objective). This can happen if the `-max-target-seqs` is set too low (defaults to 100 in NCBI's web BLAST and 500 in command line BLAST):

```bash
blastn -help | grep ' -max_target_seqs' -A 5

 -max_target_seqs <Integer, >=1>
   Maximum number of aligned sequences to keep 
   (value of 5 or more is recommended)
   Default = `500'
    * Incompatible with:  num_descriptions, num_alignments
```

You can intuitively guess that, for something like amplicon studies it may be better to use as query, only the most informative region (i.e. the amplicon without the primers, which are conserved in your whole amplicon). However, in the case he describes: 

> **It so happens that the first 32bp of our typical Phytophthora amplicons (immediately after the forward primer site) are also conserved - and importantly often missing in published ITS1 sequences**. That means when using NCBI BLAST to check an amplicon, although we hope to see full length perfect matches, the most interesting sequences are often only about 85% of the query (due to the subject match in the database missing the first 32bp) but in the region of 99% to 100% identical. Importantly those hits may not be ranked first by the BLAST e-value or bitscore (but you can change the sort order online).

Just remember what it is that you are really looking for and consider whether the defaults (in output filtering and sorting) are appropiate. In any case you can also play with the `-perc_identity` (% identity) and `-qcov_hsp_perc` (% query cover) options to regulate the output. My go to here lately is to use a high value for `-max-target-seqs` (the design choice of defaulting to 500 in the command line BLAST should tell you something). 

And importantly, be aware how the tool actually works. Read a bit on [how the thresholds are applied](https://gist.github.com/sujaikumar/504b3b7024eaf3a04ef5?permalink_comment_id=1633611#gistcomment-1633611), and be disturbed (note that this relates to a different example):

> This can happen because **limits, including max target sequences, are applied in an early ungapped phase of the algorithm, as well as later. In some cases a final HSP will improve enough in the later gapped phase to rise to the top hits. In your case, relaxing the limit to 200 appears to have allowed hits that would have been excluded in the ungapped phase at 100 max target sequences to rise.**

## Avoid certain taxids

TODO


## On parallelization

TODO


# Tricks 

Important know-how's or stuff I haven't seen elsewhere.

## Dump the DB

Dump all the sequences:

`blastdbcmd -db storage/dbs/nt/nt -entry all > nt.fna`

Dump the accesion to taxid mapping:

`blastdbcmd -db storage/dbs/nt/nt -entry all -outfmt "%a %T" > nt.fna.taxidmapping`

Got these 2 from: [https://github.com/martin-steinegger/conterminator](https://github.com/steineggerlab/conterminator?tab=readme-ov-file#mapping-file)

Get the taxid in the database for an accesion id (there is a tab in the outfmt string)
`blastdbcmd -db /usr/local/BBDD/nt/nt -entry_batch accid_list.txt -outfmt "%a   %T" > acc2taxid.tsv`

*Be careful, this seems to retrieve other accids in the taxids corresponding to your queried accds*

## Loading a BLAST file into QIIME2

You can export a `search-results.qza` file from the `feature-classifier` QIIME2 plugin functions `blast` or `classify-consensus-blast` (both have a `--o-search-results` option) with:

`qiime tools export --input-path search_results.qza --output-path search_results.qza_exported`

Should return:

`Exported search_results.qza as BLAST6DirectoryFormat to directory search_results.qza_exported`

Within  `search_results.qza_exported` there will be a `blast6.tsv` file in the default `-outfmt 6` format. You can filter and modify it, for example, keep all the hits with the maximum bitscore:

`awk '{if($1 != asv) {asv=$1; maxbitscore=$12; print $0} else if($1==asv && $12==maxbitscore) { print $0 }}' search_results.qza_exported/blast6.tsv > filtered_blast6.tsv`

and load into QIIME2 again (without adding or deleting columns) using:

`qiime tools import --type FeatureData[BLAST6] --input-path filtered_blast6.tsv --output-path filtered_search_results.qza`

Inside QIIME2 again you can use `qiime feature-classifier find-consensus-annotation` to take the Lowest Common Ancestor (LCA) among the hits in the BLAST file for each feature:

`qiime feature-classifier find-consensus-annotation --i-search-results filtered_search_results.qza --i-reference-taxonomy tax_ref.qza --o-consensus-taxonomy topbitscore_taxonomy.qza`

The taxonomic assignment `topbitscore_taxonomy.qza` can be used with other QIIME2 functions such as `qiime taxa barplot`. This give you more freedom to filter the BLAST results.

Note that if you attempt to load a BLAST search from the commandline BLASTn, the output does not include sequences that have no hits, as opposed to the exported `search_results.qza`, where the features without a hit passing the thresholds will be in the `blast6.tsv` file in the format of `feature3` and `feature4` :

```
feature1	MATCHEDSEQ1	100.0	200.0	0.0	0.0	1.0	200.0	329.0	528.0	7.4e-104	370.0
feature2	MATCHEDSEQ2	100.0	200.0	0.0	0.0	1.0	200.0	329.0	528.0	7.4e-104	370.0
feature3	*	0.0	0.0	0.0	0.0	0.0	0.0	0.0	0.0	0.0	0.0
feature4	*	0.0	0.0	0.0	0.0	0.0	0.0	0.0	0.0	0.0	0.0
```

---

## Scripts

### [blast_taxid2lineage.sh](blast_taxid2lineage.sh): A script to add the taxonomic lineage string to the results of a BLAST search.

**Use:** Adds the levels (if they exist in the tax2lin file): superkingdom, phylum, class, order, family, genus, species to a BLAST output file.

**Requires:**

1. The (unzipped) output of [ncbitax2lin](https://github.com/zyxue/ncbitax2lin): `ncbi_lineages_[date].csv`.

2. An [`-outfmt 6`](https://www.metagenomics.wiki/tools/blast/blastn-output-format-6) BLAST output result file containing the TaxIDs corresponding to the matched sequence. By default it is assumed to be in the last field.

**Usage:**

`./blast_taxid2lineage.sh <taxid_to_lineage_file> <blast_result_file> <output_file> [taxid_field]`

This is an `awk` based script that loads the `ncbitax2lin` mapping file into an associative array with TaxIDs as keys and the taxonomic string as values. Then, the BLAST results file is read and when the TaxID in the current line exists in the TaxID-Taxonomy array, it adds the corresponding string in the last field of the BLAST result file.

---

## Links

Projects, blog posts that may be of interest. I mostly haven't use these but might try or copy something from them.

- [taxdb](https://github.com/lskatz/taxdb). Transform a taxonomy database into sqlite and manipulate it from there
- [(Another) LCA_BLAST_calculator](https://github.com/gjeunen/LCA_BLAST_calculator)
- [An old classic](https://www.polarmicrobes.org/somethings-should-be-easy-and-they-are/). You have to export the BLASTDB variable and have the taxonomy files in the same place to get taxonomy in the blast results.
- [BLASTGrabber: a bioinformatic tool for visualization, analysis and sequence selection of massive BLAST data](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-15-128)
- [On parallelization](https://www.biostars.org/p/487527/). (Its not as easy to do right as you may think)

