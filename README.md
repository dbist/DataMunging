##### last edit: 04/20/16
##### Sandbox: 2.4.0 v3

# Data Munging with Hadoop by Ofer Mendelevitch and Casey Stella

#### As part of the US government’s efforts to make the healthcare system more transparent and accountable, the Centers for Medicare and Medicaid Services (CMS) released data surrounding payments rendered and procedures provided by physicians to Medicare beneficiaries. The data contains information such as
- The NPI (National Provider Identifier) of the physician rendering services
- The physician’s specialty
- The code of the procedure rendered
- The place of service
- Aggregate payment information for the rendering physician

The examples in this eBook are based on the CMS data from 2013
You can download the datasets from https://www.cms.gov/Research-Statistics-Data-and-Systems/Statistics-Trends-and-Reports/Medicare-Provider-Charge-Data/Physician-and-Other-Supplier2013.html

### files are in Excel format, convert to csv prior to loading into Hive

## Data Quality

#### Example: Pig UDF for HCPCS Codes
Procedures rendered in the course of treating a patient are coded, often using a coding standard called Current Procedural Terminology (CPT). The CMS encodes procedures in a generalization of this scheme called HCPCS (Healthcare Common Procedure Coding System), which we use in our example.
Healthcare workers often enter these codes manually, so they are prone to errors, and data quality checks are required. A simple way to validate an HCPCS code is by using a regular expression. This is a simple example of a cell-based data quality rule.
Our example dataset has listed the HCPCS code of the procedure rendered to a Medicare beneficiary by every provider who submitted for reimbursement from the government. The following implements, using Pig, a simple validity check for HCPCS codes and generates a report of the erroneous codes and the count per code:

#### prepare tables
```
CREATE DATABASE medicare_part_b;
DROP TABLE IF EXISTS medicare_part_b.medicare_part_b_2013_text;
CREATE TABLE medicare_part_b.medicare_part_b_2013_text (
  npi string,
  nppes_provider_last_org_name string,
  nppes_provider_first_name string,
  nppes_provider_mi string,
  nppes_credentials string,
  nppes_provider_gender string,
  nppes_entity_code string,
  nppes_provider_street1 string,
  nppes_provider_street2 string,
  nppes_provider_city string,
  nppes_provider_zip string,
  nppes_provider_state string,
  nppes_provider_country string,
  provider_type string,
  medicare_participation_indicator string,
  places_of_service string,
  hcpcs_code string,
  hcpcs_desc string,
  hcpcs_drug_indicator string,
  line_srvc_cnt int,
  bene_unique_cnt int,
  bene_day_srvc_cnt int,
  average_Medicare_all_owed_amt string,
  average_submitted_chrg_amt string,
  stdev_submitted_chrg_amt string,
  average_Medicare_payment_amt string,
  stdev_Medicare_payment_amt string)
COMMENT 'Data Munging Medicare Part B Dataset 2013'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\,'
STORED AS TEXTFILE;
```

#### upload sample data
```
sudo -u hdfs hdfs dfs -put Medicare_Provider_Util_Payment_PUF_a_CY2013.csv /tmp/
sudo -u hdfs hdfs dfs -mkdir /user/admin
sudo -u hdfs hdfs dfs -chown -R admin:hdfs /user/admin
sudo -u hdfs hdfs dfs -chown -R admin:hdfs /tmp/Med*

LOAD  DATA  INPATH  '/tmp/Medicare_Provider_Util_Payment_PUF_a_CY2013.csv'
OVERWRITE INTO TABLE medicare_part_b.medicare_part_b_2013_text;
```

#### Confirm the data is readable

