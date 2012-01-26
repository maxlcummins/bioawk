About bioawk
------------

Bioawk is an extension to Brian Kernighan's awk_ that adds support for
several common biological data formats, including optionally gzip'ed
BED, GFF, SAM, VCF, FASTA/Q and TAB-delimited formats with the column names.

Bioawk adds a new `-c fmt` option that specifies the input format. The behavior
of bioawk will vary  depending  on  the value of fmt.

For the formats that awk recognizes specially named variables will be created. For
example for the supported sequence formats the *$name*, *$seq* and, if applicable
*$qual* variable names may be used to access the name, sequence and quality string of the
sequence record in each iteration. Here is an example of iterating over a fastq
file to print the sequences::

    awk -c fastq '{ print $seq }' test.fq

For known interval formats the columns can be accessed via
the variables called *$start*, *$end*, *$chrom* (etc). For example
to print the feature lenght of a file in BED format one could write::

    awk -c bed '{ print $end - $start }' test.bed
    
One important change (and innovation) over the original awk is that bioawk
will treat sequences that may span multiple lines as a single record.
The parsing, implemented in C, may be several orders of magnitude
faster than similar code programmed in interpreted languages: Perl, Python, Ruby.

When the format mode is `header` or `hdr`, bioawk parses named columns. It automatically adds
variables whose names are taken from the first line and values from the column index.
Special  characters are  converted  to a underscore.
       
Bioawk also adds a few built-in functions including, as of now, `and()`, `or()`, `xor()`,
and others (see comprehensive list below).

Detailed help is maintained in the bioawk manual page, to access it type::

    man ./awk.1

Usage Examples
--------------

#. Extract unmapped reads without header::

    awk -c sam 'and($flag,4)' aln.sam.gz

#. Extract mapped reads with header::

    awk -c sam -H '!and($flag,4)'

#. Reverse complement FASTA::

    awk -c fastx '{ print ">"$name;print revcomp($seq) }' seq.fa.gz

#. Create FASTA from SAM (uses revcomp if FLAG & 16)::

    samtools view aln.bam | \
        awk -c sam '{ s=$seq; if(and($flag, 16)) {s=revcomp($seq) } print ">"$qname"\n"s}'

#. Get the %GC from FASTA::

    awk -c fastx '{ print ">"$name; print gc($seq) }' seq.fa.gz

#. Get the mean Phred quality score from FASTQ::

    awk -c fastx '{ print ">"$name; print meanqual($qual) }' seq.fq.gz

#. Take column name from the first line (where "age" appears in the first line
   of input.txt)::

    awk -c header '{ print $age }' input.txt

Note that when "-c" is not specified and the new built-in functions are not
used, bioawk should behave exactly the same as the original BWK awk. At least
this is the intention. Bioawk also tries to minimize the modification to the
original code base such that improvements in the future versions of BWK awk
can be readily incorporated into bioawk (yes, Brian Kernighan is still
maintaining his code).

New Builtin Functions
---------------------

gc($seq)
    Returns the GC percentage of a sequence.
    
meanqual($seq)
    Returns the average quality of the fastq sequence.
    
reverse($seq)
    Returns the reverse of the sequence.
    
revcomp($seq)
    Returns the reverse complement of the sequence.
    
qualcount($qual, threshold)
    Returns the number of quality values above the threshold parameter.
    
trimq(qual, beg, end, param)
    Trims the quality string qual in the Sanger scale using Richard Mott's algorithm (used in Phred). The
    0-based beginning and ending positions are written back to beg and end, respectively. The  last  argument
    param is the single parameter used in the algorithm, which is optional and defaults 0.05.

and(x, y)
    bit AND operation (& in C)

or(x, y)
    bit OR operation (| in C)

xor(x, y)
    bit XOR operation (^ in C)
              
Recognized Formats
------------------

These formats may be passed as the -c flag:

bed
    1:chrom 2:start 3:end 4:name 5:score 6:strand 7:thickstart 8:thickend 9:rgb 10:blockcount 11:blocksizes 12:blockstarts 

sam
    1:qname 2:flag 3:rname 4:pos 5:mapq 6:cigar 7:rnext 8:pnext 9:tlen 10:seq 11:qual 

vcf
    1:chrom 2:pos 3:id 4:ref 5:alt 6:qual 7:filter 8:info 

gff
    1:seqname 2:source 3:feature 4:start 5:end 6:score 7:filter 8:strand 9:group 10:attribute 

fastx
    1:name 2:seq 3:qual

The `fastx` flag can handle both FASTA and FASTQ formats.

Adding New Functionality
------------------------

Follow these steps when adding a new function.

#. Add a function index (e.g. #define BIO_FFOO 102) in addon.h.
   The integer index must be larger than 14 in the current awk implementation
   (see also macros starting with "F" defined in awk.h).
#. Add the function name and the function index in the keywords array defined in lex.c.
   *Remember to keep the array sorted!*
#. Implement the actual function in bio_func().
 

Note
----

Bioawk may have the following limitations:

1. To parse FASTA and FASTQ formats, bioawk replaces the line reading module of
   awk, which also allows bioawk to seamlessly parse gzip'ed files. However,
   the new line reading code does not fully mimic the original code. It may
   fail in corner cases. Thus when "-c" is not specified, awk falls back to the
   original line reading code and does not support gzip'ed input.

2. When "-c" is in use, several strings allocated in the new line reading
   module are not freed in the end. These will be reported by valgrind as
   "still reachable". To some extend, these are not memory leaks.


.. _awk: http://www.cs.princeton.edu/~bwk/btl.mirror/
