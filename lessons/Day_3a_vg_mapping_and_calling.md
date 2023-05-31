# VG refresher

## What to do during your down time

If you have have extra down time while you are waiting for tools to complete running, we have an optional activity that you can use it on--just for fun. This HPRC "[scavenger hunt](https://github.com/pangenome/MemPanG23/blob/main/lessons/Day_3a_scavenger_hunt.md)" has questions whose answers you can find the HPRC's marker paper.

## Getting started

Since we have several VG versions installed, please execute the following to ensure you will run the correct VG version:

    alias vg='/usr/local/bin/vg.1.48.0'

Once again, we're going to start with some toy data in the `vg` respository, so make sure you still have it checked out (if you still have this from day 1, you can omit this step):

    cd ~
	git clone https://github.com/vgteam/vg.git

Now create a directory to work on for this tutorial:

	mkdir vg_refresher
	cd vg_refresher
	ln -s ~/vg/test/small


## Review: using `vg` to build and view graphs

**Exercise:** Repeat the `vg construct` procedure from day 1 to construct a graph using `small/x.fa` and `small/x.vcf.gz`. Call the graph `small.vg`.

<details>
<summary>See answer</summary>

    vg construct -r small/x.fa -v small/x.vcf.gz > small.vg
    
</details>


**Exercise:** Use `vg view` and `dot` to visualize the graph. Find nodes 56 and 57 in the visualization. **Do you notice anything strange about them?**

<details>
<summary>See answer</summary>

	vg view -d small.vg | dot -Tpdf -o small.pdf

These nodes are separated by an edge, but not by any variation. It seems like it would be possible to have a longer single node in place of these two nodes.

This is in fact a default behavior in `vg`, which you can also see in nodes 101, 123, and 148. 
</details>

Let's also revisit the [GFA format](https://github.com/GFA-spec/GFA-spec/blob/master/GFA1.md), the text-based interchange format for graphs (similar to FASTA for sequences). Make a GFA for this graph.

	# the -f flag indicates converting to GFA
	vg convert -f small.vg > small.gfa

Since GFA is an interchange format for graphs, you can construct a graph by ingesting a GFA. 

	# -g indicates that the input is GFA, -p produces a .vg graph
	vg convert -g -p small.gfa > small.copy.vg
	
**Exercise:** Convert `small.copy.vg` back into GFA format and confirm that the graphs are identical using `diff`. **Do we get identical graphs?**

<details>
<summary>See answer</summary>

The graphs are in fact identical, but you may still get a non-trivial `diff`. The reason is that the order of lines in a GFA is not fixed by the specification. Try using `sort` on both GFAs to put them in the same order and then comparing with `diff` again.

</details>


# Indexing and mapping in `vg`

## `vg` mapping preliminaries

Although `vg` contains a number of tools for working with pangenome graphs, it is best-known for read mapping. This is ultimately what many of its users are interested in `vg` for. In fact, `vg` contains three mature short read mapping tools:

1. [`vg map`](https://www.nature.com/articles/nbt.4227): the original, highly accurate mapping algorithm
2. [`vg giraffe`](https://www.science.org/doi/10.1126/science.abg8871): the much faster and still accurate haplotype-based mapping algorithm
3. [`vg mpmap`](https://www.nature.com/articles/s41592-022-01731-9): the splice-aware RNA-seq mapping algorithm

## Indexing a graph for mapping with `vg` (the hard way)

Like many other read mapping tools, each of these mapping tools requires specialized index data structures. Accordingly, the first step to mapping reads is to build and index a graph. We will start by indexing a graph for `vg map`, which has the simplest indexing pipeline of the three mapping tools.

We will start with some simple test data that is included in the VG source repository. Start by making a directory for us to work in today and then link the data into it.

    cd ~
    mkdir vg_mapping_exercises
    cd vg_mapping_exercises
    ln -s ~/vg/test/1mb1kgp
    ln -s ~/vg/test/tiny

### A `vg map` indexing pipeline

We begin the indexing pipeline by constructing a graph.

**Excercise:** Construct a graph using `vg construct` on reference sequence `1mb1kgp/z.fa` and VCF `1mb1kgp/z.vcf.gz`. Name the resulting file `1mb1kgp.vg`. 

<details>
<summary>See answer</summary>
You can use the following command:
    
    vg construct -r 1mb1kgp/z.fa -v 1mb1kgp/z.vcf.gz > 1mb1kgp.vg
    
</details>

The graph we've just constructed is a mutable data structure---it allows for changes to its structure and contents. This is important for the `vg construct` algorithm, which builds the graph incrementally as it processes the input data. However, there are performance costs to maintaining mutability. A static graph can support a greater range of queries at faster speeds. When we are mapping reads, we will no longer need the graph to be mutable, so we can improve performance by converting to a static graph, which we call "XG format".

    # -x indicates that the output is XG format
    vg convert -x 1mb1kgp.vg > 1mb1kgp.xg

Next we want to index the graph for fast exact match queries. To do that `vg map` uses the [GCSA2 index](https://epubs.siam.org/doi/abs/10.1137/1.9781611974768.2), which is a graph-specific generalization of the better known FM-index (used in many widely-used mapping tools, such as `bwa mem` and `bowtie2`). The risk with a GCSA2 index is that its size can grow exponentially with the size of the graph, and in fact we entirely expect it do so on genome-scale graphs. To prevent this from happening, we need to simplify the complicated regions of the graph by removing some nodes and edges using the `vg prune` subcommand.

    vg prune 1mb1kgp.vg > 1mb1kgp.pruned.vg

**Excercise:** Try using `1mb1kgp.xg` as input instead. What happens and why?

<details>
<summary>See answer</summary>
	
You should see an error about being unable to load a mutable graph. The `vg prune` algorithm will modify the graph by removing part of it, so the graph must be mutable. Recall that the XG is a static format.   

</details>

**Excercise:** Confirm that the nodes in `1mb1kgp.pruned.vg` have matching node IDs to the `1mb1kgp.vg`.

<details>
<summary>See answer</summary>
	
This could be accomplished in multiple ways. One easy way is to match the `S` lines from their respective GFAs.
    
    # grep out the sequence (i.e. node) lines and order them consistently
    vg convert -f 1mb1kgp.vg | grep "^S" | sort > 1mb1kgp.nodes.txt
    vg convert -f 1mb1kgp.pruned.vg | grep "^S" | sort > 1mb1kgp.nodes.pruned.txt
    
    # count the nodes in 1mb1kgp.pruned.vg that have no pair in 1mb1kgp.vg (should be 0)
    comm -23 1mb1kgp.nodes.pruned.txt 1mb1kgp.nodes.txt | wc -l
    
</details>

The fact that these graphs have the same node IDs is a subtle but key point. Different methods of processing graphs can produce different node IDs. To collate results across steps in a pipeline, we need matching node IDs---for example, between graph mappings and the graph. This is a frequent source of bugs in pangenomics pipelines. 

We are now ready to construct the GCSA index.

    vg index -g 1mb1kgp.gcsa -t 16 1mb1kgp.pruned.vg

**Important note:** The `-t` parameter in the command above sets the thread usage (to 16 threads). In most `vg` commands, the default number of threads is the number of cores on the machine, and `vg` will happily keep all of these cores busy. If you are working in a shared compute environment, you should usually use `-t` to be considerate of other users.

You should now see two files: `1mb1kgp.gcsa` and `1mb1kgp.gcsa.lcp`. These should be thought of as a single index spread out over two files. You need both of them for mapping.

## Mapping with `vg map`

Now we are finally ready to map reads.

    # -x and -g supply the XG and GCSA, -f is for FASTQ input, and the second 
    # -f indicates that these are paired end FASTQs
    vg map -x 1mb1kgp.xg -g 1mb1kgp.gcsa -f /opt/vg_data/1mb1kgp/1mb1kgp_1.fq -f /opt/vg_data/1mb1kgp/1mb1kgp_2.fq -t 16 > 1mb1kgp.mapped.gam

The default output format is GAM, which is based on Google's [Protobuf](https://protobuf.dev/) serialization method. To view what's in one of these alignments, we can convert it into a text-based [JSON](https://en.wikipedia.org/wiki/JSON) record.

    # -a indicates alignment input, -j is JSON output
    vg view -a -j 1mb1kgp.mapped.gam | head -n 1 

The output is a bit busy, though. Let's use the `jq` tool to format it nicely.

    vg view -a -j 1mb1kgp.mapped.gam | head -n 1 | jq
    
**What is the name of the field that in the GAM that expresses the alignment of the read to the graph?**

<details>
<summary>See answer</summary>
	
The alignment is encoded in the `path` field. The walk through the graph is indicated by a list of `mapping` records, which each have a `position` that includes the node ID. The mappings also have a list of `edit`s that expression the alignment of the read sequence to the node sequence.  

</details>

Another common format that used to express graph alignments is [GAF](https://github.com/lh3/gfatools/blob/master/doc/rGFA.md#the-graph-alignment-format-gaf). We can use `vg convert` to convert between the formats.

    vg convert -G 1mb1kgp.mapped.gam 1mb1kgp.xg > 1mb1kgp.mapped.gaf
    
GAF is a text-based, tab-separated format. Look at a few lines.

    head 1mb1kgp.mapped.gaf | less -S

**How does GAF show the alignment's walk through the graph?**

<details>
<summary>See answer</summary>
The walk is expressed in column 6 by a series of node IDs with arrows to indicate their orientation.  
</details>
    
    
**Exercise:** Look at the output to confirm that the first alignment has the same walk through the graph in both GAF and GAM format.

## Indexing a graph for mapping with `vg` (the easy way)

We have seen that the indexing pipeline for `vg map` requires a number of steps to preprocess the data and construct indexes. These steps are spread out across multiple subcommands, each of which handles one or a few indexes. This is complicated, and it gets worse. Both `vg giraffe` and `vg mpmap` have more complicated indexing pipelines.

The `vg autoindex` subcommand is designed to alleviate this complexity by focusing on the mapping tool you want to use, rather than the indexes you need to create. The inputs are all common interchange formats like FASTA, VCF, and GFA. Power users might still find need for the individual indexing subcommands, but common use cases should be handled in a single command.

Let's re-do the indexing pipeline from the previous section in a single command.
    
    # -p indicates the prefix for the output indexes
    # -w indicates that we are indexing for the "map" workflow
    vg autoindex -r 1mb1kgp/z.fa -v 1mb1kgp/z.vcf.gz -p 1mb1kgp_again -w map
    
See what got created:

    ls 1mb1kgp_again*
    
**Is there anything that you expected to see but do not?**

<details>
<summary>See answer</summary>
	
You may have expected to see a `.vg` like the ones we made using `vg construct` and `vg prune`. However, these files are not needed for mapping with `vg map`. They are constructed as temporary files and then discarded after being converted to more efficient indexes.
	
</details>
    
**Exercise:** Look at the logging output (lines starting with `[IndexRegistry]`) and try to correspond the steps `vg autoindex` takes to the ones we performed in the previous exercises. 


    
We will soon shift focus to the `vg giraffe` mapping tool, so let's make indexes for it. Start by looking at the help documentation.

	vg autoindex --help

**Exercise:** Using `vg autoindex`, create the graph and indexes for `vg giraffe`, using `tiny/tiny.fa` and `tiny/tiny.vcf.gz`. Give the indexes the prefix `giraffe_indexes`.
	
**What indexes did `vg autoindex` create?**

<details>
<summary>See answer</summary>

You should see `giraffe_indexes.dist`, `giraffe_indexes.giraffe.gbz`, and `giraffe_indexes.min`. If you don't see these indexes, did you request the correct workflow with the `-w` parameter?

</details>


# Calling structural variants using `vg`


## Background
    
One of the greatest promises for pangenome graphs is in characterizing structural variants (SVs). Methods based on linear references must contend with the fact that SVs often lead to poor read mapping. In contrast, any SV that is present in the pangenome graph is easy to map to. This makes pangenome graphs very useful tools for SV characterization using short reads.

This use case has been a particular focus for the `vg giraffe` mapping tool. It has been designed so that its speed is comparable to linear reference mapping tools, provided that the graph is relatively simple. On graphs with more complex topologies, some of its specialized indexes can become inefficient or intractable.
    
## Constructing (simple-enough) graphs with structural variation

Presently, there are two types of data sources for SVs. The first are VCFs containing SV calls, often from long read sequencing panels. The second is from phased assemblies. We will discuss both options.

### VCF input
    
We have already seen how to construct graphs using VCF input.
    
**Exercise:** How would you construct indexes for `vg giraffe` using a reference sequence `Reference.fasta` and a SV call set `SVs.vcf.gz`?
    
<details>
<summary>See answer</summary>
	
The easiest method would use `vg autoindex`.

    vg autoindex -r Reference.fasta -v SVs.vcf.gz -w giraffe -p index
    
</details>
    
**Important considerations**

1. `vg` can only use SVs that are fully base-resolved. It will ignore variants that have approximate breakpoints.
2. For `vg giraffe` in particular, the results will be best if the variants are phased. Much of its speed comes from focusing on the paths that haplotypes take through the graph, so it needs this haplotype information.
    
### Phased assembly input
    
Presently, the most capable method of producing pangenome graphs from phased assemblies that are both a) fully base-resolved, and b) simple enough for `vg giraffe` is the [Cactus-Minigraph](https://github.com/ComparativeGenomicsToolkit/cactus/blob/master/doc/pangenome.md) pipeline. It incorporates components from the [`cactus`](https://github.com/ComparativeGenomicsToolkit/cactus) whole genome aligner and the [`minigraph`](https://github.com/lh3/minigraph) pangenome graph constructor, which does not produce fully detailed graphs on its own.

Minigraph-Cactus is shipped along with `cactus`, which coordinates among a large number of sub-tools. For this reason, `cactus` is typically run out of a Python `virtualenv` that has been set up to ensure that all of the sub-tools are on the Unix paths that they need to be (see [here](https://github.com/ComparativeGenomicsToolkit/cactus/blob/v2.5.1/BIN-INSTALL.md) for instructions on how to set up this `virtualenv`). Accordingly, we start by activating the `virtualenv`, which has already been set up on your server.

    source /opt/cactus-bin-v2.5.2/cactus_env/bin/activate


Now let's build a pangenome graph. For demonstration, we will use a small set of haplotypes from the human MHC region---an immune-related region with extremely high levels of genetic polymorphism. Let's take a look at them:

    ls -l /opt/vg_data/mhc_haplotypes/

The primary command line input to Cactus-Minigraph is called the "seqFile", and it is essentially just a list of the sequences you are including. The format is:

    seqName1    /location/of/seqName1
    seqName2    /location/of/seqName2
    ...

The locations in the second column can be local filepaths, URLs, or addresses in AWS s3 storage.

**Exercise:** Construct a seqFile for the MHC haplotypes and call it `seqFile.txt`.

<details>
<summary>See answer</summary>
The contents of the seqFile should be:
    
    MHC-GRCh38	/opt/vg_data/mhc_haplotypes/MHC-00GRCh38.fa
    MHC-CHM13	/opt/vg_data/mhc_haplotypes/MHC-CHM13.0.fa
    MHC-HG002.1	/opt/vg_data/mhc_haplotypes/MHC-HG002.1.fa
    MHC-HG002.2	/opt/vg_data/mhc_haplotypes/MHC-HG002.2.fa
    MHC-HG00438.1	/opt/vg_data/mhc_haplotypes/MHC-HG00438.1.fa
    MHC-HG00438.2	/opt/vg_data/mhc_haplotypes/MHC-HG00438.2.fa
    MHC-HG005.1	/opt/vg_data/mhc_haplotypes/MHC-HG005.1.fa
    MHC-HG005.2	/opt/vg_data/mhc_haplotypes/MHC-HG005.2.fa
    
</details>

One drawback of Cactus-Minigraph is that it is reference-based. The graph starts out with one sequence, and then adds new sequences onto it. This allows it to use synteny as a hint for sequence orthology, which is a large part of why Minigraph-Cactus produces simple graphs. For our example, we will use the T2T haplotype of CHM13 (called `MHC-CHM13` in this data set) as the reference.

We are now ready to execute Cactus-Minigraph.

    cactus-pangenome ./working_dir seqFile.txt --outDir mhc_pangenome --outName mhc --reference MHC-CHM13 --vcf --giraffe --gfa --gbz

This may take several minutes to run. While you are waiting, we will note a few things about Cactus-Minigraph and this command line invocation.

1. Cactus is built on top of the workflow framework [Toil](https://toil.ucsc-cgl.org/). Toil provides it with methods to run on both clouds and clusters, although the [complexity](https://github.com/ComparativeGenomicsToolkit/cactus/blob/master/doc/running-in-aws.md) of [running](https://github.com/ComparativeGenomicsToolkit/cactus/blob/master/doc/pangenome.md#hprc-graph) it increases. Our examples will all run locally. Running locally is sufficient for small inputs, or for larger inputs in a time frame of a week or two.
2. The arguments `--vcf --giraffe --gfa --gbz` each add additional outputs to the pipeline. As you may guess, these can be used to provide the indexing input for `vg giraffe`. In fact, Cactus-Minigraph will run `vg autoindex` in the background to make these indexes.
3. You will notice that the `vg giraffe` indexes' file names include `d2`. This refers to the fact that these indexes exclude non-reference nodes that are present on only 1 haplotype (i.e. do not have "depth 2"). In the current iteration of the pipeline, we have found that this improves mapping performance. You can use the argument `--giraffe full` to keep all sequences and `--giraffe clip` to remove all rare nodes below 10% frequency.

_Aside: if you are still waiting for Minigraph-Cactus to complete, you can continue to the next section and return to the following exercise when it finishes. The next section does not depend on the output of this one._

**Exercise:** Once Minigraph-Cactus is done building the pangenome, use `Bandage` to visualize the graph.
    
<details>
<summary>See answer</summary>
Bandage takes GFA as input, so first we need to locate the GFA, which turns out to be gzipped by default.
    
    gunzip mhc_pangenome/mhc.gfa.gz
    Bandage load mhc_pangenome/mhc.gfa
    
</details>

## Mapping to a pangenome with `vg giraffe`

We are now going to shift focus to a _Drosophila melanogaster_ pangenome graph that was also constructed with Minigraph-Cactus. The _D. melanogaster_ genome is small for a multicellular eukaryote (~180 Mbp), so we can work with the entire genome and real sequencing data and still have practically-sized exercises.

Find the indexes in `/opt/vg_data/drosophila/`. What is there? 

**Exercise:** Look at the `vg giraffe --help` documentation to see which arguments to supply the indexes to. Then map the paired reads in `/opt/vg_data/drosophila/SRR835025_chr2_1.fastq.gz` and `/opt/vg_data/drosophila/SRR835025_chr2_2.fastq.gz`. Save the result in GAM format as `SRR835025_chr2.gam`. (Tip: `vg` can read gzipped FASTQ files directly.)

<details>
<summary>See answer</summary>
    
    vg giraffe -Z /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.gbz -d /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.dist -m /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.min -f /opt/vg_data/drosophila/SRR835025_chr2_1.fastq.gz -f /opt/vg_data/drosophila/SRR835025_chr2_2.fastq.gz > SRR835025_chr2.gam
    
</details>

The mapping may take several minutes. This is a real 30x Illumina data set taken from SRA, although (as the name suggests) it has been subsetted to the reads that map to chromosome 2. 

## Calling structural variants with `vg call`

`vg call` is the subcommand in `vg` that genotypes variants from mapped reads. However, it does not operate directly on the GAM file. First, we must preprocess the mappings into coverage depth for each of the nodes in a format that is called a "pack".

	# -x: graph
	# -g: GAM (-a to use GAF)
	# -Q: minimum mapping quality to include an alignment
	# -t: number of threads
	# -o: output name
	vg pack -x /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.gbz -g SRR835025_chr2.gam -Q 5 -t 16 -o SRR835025_chr2.pack

**Why do you think we exclude alignments with mapping quality < 5?**

<details>
<summary>See answer</summary>
	
For these reads, we believe that there is at least a ~30% chance that they are mapped to the wrong location. Including these alignments can introduce noise that makes variant calling less accurate. The developers consider this mapping quality filter a best practice.

</details>

Now we can use the pack to call variants.

	vg call /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.gbz -k SRR835025_chr2.pack -t 16 > SRR835025_chr2.vcf

Open the VCF file with a text viewer. **Roughly speaking, what proportion of the called variants are SVs?**

<details>
<summary>See answer</summary>
	
Very few of the variants are SVs. `vg call` calls variants at all "snarls" in the pangenome graph: bubble-like structures that usually indicate variation. There are many more point variants than SVs in most populations (even though SVs affect a large amount of total sequence). Accordingly, most of the of the called variants are point variants. To restrict to SVs, you may want to filter downstream using bcftools. 

</details>


**What is given as the variant ID (column 3)?**

<details>
<summary>See answer</summary>
	
It is the node IDs of the snarl boundaries.

</details>

Look for homozygous reference allele calls (indicated by a `0/0` genotype) in the VCF. **Roughly speaking, what proportion of the variants are homozygous reference?** 

<details>
<summary>See answer</summary>
	
There are none. By default, `vg call` excludes homozygous reference calls. However, you may want to include them in the VCF because, for example, you are combining individual VCFs into a larger panel VCF. 

**Exercise:** Find the option that includes homozygous reference calls using `vg call --help`.

<details>
<summary>See answer</summary>
	
The option is `-a`.

</details>
</details>

### Calling novel variants and fine-tuning SVs

The `vg call` algorithm calls variants only at the bubble-like "snarl" motifs in the graph. As a consequence, it can't on its own call any novel variants (those that are present in the sample but not the graph). To call novel variants, you have to add them into the graph as bubbles using the `vg augment` subcommand. This can also help "fine tune" SV sequences by, for example, correcting a breakpoint or adding a variant to an insertion. 

To start we make a mutable graph from our static one.

    vg convert /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.gbz > 16-fruitfly-mc-2022-05-26-d2.vg

And then we augment it.

    vg augment 16-fruitfly-mc-2022-05-26-d2.vg SRR835025_chr2.gam -A SRR835025_chr2.aug.gam -m 2 -q 10 > 16-fruitfly-mc-2022-05-26-d2.aug.vg

**The `-A` argument saves a new GAM file. Why is this necessary?**

<details>
<summary>See answer</summary>
	
Adding new bubbles into the graph requires us to break existing nodes and add new nodes. This requires the existing node IDs to change in the augmented graph. Accordingly, we need to update the paths of the alignments if we want them to match the graph that we are going to call variants with.
	
</details>

**The arguments `-m 2 -q 10` filter out bubbles unless they are supported by at least 2 reads with an average base quality of 10. Why might we do this?**

<details>
<summary>See answer</summary>
	
Putative variants that are only supported by a single read or by only bases of low quality are likely to be sequencing errors. Adding these candidate variants will expend unnecessary computational effort. In addition, it would increase the multiple testing burden, which can erode accuracy.
	
</details>

**Exercise:** Repeat the `vg call` pipeline with the augmented graph and augmented GAM. Name the resulting VCF `SRR835025_chr2.aug.vcf`.

<details>
<summary>See answer</summary>
        
    vg pack -x 16-fruitfly-mc-2022-05-26-d2.aug.vg -g SRR835025_chr2.aug.gam -Q 5 -t 16 -o SRR835025_chr2.aug.pack
    vg call 16-fruitfly-mc-2022-05-26-d2.aug.vg -k SRR835025_chr2.aug.pack -t 16 > SRR835025_chr2.aug.vcf
	
</details>

**Exercise:** Use a text viewer to look at the first handful of variants in `SRR835025_chr2.aug.vcf` and `SRR835025_chr2.vcf`. Try to locate the same variant in both VCFs. What do you notice about their IDs (column 3)?

<details>
<summary>See answer</summary>
	
Many of the variants will now have different IDs, because the node IDs in the graph have changed through the augmentation process.

</details>

**Exercise:** Find a variant that is present in the augmented VCF but not in the original VCF. (Hint: you can use `grep -v "#" filename.vcf | cut -f 2` to extract the positions of the variants).

# Calling small variants with `vg giraffe` and `DeepVariant`

Many state-of-the-art small variant calling algorithms are based on deep neural network technology, and [Google's `DeepVariant`](https://github.com/google/deepvariant) is among the most successful. It is fundamentally a linear reference method. `DeepVariant` uses the coordinates of a linear reference to make numerical vector inputs for the neural network. 

Nevertheless, `DeepVariant` can benefit from pangenome read mapping. The pangenome can alleviate alignment issues caused by reference bias, which improves the accuracy of small variant calling. The combination of `vg giraffe` and `DeepVariant` can produce some of the best-in-class small variant calls. 

To use `vg` mappings in a linear reference method, it is first necessary to convert them into linear mappings. This is accomplished with the `vg surject` subcommand, which realigns the reads to the path of the reference while preserving as much as possible of the graph alignment.

**Exercise:** We need to identify which paths in the graph are the reference. Use `vg paths --list` to compare the names of the graph's paths to `/opt/vg_data/drosophila/reference_paths.txt`.

<details>
<summary>See answer</summary>
You should see a very long list of paths if you execute 
    
    vg paths --list -x /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.gbz
    
However, the paths from `/opt/vg_data/drosophila/reference_paths.txt` are all present (try piping it through `grep dm6`). The other paths are regions of non-reference haplotypes added by Minigraph-Cactus during graph construction. We need the `reference_paths.txt` file to identify which of these paths we should be realigning to.
</details>

Now let's actually use `vg surject`.

    # -b for BAM output
    # -F is a list of the paths embedded in the graph that we project the alignments onto
    vg surject -x /opt/vg_data/drosophila/16-fruitfly-mc-2022-05-26-d2.gbz -t 16 -F /opt/vg_data/drosophila/reference_paths.txt -b SRR835025_chr2.gam > SRR835025_chr2.bam
    
Don't feel any need to wait for the realignment to complete, we won't be using it. We only include this command to introduce it to you. In truth, the best practices pipeline to run `DeepVariant` with `vg giraffe` is quite complicated. Fortunately, it has already been packaged in a [WDL (Workflow Description Language)](https://github.com/openwdl/wdl) script. 

Download the `vg` WDL repository.

    git clone https://github.com/vgteam/vg_wdl.git

WDL scripts can be run in cloud and cluster environments, but we will use the `miniwdl`, a WDL runner that uses local compute. We have already collected the inputs for the workflow. They are passed to `miniwdl` through an input file at `/opt/vg_data/giraffe_deepvariant/inputs.json`. 

**Exercise:** Look at the contents of `/opt/vg_data/giraffe_deepvariant/inputs.json`. Do you recognize the inputs for the `vg` subcommands we have used?

<details>
<summary>See answer</summary>
	
We previously saw `GBZ_FILE`, `MIN_FILE`, and `DIST_FILE` as the outputs of `vg autoindex`. We also saw `PATH_LIST_FILE` as an input for `vg surject` in the previous exercise.

</details>

Let's run the pipeline.

    miniwdl run vg_wdl/workflows/giraffe_and_deepvariant.wdl -i /opt/vg_data/giraffe_deepvariant/inputs.json
    
The WDL pipeline includes both mapping and variant calling, and you will be able to explore the outputs of every step. The outputs will be placed in a subdirectory of your current working directory when it completes. 

**Exercise:** Find the file that describes the pipeline's outputs, and use it to find the output VCF.

<details>
<summary>See answer</summary>
	
The output directory should be named something like `(current date and time)_GiraffeDeepVariant`. It contains a file `outputs.json` which gives the path to the VCF under `GiraffeDeepVariant.output_vcf`.

</details>

**Open-ended final exercise:** What can you infer about the sample's blood type based on the results in this VCF?