```
SELECT * FROM medicare_part_b.medicare_part_b_2013_text LIMIT 100;
```	
```
100 rows selected (0.668 seconds)
0: jdbc:hive2://localhost:10000> SELECT count(*) FROM medicare_part_b.medicare_part_b_2013_text;
INFO  : Tez session hasn't been created yet. Opening session
INFO  : Dag name: SELECT count(*) FROM ...are_part_b_2013_text(Stage-1)
INFO  :

INFO  : Status: Running (Executing on YARN cluster with App id application_1461199791760_0001)

INFO  : Map 1: -/-	Reducer 2: 0/1
INFO  : Map 1: 0/7	Reducer 2: 0/1
INFO  : Map 1: 0/7	Reducer 2: 0/1
INFO  : Map 1: 0(+1)/7	Reducer 2: 0/1
INFO  : Map 1: 0(+2)/7	Reducer 2: 0/1
INFO  : Map 1: 0(+3)/7	Reducer 2: 0/1
INFO  : Map 1: 0(+6)/7	Reducer 2: 0/1
INFO  : Map 1: 0(+7)/7	Reducer 2: 0/1
INFO  : Map 1: 0(+7)/7	Reducer 2: 0/1
INFO  : Map 1: 1(+6)/7	Reducer 2: 0/1
INFO  : Map 1: 7/7	Reducer 2: 0/1
INFO  : Map 1: 7/7	Reducer 2: 0(+1)/1
INFO  : Map 1: 7/7	Reducer 2: 1/1
+---------+--+
|   _c0   |
+---------+--+
| 733917  |
+---------+--+
1 row selected (24.583 seconds)
```

#### Create ORC table from text
```
DROP TABLE IF EXISTS medicare_part_b.medicare_part_b_2013_raw;	
CREATE TABLE medicare_part_b.medicare_part_b_2013_raw 
COMMENT 'Data Munging Medicare Part B Dataset 2013'
STORED AS ORC tblproperties ("orc.compress"="ZLIB") 
AS
SELECT * FROM medicare_part_b.medicare_part_b_2013_text
where npi != '';
```

#### Now in PIG
enter the grunt shell with the following command and then paste the code below

```
pig -x tez -useHCatalog

ROWS = load 'medicare_part_b.medicare_part_b_2013_raw' using org.apache.hive.hcatalog.pig.HCatLoader();
HCPCS_CODES = foreach ROWS generate hcpcs_code, REGEX_EXTRACT(hcpcs_code,'(^[A-Z0-9]\\d{3,3}[A-Z0-9]$)',1) as match;
INVALID_CODES = filter HCPCS_CODES by match is null;
INVALID_CODE_G = group INVALID_CODES by hcpcs_code;
INVALID_CODE_CNT = foreach INVALID_CODE_G generate group as hcpcs_code, COUNT(INVALID_CODES) as count;
rmf medicare_part_b/bad_codes;
STORE INVALID_CODE_CNT into 'medicare_part_b/bad_codes' using PigStorage(',');
quit;
```

We use a complex regular expression to identify valid HCPCS codes as follows:
- They start with either a capital letter or a number.
- This is followed by three digits.
- The code ends in either a capital letter or a number.
Pig’s REGEX_EXTRACT returns NULL if the regular expression match is not met, so our hcpcs_code field in relation ROWS would therefore be NULL if the HCPCS code in the original file is invalid.
The report generated can and should be used, ideally, to correct the syntactic mistakes or, at a minimum, to get a firm grasp on the expected set of bad data and types of mistakes that are most common. Our dataset, thankfully, is quite clean, though there are some empty fields.

#### inspect the results
```
hdfs dfs -ls medicare_part_b/bad_codes
hdfs dfs -cat medicare_part_b/bad_codes/part-v001-o000-r-00000 | less
```

## Feature Matrix
### Sampling

Example: Sampling with Pig, Hive, and Spark
Consider our example Medicare dataset; it’s often advantageous to take only a sample of the whole dataset for processing. Of course, there are many types of sampling and a variety of ways to accomplish this in the Hadoop ecosystem.
With Pig, there are a variety of options for sampling. The built-in SAMPLE operator provides simple probabilistic sampling:

