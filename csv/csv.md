# Powerful CSV processing with kdb+

Comma separated text files (CSV) are the most fundamental format for data processing. All programming languages and software that supports working with relational data, also provide some level of support for CSV files. You can persist and process data without installing a DBMS.

In this article I review available tools to process CSV files and then show what kdb+ contribute add to the basket.

## Tools
### Linux command line tools
Bash itself supports array so you can read a CSV line-by-line and store all fields in an array variable. You can use built-in string manipulation and integer calculations (even float calculation with e.g `bc -l`). The code will be lengthy and hard to maintain.

General text processing tools like `awk` scripts may result shorter and simpler code. Command like `cut`, `sort` or `paste` comes handy, you can specify the separator and **refer to fields by positions**.

Position-based reference breaks if new column is added ahead of our referred column or columns are shuffled e.g. to move related columns next to each other. The problem manifests silently, maybe your scripts will run fine, you just use a different column for your calculation. This can be embarrassing if you end user discovers the discrepancy.

Position based reference create a fragile code. Processing CSV in bash by these commands is great for prototyping and to do a quick analyses but you bump into the limits once your codebase starts increasing or you share scripts with other colleagues.

You can refer to a column by name. The column names are stored in the first row. Reference by name is sensitive to column renaming but this is regarded to be less risk than position based reference.

The huge benefit of Linux command line tools is that no installation is required. Your bash script will likely run on other's Linux system. Get familiar with Linux tools but refrain from using them in complex tasks.

### CSVKit

Python library CSVKit offers a more robust solution than native Linux commands. Most importantly CSVKit commands allows column name based reference. Also they are better than general purpose text tools because e.g. they treat first column differently. Linux command `sort` treats the first row as normal row and can place it in the middle of the output. However, `csvsort` leaves first row intact.

The naming is coherent, `csvcut`, `csvgrep` and `csvsort` replace traditional Linux commands `cut`, `grep` and `sort`.

Function `csvlook` displays a cvs file that is easier to visualize and understand. The columns are aligned under the header column name.

You probably use Linux commands head, tail, less/more and cat to take a quick look at the content of a text file. Unfortunately the output of these tools for CSV files is not appealing. The columns are not aligned and you will spend lot of time squinting a black and white screen figuring out to which column a given cell belongs. You may give up and import the data into excel or Google Sheet. However the file is on a remote machine so first you need to scp the file to your desktop. You can save time by using `csvlook`.

```bash
$ csvlook --max-rows 20 data.csv
```

Don't worry if your console is narrow, pipe the output to `less -S` and use arrow keys to move left and right.

Another useful extension of CSV tools is command `csvstat`. It analyzes the content of and displays statistics about the column like number of distinct values. Most importantly, it tries to infer types. If the column type is a number then it also returns max/min/mean/median/stddev of the values.

### xsv

Some CSVKit commands are slow because they load the entire file into the memory and creates an in-memory database. Rust developers reimplemented several traditional tools like `cat`, `ls`, `grep` and `find` and tools like `bat`, `exa`, `ripgrep` and `fd` were born. No wonder that they also created a tool for CSV processing and claimed their stake on outstanding speed.

Library xsv also supports selecting column, filtering, sorting and joining csv. An index can be added to CSV files that are frequently processed to speed up operations.


## Type Inference
CSV is a text file, each cell contains a string. Strings can be converted to a type based on its value. If all values of a column matches the pattern `YYYY.MM.DD` then we can conclude that the column stores date. But how shall we treat the literal 100000. Is it an integer, or a time 10am? Maybe the upstream process only supported digits hence the time separators are missing. If all values of the column matches `HHMMSS` string then we **may** conclude that the column stores time values. We can follow two approaches to make the decision.

First, we can be strict, i.e. we predefine the pattern any type needs to match. The patterns are not overlapping. If time is defined as `HH:MM:SS` then 100000 is an integer.

Second, we let overlapping patterns and in case of conflict we choose the type that has pattern with smaller domain. This approach prefers time over int for 100000 if time pattern also contains `HHMMSS`.

Library csvkit implements the first approach. TODO: how can the user add extra conversion?


## kdb+
Kdb+ natively supports exporting and importing CSV files. Table `t` can be saved by command

```
q) save `t.csv
```

If you would like to chose different name e.g. `output.csv` then

```
q) `output.csv 0:","0: t;
```

