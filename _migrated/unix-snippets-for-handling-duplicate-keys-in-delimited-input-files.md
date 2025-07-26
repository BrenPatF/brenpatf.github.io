---
layout: post
title: "Unix Snippets for Handling Duplicate Keys in Delimited Input Files"
date: 2014-11-30
migrated: true
categories: 
  - "unix"
tags: 
  - "awk"
  - "grep"
  - "unix"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

Recently I had a problem where input data files for an interface contained duplicate keys that caused the interface to fail. Sometimes the complete lines were duplicated, but in other cases only the key fields were duplicated with differing non-key attribute values. I wanted, firstly, to report on the problem lines, and, secondly, to clean up the files as far as possible. For lines that are fully duplicated, it's easy enough to deduplicate, and the following command does that, producing a sorted file with unique lines:

_sort -u input.dat > output.dat_

The unix _uniq_ comand leaves the lines in a file in the original order, but only deduplicates adjacent sets of lines.

It's more difficult when keys are duplicated but non-key fields differ and we do not know which line to retain. In this case, we wanted to just delete those lines and report them to the source system for correction.

I wrote some snippets of unix code to report on duplicate-key lines and clean up the files, using unix utilities such as sort, awk, grep. This article lists the code snippets with outputs for a test data file.

**Update 22 December 2014** I added a section 'Using awk Hash Variables for Group Counting'.

## Test Scenarios

The first test data file has two leading key fields, followed by four attribute fields, with pipe delimiters. There is one pair of complete duplicates, a set of three lines having the same key but varying attributes, and two unique lines. Because we will be using grep for pattern matching, an obvious potential bug would be to match strings that are not in the first two positions, so I have included one of the duplicate key pairs as fields 4 and 5 to trap that if it occurs.

### test\_dups\_in\_1\_2.dat with the 2 leading fields key

```
k91|k92|a1|k41|k42|a
k11|k12|a1|k41|k42|b
k41|k42|a4x|k41|k42|c
k21|k22|a2|k41|k42|d
k41|k42|a4|k41|k42|c
k41|k42|a4|k41|k42|c
k11|k12|a1|k41|k42|b

```

Here are the functions applied to the initial test set:

- List the distinct keys that have duplicates
- List all lines for duplicate keys
- Strip all lines with duplicate keys from the file, returning unsorted lines
- Strip the inconsistent duplicates from the file, returning unique sorted lines

The solutions rely on sorting the lines, taking advantage of the fact that the key fields are leading. If the key fields are not leading a little more work is required, and I repeated the fourth function for this case, after swapping fields 2 and 3, so that then key fields are 1 and 3.

- Strip the inconsistent duplicates from the file, returning unique sorted lines - key fields not leading

## List the distinct keys that have duplicates

### Unix Code

```
sort $INFILE1_2 | awk -F"|" ' $1$2 == last_key && !same_key {print $1"|"$2; same_key=1} $1$2 != last_key {same_key=0; last_key=$1$2}'
```

### Output

```
k11|k12
k41|k42
```

## List all lines for duplicate keys

### Unix Code

```
sort $INFILE1_2 | awk -F"|" ' 
$1$2 == last_key && !same_key {print last_line; same_key=1} 
$1$2 == last_key {print $0; same_key=1} 
$1$2 != last_key {same_key=0; last_key=$1$2; last_line=$0}'

```

### Output

```
k11|k12|a1|k41|k42|b
k11|k12|a1|k41|k42|b
k41|k42|a4|k41|k42|c
k41|k42|a4|k41|k42|c
k41|k42|a4x|k41|k42|c
```

I think this should work for any number of key fields as long as they are all at the start, if you replace $1$2 by the relevant field positions. If the key fields are not all at the start then you have to pre/post-process as I show later.

## Strip all lines with duplicate keys from the file, returning unsorted lines

This code creates a copy of the input file with the lines for the duplicate keys stripped out, using the previous awk code to generate a list that we then loop over doing a "grep -v" on the copied file.

### Unix Code