```
DEFINE HCatLoader org.apache.hive.hcatalog.pig.HCatLoader();
ROWS = load 'medicare_part_b.medicare_part_b_2013_raw' using HCatLoader();
SAMPLE_ROWS = sample ROWS 0.2;
rmf medicare_part_b/ex2_simple_sample;
STORE SAMPLE_ROWS into 'medicare_part_b/ex2_simple_sample' using PigStorage(',');
```
### inspect the results using fs commands in Grunt
```
grunt> fs -ls medicare_part_b/ex2_simple_sample
Found 2 items
-rw-r--r--   3 root hdfs          0 2016-04-21 01:14 medicare_part_b/ex2_simple_sample/_SUCCESS
-rw-r--r--   3 root hdfs   16793047 2016-04-21 01:14 medicare_part_b/ex2_simple_sample/part-v000-o000-r-00000
grunt> fs -cat medicare_part_b/ex2_simple_sample/part-v000-o000-r-00000 | less
1992996078,ALKASS,RAMZI,,MD,M,I,263 FARMINGTON AVE,,FARMINGTON,060300001,CT,US,Obstetrics/Gynecology,Y,O,Q0091,"Screening papanicolaou smear; obtaining, preparing and conveyance of cervical or vaginal smear to laboratory",,,, 13 ,$50.03,$0.00,$71.54,$5.33
1992998306,ABRAHAM,REFAAT,A,M.D.,M,I,500 W WILLOW ST,,LONG BEACH,908062831,CA,US,General Practice,Y,O,99213,"Established patient office or other outpatient visit, typically 15 minutes",,,, 24 ,$78.91,$0.00,$100.00,$0.00
```
### DataFu Sampling Example (DOES NOT WORK)
##### tracking the progress at https://community.hortonworks.com/questions/19638/data-munging-with-hadoop-datafu-sampling-example-g.html
```
DEFINE HCatLoader org.apache.hive.hcatalog.pig.HCatLoader();
DEFINE SampleByKey datafu.pig.sampling.SampleByKey('0.2');
ROWS = load 'medicare_part_b.medicare_part_b_2013_raw' using HCatLoader();
SAMPLE_BY_PROVIDERS = filter ROWS by SampleByKey(npi);
rmf medicare_part_b/ex2_by_npi_sample;
STORE SAMPLE_BY_PROVIDERS into 'medicare_part_b/ex2_by_npi_sample' using PigStorage(',');
```

### Sampling in Hive with TABLESAMPLE
to sample 10,000 rows, randomly from the medicare_part_b_2013_raw table, whereas
```
SELECT * FROM medicare_part_b.medicare_part_b_2013_raw TABLESAMPLE(10000 ROWS);
```