To import a CSV `data.csv`, you need to specify the column types and the separator. The following command assumes that the column names are in the first row.

```
q) ("JFDS* I";enlist",") 0:hsym `output.csv
```

The type mapping is available on the [kdb+ reference card](https://code.kx.com/q/ref/#datatypes), `J` stands for long integer, `D` stands for date, white space will ignore the column at that index.

Specifying types manually has high maintenance costs, is laborious for wide CSV files and is prone to error. Inserting a single new column can break the code.

Fortunately Kx open source libraries [csvutil](https://github.com/KxSystems/kdb/blob/master/utils/csvutil.q) and [csvguess](https://github.com/KxSystems/kdb/blob/master/utils/csvguess.q) offer a comfortable and robust solution.

Scripts `csvutil.q` contains function to load a CSV file, analyzes its values, infers types and return a kdb+ table.

```
q) t: .csv.read `data.csv
```

Scripts `csvguess.q` allows saving a meta information about the columns. Developers can review and adjust this information and use it in production to load CSV with proper type. The two scripts have different users. Data scientists prefer `csvutil.q` for their ad hoc analyses. When IT is setting up a kdb+ CSV feed then they use meta data export and import feature `csvguess.q`.

### Type conversion

Library `csvutil.q` supports both type conversions. The strict mode is implemented by `.csv.basicread`. Function `.csv.read` checks more pattern to infer types.

Function `.csv.read` is just a wrapper around `.csv.data` that accepts a filename and a meta data table. The meta data can be generated by `.csv.basicinfo` and `.csv.info` depending on the inference rule set we would like to employ.

The column meta data table is a bit similar to `csvstat` output. Each row belongs to a column and each field stores some useful information about the column, like name (`c`), inferred type (`t`), max width (`mw`), etc. Field `gr` short for granularity is particularly interesting as it gives hint how well the column values compresses and if it should be stored as enumeration (symbol in kdb+ parlance)rather than string.

You can control the number of lines to be examined for type inference by variable `READLINES`. The default value is 5555. The smaller this number the more chance of an inference rule to be accidental. For example in sample table column `fips` matches the patter `HMMSS` for the first 916 rows, so we could infer time as type. To disable partial file based type inference, just change `READLINES`.

```
q) .csv.READLINES: count read0 `data.csv
```


## kdb+-based one-liners

Let us wrap the q interpreter and `csvutil.q` into a simple bash function.

```bash
$ function qcsv { q -c 25 320 <<< 'system "l utils/csvutil.q";'"$1" }
```

The `-c 25 320` command line parameter modifies the default 25x80 console size to better display wide tables.

This wrapper can easily be used to achieve what csvlook, csvcut, csvgrep and csvsort implements. Function .csv.read creates a kdb+ table from a CSV file. All columns of a kdb+ must have a type, .csv.read applies automatic type conversion.

```bash
$ qcsv '20#.csv.read `data.csv'
```

In q the `#` operator takes a subset of the entity on the right, let it be a table, list or a dictionary. If you would like to help junior developers understand your code, then you can use the more explicit `sublist` keyword.

### Filtering, selecting columns, sorting
Once we have a kdb+ table, we can use the full power of q-sql to do any data manipulation. To select columns.