```
dup_list=`sort $INFILE1_2 | awk -F"|" '
$1$2 == last_key && !same_key {print $1"|"$2; same_key=1}
$1$2 != last_key {same_key=0; last_key=$1$2}'`
cp $INFILE1_2 $OUTFILE1_2; for i in $dup_list; do grep -v ^$i $OUTFILE1_2 > $WRKFILE1; mv $WRKFILE1 $OUTFILE1_2; done
cat $OUTFILE1_2
```

### Output

```
k91|k92|a1|k41|k42|a
k21|k22|a2|k41|k42|d
```

## Strip the inconsistent duplicates from the file, returning unique sorted lines

Here is a variation on the above, where we keep one line for duplicates where all the attributes are also duplicated, but drop the duplicates where the attributes vary and so we cannot clean them. We do this by apply the "-u" flag to the initial sort, and using that output also as the basis for the stripping part.

### Unix Code

```
dup_list=`sort -u $INFILE1_2 | awk -F"|" '
$1$2 == last_key && !same_key {print $1"|"$2; same_key=1}
$1$2 != last_key {same_key=0; last_key=$1$2}'`
sort -u $INFILE1_2 > $OUTFILE1_2; for i in $dup_list; do grep -v ^$i $OUTFILE1_2 > $WRKFILE1; mv $WRKFILE1 $OUTFILE1_2; done
cat $OUTFILE1_2
```

### Output

```
k11|k12|a1|k41|k42|b
k21|k22|a2|k41|k42|d
k91|k92|a1|k41|k42|a
```

## Test Scenario with Non-leading Key Fields

A simple awk command creates a a copy of the original test data file, but with fields 2 and 3 swapped.

### Unix Code

```
awk -F"|" '{print $1"|"$3"|"$2"|"$4"|"$5"|"$6}' $INFILE1_2 > $INFILE1_3
```

### test\_dups\_in\_1\_3.dat with fields 1 and 3 key

```
k91|a1|k92|k41|k42|a
k11|a1|k12|k41|k42|b
k41|a4x|k42|k41|k42|c
k21|a2|k22|k41|k42|d
k41|a4|k42|k41|k42|c
k41|a4|k42|k41|k42|c
k11|a1|k12|k41|k42|b
```

## Strip the inconsistent duplicates from the file, returning unique sorted lines - key fields not leading

This variation caters for non-leading key fields by first swapping the fields so that the key fields now lead, applying similar processing to the previous case, then swapping the fields back.

### Unix Code

```
awk -F"|" ' {print $1"|"$3"|"$2"|"$4"|"$5"|"$6}' $INFILE1_3 | sort -u > $WRKFILE1
dup_list=`awk -F"|" ' $1$2 == last_key && !same_key {print $1"|"$2; same_key=1} $1$2 != last_key {same_key=0; last_key=$1$2}' $WRKFILE1`
for i in $dup_list; do grep -v ^$i $WRKFILE1 > $WRKFILE2; mv $WRKFILE2 $WRKFILE1; done
awk -F"|" '  {print $1"|"$3"|"$2"|"$4"|"$5"|"$6}' $WRKFILE1 > $OUTFILE1_3
```

### Output

```
k11|a1|k12|k41|k42|b
k21|a2|k22|k41|k42|d
k91|a1|k92|k41|k42|a