to sample 20% of the original table 
##### (requires a large dataset > 100GB to work, otherwise the workaround is to set hive.tez.input.format=${hive.input.format};)
```
SELECT * FROM medicare_part_b.medicare_part_b_2013_raw TABLESAMPLE(20 percent);
```
Running TABLESAMPLE with PERCENTAGE yields "FAILED: SemanticException 1:68 Percentage sampling is not supported in org.apache.hadoop.hive.ql.io.HiveInputFormat. Error encountered near token '20'"
```
set hive.tez.input.format=${hive.input.format};
SELECT * FROM medicare_part_b.medicare_part_b_2013_raw TABLESAMPLE(1 percent) limit 10;
```
```
OK
CPT copyright 2012 American Medical Association.  All Rights Reserved.																			NULL	NULL	NULL
npi	nppes_provider_last_org_name	nppes_provider_first_name	nppes_provider_mi	nppes_credentials	nppes_provider_gender	nppes_entity_code	nppes_provider_street1	nppes_provider_street2	nppes_provider_city	nppes_provider_zip	nppes_provider_state	nppes_provider_country	provider_type	medicare_participation_indicator	place_of_service	hcpcs_code	hcpcs_description	 hcpcs_drug_indicator 	NULL	NULL	NULL	average_Medicare_allowed_amt	stdev_Medicare_allowed_amt	average_submitted_chrg_amt	stdev_submitted_chrg_amt	average_Medicare_payment_amt
1003002494	ANDERSON	JOSEPH	M	M.D.	M	I	802 B ST		SAN RAFAEL	949013026	CA	US	Pathology	Y	O	88342	Tissue or cell analysis by immunologic technique	 N 	NULL	NULL	NULL	$145.57	$0.00	$257.00	$0.00	$115.52
1003002494	ANDERSON	JOSEPH	M	M.D.	M	I	802 B ST		SAN RAFAEL	949013026	CA	US	Pathology	Y	O	G0416	"Surgical pathology	 gross and microscopic examinations for prostate needle saturation biopsy sampling	NULL	NULL	NULL	 59 	 59 	$866.35	$0.00	"$1
1003002502	ADKINS	CAROL	J	PT	F	I	1605 SCHERM RD		OWENSBORO	423015300	KY	US	Physical Therapis	Y	O	97001	Physical therapy evaluation	 N 	NULL	NULL	NULL	$70.26	$0.00	$103.76	$14.40	$53.11
1003002502	ADKINS	CAROL	J	PT	F	I	1605 SCHERM RD		OWENSBORO	423015300	KY	US	Physical Therapis	Y	O	97035	"Application of ultrasound to 1 or more areas	 each 15 minutes"	NULL	NULL	NULL	 45 	$10.20	$0.67	$19.78	$5.51
1003002502	ADKINS	CAROL	J	PT	F	I	1605 SCHERM RD		OWENSBORO	423015300	KY	US	Physical Therapis	Y	O	97110	"Therapeutic exercise to develop strength	 endurance	NULL	NULL	NULL	 N 	 426 	 77 	 181 	$26.84
1003002502	ADKINS	CAROL	J	PT	F	I	1605 SCHERM RD		OWENSBORO	423015300	KY	US	Physical Therapis	Y	O	97116	"Walking training to 1 or more areas	 each 15 minutes"	NULL	NULL	NULL	 11 	$22.58	$1.84	$35.00	$4.67
1003002502	ADKINS	CAROL	J	PT	F	I	1605 SCHERM RD		OWENSBORO	423015300	KY	US	Physical Therapis	Y	O	97140	"Manual (physical) therapy techniques to 1 or more regions	 each 15 minutes"	NULL	NULL	NULL	 50 	$24.70	$2.04	$48.92	$3.19
1003002502	ADKINS	CAROL	J	PT	F	I	1605 SCHERM RD		OWENSBORO	423015300	KY	US	Physical Therapis	Y	O	G0283	"Electrical stimulation (unattended)	 to one or more areas for indication(s) other than wound care	NULL	NULL	NULL	 15 	 20 	$9.96	$1.11	$31.00
```
### Sampling in Spark
Spark has a sampling feature in its RDD and Data Frame API. For example, the following code shows how to sample a Spark data frame using the pyspark API:

start pyspark REPL with
```
/usr/hdp/current/spark-client/bin/pyspark
```
```
from pyspark.sql import SQLContext
from pyspark.sql import HiveContext
hc = HiveContext(sc)

# load medicare dataset as a Spark dataframe
rows = hc.sql("select * from medicare_part_b.medicare_part_b_2013_raw")

#Create a new Spark dataframe with 20% sample rows, without replacement
sample = rows.sample(False, 0.2)
sample.take(5)
```

### Example: Feature Generation with Hive
For example, consider our Medicare dataset. We can consider as an aggregated feature for each provider the percentile of the number of times each procedure was performed:

```
SELECT d.NPI as provider, d.HCPCS_CODE as code,
CASE
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[0] THEN "10th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[1] THEN "20th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[2] THEN "30th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[3] THEN "40th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[4] THEN "50th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[5] THEN "60th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[6] THEN "70th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[7] THEN "80th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[8] THEN "90th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[9] THEN "95th"
    WHEN cast(LINE_SRVC_CNT as int) <= p.percentiles[10] THEN "99th"
    ELSE "99+th"
END as percentile
from medicare_part_b.medicare_part_b_2013_raw d
join
(
  select HCPCS_CODE,
    percentile(cast(LINE_SRVC_CNT as int),
      array( 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 0.95, 0.99)
    ) as percentiles
  from medicare_part_b.medicare_part_b_2013_raw
  group by HCPCS_CODE
) p on d.HCPCS_CODE=p.HCPCS_CODE;
```
```
| 1083693022                                                              | Q2034                                        | 99+th       |
| 1558355107                                                              | Q2034                                        | 99+th       |
| 1750588729                                                              | Q2034                                        | 99+th       |
| 1629054481                                                              | S0281                                        | 99+th       |
| 1326150889                                                              | S0281                                        | 99+th       |
| 1891776837                                                              | S0281                                        | 99+th       |
| 1285661967                                                              | S0281                                        | 99+th       |
| 1356320675                                                              | S0281
```