```bash
$ qcsv 'select nsn,item_name from .csv.read `data.csv'
```

If you are the type of person who switches off the light when leaving the room for a few minutes, then you probably have bad feeling for devoting resources to analyze columns, load into memory and then throw them away. Good news, we can do a shortcut.

In kdb+, function `.csv.infoonly` accepts a list of column to restrict its analyses. Function `.csv.read` calls `.csv.data` which requires a file name and an info table. Therefore to restrict column, you can do.

```
$ qcsv '.csv.data[`data.csv; .csv.infoonly[`data.csv; `nsn`item_name]]'
```

We can employ complex criteria to select rows.

```bash
$ qcsv 'select from .csv.read `data.csv where tem_name like "RIFLE*", fips > 32000'
```

We can use q keywords `xasc` and `xdesc` to mock `csvsort`.

```bash
$ qcsv '`fips xdesc .csv.read `data.csv'
```

### Exotic functions
q-sql is the superset of ANSI SQL. This means that with our one-liner `qcsv` we can express complex logic that ANSI SQL cannot handle. Furthermore, q-sql is the subset of q, we can employ all features of q to further massage a CSV file. These include the vector operations, the functional

#### Pivot
Pivoting a table is a frequently used function in data analyses.

```
$ qcsv 'system "l utils/pivot.q"; .pvt.pivotSingle select sum Population by Region, Country from cities'
```

#### Array columns
Nothing prevents you technically to put a list of values into a cell of a CSV. You just need to use a separator other than comma, e.g. whitespace or semicolon. Unlike ANSI SQL, kdb+ can handle array columns.

Kdb+ function `vs` (that abbreviates vector from string) splits a string by a separator string.

```
q) " " vs "10 3 0"
```

returns and array of strings. kdb+ supports functional programming so if you are given a list of strings then you can use the `each` operator

```
q) vs[" "] each ("10 3 4"; "0 0 7")
```

or more elegantly with `each both` (denoted by ') if the separator is a single character

```
q) " " vs' ("10 3 4"; "0 0 7")
```

It is not a problem if the list of string is given as a column of a table. Let us assume that column `a` contains whitespace separated integers. Function `.csv.read` returns string column that we can easily convert to an array column.

```
q) update "I"$" " vs' a from .csv.read `data.csv
```

the construct `"I"$` denotes integer casting.

Just to illustrate the power of the q language, let us assume that `data.csv` contains another array column called `idx` and contains indices. for each row we need to calculate the the values of array column a restricted to the indices specified by `idx`.

In kdb+ you can index a list the same way as you do in other programing language
```
q) l: 4 1 6 3
q) l[2]
6
```

kdb+ is a vector languages. Many operations accepts not only scalars but lists as well. Indexing is such operation.

```
q) l[2 1]
6 1
```

The square brackets are just syntactic sugar, you can also use the `@` operator
q) l @ 2 1
6 1

If we pass list of lists to both sides of the `@` operator then we need the `each both` construct again. Putting all together


```bash
$ qcsv $'update sum_a_of: sum each a@\'idx from
           update "I"$" " vs\' a, "I"$" " vs\' idx from .csv.read `data.csv'
```

The quotation marks need to be escaped and we need to use ANSI-C quoting, hence the `$` before the opening quotation mark.

#### Join
Joining two CSV files already supported by command `join` of Linux. Command `csvjoin` goes further and support all types of SQL joins inner, left, right and outer.

For time series there is another type of join that are frequently used. These is called asof and its generalization window join.

Let me demonstrate the usage of window join is a real life scenario. Our master process sends request to slave processes. Each requests results in multiple tasks. We store the start and end times of the requests and the start and end times of the tasks. We would like to see the ratio of time a slave devoted to the request. Due to network delay start time of a task happens after start time of a request. An example of the master's data is below.

|requestID|slaveID|start|end|
| --- | --- | --- | --- |
RQ1|SL1|12:31|12:52|
RQ2|SL2|12:31|12:50|
RQ3|SL1|12:54|12:59|
RQ4|SL3|12:51|13:00|
RQ5|SL1|13:10|13:13|

And and merged slaves data is

|slaveID|taskID|start|end|
| --- | --- | --- | --- |
|SL1|1|12:32|12:33|
|SL1|2|12:35|12:37|
|SL1|3|12:37|12:47|
|SL2|1|12:31|12:48|
|SL1|4|12:55|12:56|
|SL1|5|12:56|12:59|
|SL3|1|12:52|12:55|
|SL3|2|12:58|12:59|
|SL1|6|13:10|13:12|

Function `wj1` helps finding the tasks with given slaveID values that happened within a time window specified by the master table's start and end columns.

```
q) m: .csv.read `master.csv
q) s: .csv.read `slave.csv
q) wj1[(m.start;m.end); `slaveID`start; m; (`slaveID`start xasc s; (::; `taskID))]
requestID slaveID start end   taskID
------------------------------------
1         sl1     12:31 12:52 1 2 3h
2         sl2     12:31 12:50 ,1h
3         sl1     12:54 12:59 4 5h
4         sl3     12:51 13:00 1 2h
5         sl1     13:10 13:13 ,6h
```

To get the ratio, we need to work with elapsed times

```
q) select requestID, rate: elapsed % end-start from
     wj1[(m.start;m.end); `slaveID`start; m;
       (`slaveID`start xasc update elapsed: end-start from s; (sum; `elapsed))]
```

#### Date/time manipulation



## Performance

## Conclusion
