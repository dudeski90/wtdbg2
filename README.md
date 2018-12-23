## <a name="start"></a>Getting Started
```sh
git clone https://github.com/ruanjue/wtdbg2
cd wtdbg2 && make
# assemble long reads
./wtdbg2 -t 16 -i reads.fa.gz -fo prefix -L 5000
# derive consensus
./wtpoa-cns -t 16 -i prefix.ctg.lay -fo prefix.ctg.lay.fa
# polish consensus, not necessary if you want to polish the assemblies using other tools
minimap2 -t 16 -x map-pb -a prefix.ctg.lay.fa reads.fa.gz | samtools view -Sb - >prefix.ctg.lay.map.bam
samtools sort prefix.ctg.lay.map.bam prefix.ctg.lay.map.srt
samtools view prefix.ctg.lay.map.srt.bam | ./wtpoa-cns -t 16 -d prefix.ctg.lay.fa -i - -fo prefix.ctg.lay.2nd.fa
```

## <a name="intro"></a>Introduction

Wtdbg2 is a *de novo* sequence assembler for long noisy reads produced by
PacBio or Oxford Nanopore Technologies (ONT). It assembles raw reads without
error correction and then builds the consensus from intermediate assembly
output. Wtdbg2 is able to assemble the human and even the 32Gb
[Axolotl][Axolotl] genome at a speed tens of times faster than [CANU][canu] and
[FALCON][falcon] while producing contigs of comparable base accuracy.

During assembly, wtdbg2 chops reads into 1024bp segments, merges similar
segments into a vertex and connects vertices based on the segment adjacency on
reads. The resulting graph is called fuzzy Bruijn graph (FBG). It is akin to De
Bruijn graph but permits mismatches/gaps and keeps read paths when collapsing
k-mers. The use of FBG distinguishes wtdbg2 from the majority of long-read
assemblers.

## <a name="install"></a>Installation

Wtdbg2 only works on 64-bit Linux. To compile, please type `make` in the source
code directory. You can then copy `wtdbg2` and `wtpoa-cns` to your `PATH`.

Wtdbg2 also comes with an approxmimate read mapper `kbm`, a faster but less
accurate consesus tool `wtdbg-cns` and many auxiliary scripts in the `scripts`
directory.

## <a name="use"></a>Usage

Wtdbg2 has two key components: an assembler **wtdg2** and a consenser
**wtpoa-cns**. Executable **wtdbg2** assembles raw reads and generates the
contig layout and edge sequences in a file "*prefix*.ctg.lay". Executable
**wtpoa-cns** takes this file as input and produces the final consensus in
FASTA. A typical workflow looks like this:
```sh
./wtdbg2 -t 16 -i reads.fa.gz -fo prefix
./wtpoa-cns -t 16 -i prefix.ctg.lay -fo prefix.ctg.lay.fa
```
where `-t` specifies the number of CPU cores (`-t 0` to use all processors).
When the default doesn't work well, you may need to apply more options briefly
explained as follows.

Wtdbg2 combines normal k-mers and homopolymer-compressed (HPC) k-mers to find
read overlaps. Option `-k` specifies the length of normal k-mers, while `-p`
specifies the length of HPC k-mers. By default, wtdbg2 samples a fourth of all
k-mers by their hashcodes. For data of relatively low coverage, you may
increase this sampling rate by reducing `-S`. This will greatly increase the
peak memory as a cost, though. Option `-e`, which defaults to 3, specifies the
minimum read coverage of an edge in the assembly graph. You may adjust this
option according to the overall sequencing depth, too. Option `-A` also helps
relatively low coverage data at the cost of performance. For PacBio data,
`-L5000` often leads to better assemblies emperically, so is recommended.
Please run `wtdbg2 --help` for a complete list of available options or consult
[README-ori.md](README-ori.md) for more help.

The following table shows various command lines and their resource usage for
the assembly step:

|Dataset                 |GSize |Cov     |Asm options        |CPU asm |CPU cns |Real tot|     RAM|
|:-----------------------|-----:|-------:|:------------------|-------:|-------:|-------:|-------:|
|[E. coli][pbcr]         |4.6Mb |PB x20  |-t32 -L5000        |     39s|  10m34s|     29s|    1.1G|
|[C. elegans][ce]        |100Mb |PB x80  |-t32 -L5000 -e4    |   1h00m|   5h06m|  16m16s|    9.5G|
|[Human NA12878][na12878]|3Gb   |ONT x36 |-t36 -p19 -AS2 -e2<br/>-L5000|822h28m|115h59m|27h42m|182.1G|
|[Human NA19240][na19240]|3Gb   |ONT x35 |-t32 -p19 -AS2 -e2 | 706h30m| 114h45m|  27h33m|  177.5G|
|[C. elegans][ce]        |100Mb |PB x80  |-t64 -L5000        |   1h46m|   5h27m|  14m17s|   10.1G|
|[Human CHM1][chm1]      |3Gb   |PB x60  |-t64 -L10000       | 186h15m| 131h52m|   7h41m|  265.2G|
|[Axolotl][axosra]       |32Gb  |PB x32  |-t96 -L5000 -AS2   |3076h13m|1180h03m| 88h01m| 1626.7G|

The timing was obtained on three local servers with different hardware
configurations. There are also run-to-run fluctuations. Exact timing on your
machines may differ.

## Limitations

* Read length
~~Wtdbg2 doesn't work with reads longer than 0x3FFFF (256kb)~~. Longer reads
  will be split into multiple parts.

* ~~ Wtdbg2 only works with up to 0x3FFFFFF (64 million) reads. If you have more
  reads, please filter short or low-quality reads first. ~~

## Getting Help

Please use the [GitHub's Issues page][issue] if you have questions. You may
also directly contact Jue Ruan at ruanjue@gmail.com.

[miniasm]: https://github.com/lh3/miniasm
[canu]: https://github.com/marbl/canu
[falcon]: https://github.com/PacificBiosciences/FALCON
[Axolotl]: https://www.nature.com/articles/nature25458
[chm1]: https://trace.ncbi.nlm.nih.gov/Traces/sra/?study=SRP044331
[na12878]: https://github.com/nanopore-wgs-consortium/NA12878/blob/master/rel5.md
[na19240]: https://www.ebi.ac.uk/ena/data/view/PRJEB26791
[pbcr]: http://www.cbcb.umd.edu/software/PBcR/data/selfSampleData.tar.gz
[ce]: https://github.com/PacificBiosciences/DevNet/wiki/C.-elegans-data-set
[axosra]: https://www.ncbi.nlm.nih.gov/bioproject/?term=PRJNA378970
[issue]: https://github.com/ruanjue/wtdbg2/issues