## Text Features

### Example: TF-IDF with Spark (TODO)
Spark provides TF-IDF functionality as part of MLlib. Consider a corpus of documents stored in HDFS under the folder hdfs://corpus/ with multiple text files inside the folder:

#### requires additional packages on Sandbox
```
yum install -y numpy
# Reportable Communicable Disease Cases, 2010 - 2012
wget -O diseases-cases https://data.illinois.gov/api/views/3bgy-qtma/rows.csv?accessType=DOWNLOAD
hdfs dfs -put diseases-cases .
```

```
/usr/hdp/current/spark-client/bin/pyspark
```

```
from pyspark import SparkContext
from pyspark.mllib.feature import HashingTF
from pyspark.mllib.feature import IDF

sc = SparkContext()
documents = sc.wholeTextFiles("hdfs://sandbox.hortonworks.com:8020/user/root/diseases-cases").map(lambda (file, contents): contents.split(" "))
tf = HashingTF().transform(documents)
tf.cache()

idf = IDF().fit(tf)
tfidf = idf.transform(tf)
tfidf.collect()
```
```
[SparseVector(1048576, {44652: 0.0, 44861: 0.0, 46935: 0.0, 76577: 0.0, 157136: 0.0, 163449: 0.0, 179695: 0.0, 203287: 0.0, 206620: 0.0, 215446: 0.0, 227980: 0.0, 274780: 0.0, 274855: 0.0, 290401: 0.0, 291820: 0.0, 305578: 0.0, 311286: 0.0, 332761: 0.0, 343168: 0.0, 352442: 0.0, 361654: 0.0, 390090: 0.0, 420048: 0.0, 440734: 0.0, 442952: 0.0, 467227: 0.0, 509909: 0.0, 525390: 0.0, 532274: 0.0, 548259: 0.0, 552760: 0.0, 583167: 0.0, 589082: 0.0, 598534: 0.0, 600581: 0.0, 604415: 0.0, 613792: 0.0, 619700: 0.0, 641431: 0.0, 645227: 0.0, 680196: 0.0, 680373: 0.0, 697155: 0.0, 697665: 0.0, 715648: 0.0, 722384: 0.0, 724998: 0.0, 725041: 0.0, 750178: 0.0, 763151: 0.0, 771367: 0.0, 798997: 0.0, 811546: 0.0, 841940: 0.0, 876792: 0.0, 881704: 0.0, 888398: 0.0, 900712: 0.0, 924157: 0.0, 940059: 0.0, 948645: 0.0, 953023: 0.0, 954540: 0.0, 956673: 0.0, 972638: 0.0, 975538: 0.0, 980269: 0.0, 992309: 0.0, 1029285: 0.0, 1030061: 0.0})]
```
```
quit()
```

### NLP: Named Entity Recognition
For instance, if we have a file on HDFS called corpus with sentences in it, one sentence per line, we could use Spark’s Python bindings along with the very powerful natural language processing library NLTK from Python to extract the named entities, line by line:

```
easy_install nltk
```
```
[root@sandbox ~]# python
Python 2.6.6 (r266:84292, Jul 23 2015, 15:22:56)
[GCC 4.4.7 20120313 (Red Hat 4.4.7-11)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import nltk
>>> nltk.download('punkt')
[nltk_data] Downloading package punkt to /root/nltk_data...
[nltk_data]   Package punkt is already up-to-date!
True
>>> nltk.download('averaged_perceptron_tagger')
...
>>> nltk.download('maxent_ne_chunker')
...
>>> nltk.download('words')
```
```
/usr/hdp/current/spark-client/bin/pyspark
```

