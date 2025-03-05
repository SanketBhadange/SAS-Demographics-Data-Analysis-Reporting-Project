/* Generated Code (IMPORT) */
/* Source File: M3_T3_V1_-_demog.xls */
/* Source Path: /home/u64143416 */
/* Code generated on: 2/28/25, 6:47 PM */


  /* Dropping existing table */
%web_drop_table(WORK.Demog);

FILENAME REFFILE '/home/u64143416/M3_T3_V1_-_demog.xls';




PROC IMPORT DATAFILE=REFFILE
	DBMS=XLS
	OUT=WORK.Demog;
	GETNAMES=YES;
	SHEET="Demog";
RUN;

PROC CONTENTS DATA=WORK.Demog; 
RUN;

%web_open_table(WORK.Demog);

/* Section 1 : Summary stats for age */
data demog1 ;
   set demog;
   format dob1 date9. ;
   
   /* Creating a New variable for date of birth */
   dob = compress (cat(month,'/',day,'/',year));
   dob1 = input (dob,mmddyy10.);

   /* Calculating the age of patients */
   age = (diagdt - dob1) / 365;
   output;
   trt = 2;
   output;
run;

/* Evaluating the statistical parameters for age */
proc sort data = demog1;
by trt;
run;

proc means data = demog1 noprint;
var age;
output out = agestats;
by trt;
run;

/* Matching agestats to allstats for stacking */
data agestats;
   set agestats;
   length value $10.;
   ord = 1;

   /* Ensuring correct assignment of subord */
   if _stat_ = 'N' then do; subord = 1; value = strip(put(age, 8.)); end;
   else if _stat_ = 'MEAN' then do; subord = 2; value = strip(put(age, 8.1)); end;
   else if _stat_ = 'STD' then do; subord = 3; value = strip(put(age, 8.2)); end;
   else if _stat_ = 'MIN' then do; subord = 4; value = strip(put(age, 8.1)); end;
   else if _stat_ = 'MAX' then do; subord = 5; value = strip(put(age, 8.1)); end;

   rename _stat_ = stat;
   drop _type_ _freq_ age;
run;

/* Obtaining statistical parameters for gender */
proc format;
value genfmt
1 = 'Male'
2 = 'Female';
run;

data demog2;
   set demog1;
   sex = put(gender, genfmt.);
run;

/* Obtaining summary stats for sex */
proc freq data = demog2 noprint;
table trt * sex / outpct out = genderstats;
run;

/* Concatenating the COUNT and PERCENT variables */
data genderstats;
   set genderstats;
   value = cat(count, ' (', strip(put(round(pct_row, .1), 8.1)), '%)');
   ord = 2;
   if sex = 'Male' then subord = 1;
   else subord = 2;

   rename sex = stat;
   drop count percent pct_row pct_col;
run;

/* Obtain statistical parameters for race */
proc format;
value racefmt
1 = 'White'
2 = 'Black'
3 = 'Hispanic'
4 = 'Asian'
5 = 'Other';
run;

data demog3;
   set demog2;
   racec = put(race, racefmt.);
run;

/* Obtaining Summary Statistics For Race */
proc freq data = demog3 noprint;
table trt * racec / outpct out = racestats;
run;

data racestats;
   set racestats;
   value = cat(count, ' (', strip(put(round(pct_row, .1), 8.1)), '%)');
   ord = 3;

   /* Ensuring correct subord assignment */
   if racec = 'Asian' then subord = 1;
   else if racec = 'Black' then subord = 2;
   else if racec = 'Hispanic' then subord = 3;
   else if racec = 'White' then subord = 4;
   else if racec = 'Other' then subord = 5;

   rename racec = stat;
   drop count percent pct_row pct_col;
run;

/* Appending All Stats Together */
data allstats;
   set agestats genderstats racestats;
run;

/* Ensuring proper sorting before transposition */
proc sort data = allstats;
by ord subord stat;
run;

/* Transposing Data By Treatment Groups */
proc transpose data = allstats out = t_allstats prefix=_;
var value;
id trt;
by ord subord stat;
run;


data final;
length stat $50;
       set t_allstats;
       by ord subord;
       output;
       if first.ord then do;
          if ord=1 then stat='Age (years)';
          if ord=2 then stat ='Gender';
          if ord=3 then stat='Race';
          subord=0;
          _0='';
          _1='';
          _2='';
       output;  
       end;
       
 proc sort ;
 by ord subord;
 run;
/*Population count N=XX part*/

proc sql noprint;
select count(*) into : placebo from demog1 where trt=0;
select count(*) into : active from demog1 where trt=1;
select count(*) into : total from demog1 where trt=2;
quit;

%let placebo=&placebo;
%let active=&active;
%let total=&total;

/*Constructing the final report*/
title 'Table 1.1';
title2 'Demographic and Baseline Characteristics by Treatment Group';
title3 'Randomized Population';
footnote 'Note: Percentages are based on the number of non-missing values in each treatment group.';

proc report data = final split ='|';
columns ord subord stat _0 _1 _2;
define ord/ noprint order;
define subord/ noprint order;
define stat / display width =50 "";
define _0 / display width =30 "Placebo| (N=&placebo)";
define _1 / display width =30 "Active| Treatment (N=&active)"; 
define _2 / display width =30 "All Patients| (N=&total)"; 
run;


