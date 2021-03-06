# Scrappie basecaller

Scrappie is a technology demonstrator for the Oxford Nanopore Research Algorithms group.
```
Ref   : GACACAGTGAGGCTGCGTCTC-AAAAAAAAAAAAAAAAAAAAAAAAATTGCCCCTTCTTAAGTTTGCATTTAGATCTCTT
Query : GACACAG-GAGGCTGCGTCTCAAAAAAAAAAAAAAAAAAAAAAAAAATTGCCCCTTCTTAAGCTT-CA--CAGA-CT-TT
```

[![Travis](https://img.shields.io/travis/nanoporetech/scrappie.svg)]()
[![Coverity Scan](https://img.shields.io/coverity/scan/12969.svg)]()

For a complete release history, see [RELEASES.md]

## Dependencies
* A good BLAS library + development headers including cblas.h.
* The HDF5 library and development headers (with multi-threading support).

On Debian based systems, the following packages are sufficient (tested Ubuntu 14.04 and 16.04)
* libcunit1
* libcunit1-dev
* libhdf5
* libhdf5-dev
* libopenblas-base
* libopenblas-dev

The Intel MKL may be used to provide the BLAS library.  The combination of the Intel `icc`
compiler and linking against the MKL can result in significant performance improvements, a
gain of 50% being observed on one machine.

On Mac _OSX_ systems, the _argp-standalone_ package is also required.  The *argp-standalone* package
can be installed using the *brew* package manager (http://brew.sh).
```bash
brew install argp-standalone
```

Scrappie makes use of the *OpenMP* extensions for multi-processing.  These are supported
by the system installed compiler on most modern Linux systems but requires a more modern version
of the *clang/llvm* compiler than that installed on Mac _OSX_ machines.  Support for *OpenMP* was
adding in *clang/llvm* in version 3.7 (see http://llvm.org or use *brew*).

## Compiling
```bash
mkdir build && cd build && cmake .. && make
```

## Running
```bash
#  Set some enviromental variables.
# Allow scrappie to use as many threads as the system will support
export OMP_NUM_THREADS=`nproc`
# Use openblas in single-threaded mode
export OPENBLAS_NUM_THREADS=1
# Call a folder of reads via events
scrappie events reads ... > basecalls.fa
# Call a folder of reads from raw signal
scrappie raw reads ... > basecalls.fa
# Call indivdual reads
scrappie raw reads/read1.fast5 reads/read2.fast5 > basecalls.fa
# Or using a strand list (skipping first line)
tail -n +2 strand_list.txt | sed 's:^:/path/to/reads/:' | xargs scrappie raw > basecalls.fa
#  Using Scrappie in single-threaded mode
find path/to/reads/ -name \*.fast5 | parallel -P ${OMP_NUM_THREADS} scrappie raw --threads 1 > basecalls.fa
```

## Commandline options
The commandline options accepted by Scrappie depend on whether it is being used to call
via events or from raw signal.
```
> scrappie help events
Usage: events [OPTION...] fast5 [fast5 ...]
Scrappie basecaller -- basecall via events

  -#, --threads=nreads       Number of reads to call in parallel
      --dump=filename        Dump annotated events to HDF5 file
      --dwell, --no-dwell    Perform dwell correction of homopolymer lengths
  -f, --format=format        Format to output reads (FASTA or SAM)
      --hdf5-chunk=size      Chunk size for HDF5 output
      --hdf5-compression=level   Gzip compression level for HDF5 output (0:off,
                             1: quickest, 9: best)
  -l, --limit=nreads         Maximum number of reads to call (0 is unlimited)
      --licence, --license   Print licensing information
      --local=penalty        Penalty for local basecalling
  -m, --min_prob=probability Minimum bound on probability of match
  -o, --output=filename      Write to file rather than stdout
  -p, --prefix=string        Prefix to append to name of each read
  -s, --skip=penalty         Penalty for skipping a base
      --segmentation=chunk:percentile
                             Chunk size and percentile for variance based
                             segmentation
      --slip, --no-slip      Use slipping
  -t, --trim=start:end       Number of events to trim, as start:end
  -y, --stay=penalty         Penalty for staying
  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version
```


```
> scrappie help raw
Usage: raw [OPTION...] fast5 [fast5 ...]
Scrappie basecaller -- basecall from raw signal

  -#, --threads=nreads       Number of reads to call in parallel
  -f, --format=format        Format to output reads (FASTA or SAM)
      --hdf5-chunk=size      Chunk size for HDF5 output
      --hdf5-compression=level   Gzip compression level for HDF5 output (0:off,
                             1: quickest, 9: best)
  -l, --limit=nreads         Maximum number of reads to call (0 is unlimited)
      --licence, --license   Print licensing information
      --local=penalty        Penalty for local basecalling
  -m, --min_prob=probability Minimum bound on probability of match
      --model=name           Raw model to use: "raw_r94", "rgr_r94",
                             "rgrgr_r94", "rgrgr_r95"
  -o, --output=filename      Write to file rather than stdout
  -p, --prefix=string        Prefix to append to name of each read
  -s, --skip=penalty         Penalty for skipping a base
      --segmentation=chunk:percentile
                             Chunk size and percentile for variance based
                             segmentation
      --slip, --no-slip      Use slipping
  -t, --trim=start:end       Number of samples to trim, as start:end
  -y, --stay=penalty         Penalty for staying
  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version
```

## Output formats
Scrappie current supports two ouput formats, FASTA and SAM.  The default format is currently FASTA;
SAM format output is enabled using the `--format SAM` commandline argument.

Scrappie can emit SAM "alignment" lines containing the sequences but no quality information.  No other fields, include a SAM header are emitted.  A CRAM or BAM file can be obtained using `samtools` (tested with version 1.4.1) as follows:

```bash
scrappie raw -f sam reads | samtools view -Sb - > output.bam
scrappie raw -f sam reads | samtools view -SC - > output.cram
```

### FASTA
When the output is set to FASTA (default) then some metadata is stored in the description
  * The sequence ID is the name of the file that was basecalled.
  * The *description* element of the FASTA header is a JSON string containing the following elements:
    *  Events
      * `normalised_score` Normalised score (total score / number of events or blocks).
      * `nevents` Number of events processed.
      * `sequence_length` Length of sequence called.
      * `events_per_base` Number of events per base called.
    *  Raw
      * `normalised_score` Normalised score (total score / number of events or blocks).
      * `nblock` Number of blocks processed.
      * `sequence_length` Length of sequence called.
      * `blocks_per_base` Mean number of blocks per base.
      * `nsample` Number of samples in read.
      * `trim` Interval of samples used (lower inclusive, upper exclusive).


## Gotya's and notes
* Model is hard-coded.  Generate new header files using
  * Events: `parse_events.py model.pkl > src/nanonet_events.h`
  * Raw: `parse_raw.py model.pkl > src/nanonet_raw.h`
* The normalised score (- total score / number of events) correlates well with read accuracy.
* Reads with unusual rate metrics (number of events or blocks / bases called) may be unreliable.
* Scrappie requires HDF5 library compiled with multi-threading support, see [HDF5 concurrent access](https://support.hdfgroup.org/HDF5/hdf5-quest.html#gconc).  If only single-threaded HDF5 library is available then single-threaded Scrappie can be built and parallelized with xargs -- see [Running](#Running) for details.