```
from pyspark import SparkContext
import nltk
def get_entities(text):
    tokens = nltk.tokenize.word_tokenize(text)
    pos = nltk.pos_tag(tokens)
    sentt = nltk.ne_chunk(pos, binary = True)
    ret = []
    for subtree in sentt.subtrees(filter=lambda t: t.label() == 'NE'):
        val = ' '.join(c[0] for c in subtree.leaves())
        ret.append(val)
    return ret
    
# hit return twice   
entities=sc.textFile("diseases-cases").map(lambda sentence : get_entities(sentence)).collect()
print(entities)
```
```
[[], [], [u'Blastomycosis'], [], [], [], [u'California'], [u'Campylobacter'], [], [], [], [u'Clostriduim'], [], [], [], [], [], [], [u'STEC'], [], [u'Foodborne'], [], [u'Group'], [], [u'Hantavirus'], [u'Hemolytic', u'HUS'], [], [], [], [], [], [u'Leprosy', u'Hansen'], [], [], [u'Lyme'], [], [], [u'Aseptic'], [], [], [], [], [], [], [], [], [], [u'Reye'], [u'Rocky Mountain'], [], [], [], [], [], [], [], [], [], [], [u'Typhoid Fever'], [], [u'Waterborne'], [u'West Nile'], []]
>>> len(entities)
64
```

### NLP: Word Vectorization
Often this vectorized format is a useful way of looking at textual data and can facilitate doing things like including multiple similar words together or finding similar words to include as features. It has been used to great effect for feature generation and for creating many of the traditional NLP tools, like named entity recognizers or machine translators.
There is a very nice implementation of word2vec in Spark’s MLlib:

```
python
import nltk
nltk.download()
d
downloader> d

Download which package (l=list; x=cancel)?
  Identifier> pil
    Downloading package pil to /root/nltk_data...
      Package pil is already up-to-date!

```
```
/usr/hdp/current/spark-client/bin/pyspark --master yarn-client --num-executors 3 --executor-memory 512m --executor-cores 1
```
##### using Count of Monte Cristo as corpus for this example, splitting on empty space and setting minimum count for Word2Vec to 2 from default 5
```
from pyspark import SparkContext
from pyspark.mllib.feature import Word2Vec
tokenized_data = sc.textFile("montecristo.txt").map(lambda row: row.split(" "))
word2vec = Word2Vec().setMinCount(2)
w2v_model = word2vec.fit(tokenized_data)
synonyms = w2v_model.findSynonyms('revenge', 10)
print(synonyms)
```
```
[(u'abandon', 0.26790176689080203), (u'sorry', 0.26672743270554655), (u'attribute', 0.26592503989436311), (u'me!', 0.26545624105837556), (u'do.', 0.26521806541581217), (u'swear', 0.26434384373504261), (u'absurd', 0.2635071062905775), (u'have?', 0.26339475990565209), (u'again;', 0.26332906976416132), (u'strive', 0.26248300471629782)]
```

## Feature Projection - PCA and Friends
##### Example: Dimensionality Reduction with Spark
The following code example shows how to perform principal component analysis with Spark. In this example, we assume there is already a RowMatrix built that contains all of our features:

#### download sample dataset
```
wget curl "https://www.mockaroo.com/00118c50/download?count=1000&key=89e294f0" > "patients.csv"
hdfs dfs -put patients.csv .
```
#### using spark-shell now
```
/usr/hdp/current/spark-client/bin/spark-shell --master yarn-client --num-executors 3 --executor-memory 512m --executor-cores 1
```

```
import org.apache.spark.mllib.linalg.Matrix
import org.apache.spark.mllib.linalg.distributed.RowMatrix

// Assume we've built a row matrix somehow before the PCA
val mat: RowMatrix = ...  // some RowMatrix as input to PCA

// Perform PCA to find top 20 principal components
// Principal components are stored in a local dense matrix 'pc'
val pc: Matrix = mat.computePrincipalComponents(20)

// Project rows to linear space spanned by top 20 principal components
val projected: RowMatrix = mat.multiply(pc)
```
