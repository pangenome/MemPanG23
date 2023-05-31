## Human Pangenome Reference Consortium scavenger hunt

If you have extra time while waiting for tools to run, you can look for the answers to these questions in the HPRC's [marker paper](https://www.nature.com/articles/s41586-023-05896-x). This activity is entirely optional.

### Questions

**How many individuals' genomes were assembled in this phase of the HPRC, and how many will ultimately be assembled?**
<details>
<summary>See answer</summary>

The current phase assembled 47 individuals, and the goal is assemble a total of 350

</details>

**What assembly algorithm was used to construct the HPRC assemblies?**
<details>
<summary>See answer</summary>

Trio-Hifiasm

</details>

**What platforms were the sequencing data generated from?**
<details>
<summary>See answer</summary>

1. HiFi
2. ONT ultralong
3. Bionano optical maps
4. Illumina (child and parents)
5. 10x Genomics
6. Hi-C (Dovetail Omni-C)

</details>

**How do the HPRC assemblies compare to the GRCh38 reference in terms of contiguity?**
<details>
<summary>See answer</summary>

Overall similar. Most of the assemblies are slightly less contiguous, but some are slightly more contiguous. However, this result does not factor in the scaffolding of GRCh38, which connects separated contigs that are believed to be on the same chromosome (by using a region of all `N` characters). With the scaffolding taken into account, GRCh38 would be almost as contiguous as the CHM13 assembly.

</details>


**Approximately what fraction of the bases in the assemblies did Flagger identify as dubious?**
<details>
<summary>See answer</summary>

Most are <1%

</details>


**How many misassemblies were corrected manually?**
<details>
<summary>See answer</summary>

3 haplotypes in 3 separate samples: HG01358, HG001123, and HG002

</details>

**How would you interpret the 4 different categories shown the colors in Figure 1j?**
<details>
<summary>See answer</summary>

These categories could have different interpretations depending on if they are real or artifactual. If they are real, the "Not covered" category indicates a deletion, the "2 Alignments" and ">2 Alignments" category indicate duplications, and the "Only 1 Alignment" category indicates no change in copy number relative to CHM13. All of these categories could also be artifactually produced by assembly errors.

</details>

**What accounts for the discrepancy in the size of the paternal male haplotype in Figure 1c?**
<details>
<summary>See answer</summary>

Paternal male chromosomes have a Y chromosome instead of an X, and the human Y chromosome is much smaller than the X chromosome

</details>
 

**How many genomes had observed gene copy number changes?**
<details>
<summary>See answer</summary>

3,210

</details>

**How concordant are the PGGB and Minigraph-Cactus pangenomes' predictions of the number of variant loci among the HPRC assemblies?**
<details>
<summary>See answer</summary>

Mostly concordant. PGGB identifies 21 million small variants to Minigraph-Cactus' 22 million. In contrast, PGGB identifies 73,000 structural variants, whereas Minigraph-Cactus identifies 73,000

</details>

**What tool was used to identify variant loci in the pangenome graph?**
<details>
<summary>See answer</summary>

`vg deconstruct`

</details>

**How does the performance of the pangenomic methods differ between the entire genome and the Genome in a Bottle's "Challenging Medically Relevant Gene" regions?**
<details>
<summary>See answer</summary>

The HPRC pangenome methods improve calling over alternatives in both benchmark sets, but the improvement is proportionally greater in the challenging medically relevant genes.

</details>

**In the structural variant genotyping experiments, why are the haplotypes removed from the pangenome before genotyping them?**
<details>
<summary>See answer</summary>

Leaving them in would "double dip" on the data: using it both to parameterize the model and the evaluate the performance. This artificially inflates performance by allowing the model to "memorize" the data. Leave-one-out analyses are a type of data splitting design that mitigate this effect.

</details>

**What accounts for the spike in indels of size ~300 base pairs in Figure 6e?**
<details>
<summary>See answer</summary>

This is the length of the Alu SINE transposon, which makes up >10% of the human genome

</details>

**How many novel structural variants were called by PanGenie using the HPRC data?**
<details>
<summary>See answer</summary>

Trick question: none. PanGenie genotypes known, previously observed SVs. It does not discover SVs de novo.

</details>