```

Test script and output file are in [a GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/unix-snippets-for-handling-duplicate-keys-in-delimited-input-files).

## Using awk Hash Variables for Group Counting

Here is a script that counts the number of instances of each key, and also the number of distinct lines within each key. Two hash arrays are used to do the counting within awk; in the expected use case the input file may be quite large but the number of duplicated keys, which is the array cardinality, should be small. The script goes on to reconcile the number of lines in the cleaned output file with the input file and the counts in the duplicates file.

I added some extra lines to the input file and pass the file name as a parameter to the script.


<div class="scrollbox">
<pre>
if [ $# != 1 ] ; then
	echo Usage: $0 [Data file]
	exit 1
fi
echo Running $0 $1...
INFILE=$1
OUTFILE_DPK=$INFILE.dpk # duplicate keys with: number of lines; number of distinct lines
OUTFILE_CLN=$INFILE.cln # cleaned file - inconsistent duplicate keys removed, and rest unique

WRKFILE=/tmp/temp1.txt

# List the distinct keys that have duplicates with: number of lines; number of distinct lines

sort $INFILE | awk -F"|" '
$1$2 == last_key {num[$1"|"$2]++; if (last_line != $0) diff[$1"|"$2]++} 
$1$2 != last_key {last_key=$1$2}
{last_line=$0}
END {for (key in num) print key, ++num[key], ++diff[key]}' > $OUTFILE_DPK

# Strip the inconsistent duplicates from the file, returning sorted lines

sort -u $INFILE > $OUTFILE_CLN
dup_list=$(awk -F"|" '
$1$2 == last_key && !same_key {print $1"|"$2; same_key=1} 
$1$2 != last_key {same_key=0; last_key=$1$2}' $OUTFILE_CLN)
for i in $dup_list; do grep -v ^$i $OUTFILE_CLN > $WRKFILE; mv $WRKFILE $OUTFILE_CLN; done

function line_print {
	echo '***' # 's needed else * translated
	echo $1
}
function line_count {
	line_print "$1" # "s needed else first word only passed
	wc -l $2
	retval=$(wc -l $2|cut -d" " -f1)
}

line_print 'Line counts...'

line_count "Input data file..." $INFILE; n_inp=$retval
line_count "List of duplicate keys..." $OUTFILE_DPK; n_dup=$retval
line_count "Cleaned data file, i.e. unique records, and inconsistent duplicates removed..." $OUTFILE_CLN; n_cln=$retval

# awk sums the second column of duplicate keys file to get total lines associated
n_dup_lines=$(awk '{sum += $2} END {print sum}' $OUTFILE_DPK)

# grep counts the number of keys that have 1 as last character preceded by space, being the number of keys with only 1 distinct line
n_dup_keys_cons=$(grep -c " 1$" $OUTFILE_DPK)

line_print 'Input file...'
cat $INFILE

line_print 'Duplicated keys with number of lines and number of distinct lines within key...'
cat $OUTFILE_DPK

line_print 'Cleaned file...'
cat $OUTFILE_CLN

line_print 'Reconciliation...'

echo Number of lines in cleaned file, which is $n_cln, should equal the following computed value...
echo "(Input lines - duplicate key lines + consistent duplicate keys) = " $n_inp - $n_dup_lines + $n_dup_keys_cons = $((n_inp-n_dup_lines+n_dup_keys_cons))
</pre>
</div>

### Output from the script

<div class="scrollbox">
<pre>
Running ./Dup_Keys.ksh test_dups_in_1_2.dat...
***
Line counts...
***
Lines in input data file...
15 test_dups_in_1_2.dat
***
Lines in duplicate keys file...
3 test_dups_in_1_2.dat.dpk
***
Lines in cleaned data file (unique records, and inconsistent duplicates removed)...
4 test_dups_in_1_2.dat.cln
***
Input file...
k91|k92|a1|k41|k42|a
k11|k12|a1|k41|k42|b
k41|k42|a4x|k41|k42|c
k21|k22|a2|k41|k42|d
k41|k42|a4|k41|k42|c
k41|k42|a4|k41|k42|c
k41|k42|a4|k41|k42|c
k41|k42|a4|k41|k42|c
k41|k42|a4|k41|k42|c
k41|k42|a4|k41|k42|z
k41|k42|a4|k41|k42|c
k51|k52|a4|k41|k42|e
k51|k52|a4|k41|k42|e
k51|k52|a4|k41|k42|e
k11|k12|a1|k41|k42|b
***
Duplicated keys with number of lines and number of distinct lines within key...
k11|k12 2 1
k51|k52 3 1
k41|k42 8 3
***
Cleaned file...
k11|k12|a1|k41|k42|b
k21|k22|a2|k41|k42|d
k51|k52|a4|k41|k42|e
k91|k92|a1|k41|k42|a
***
Reconciliation...
Number of lines in cleaned file, which is 4, should equal the following computed value...
(Input lines - duplicate key lines + consistent duplicate keys) =  15 - 13 + 2 = 4
</pre>
</div>
