PROC SORT → NODUPKEY, OUT=, DUPOUT= 차이까지 정리

PROC IMPORT → GUESSINGROWS= 기본값과 주의점, GETNAMES= 옵션

MERGE → IN= 옵션 응용, BY 없을 때 에러/카르테시안 곱 발생

PROC TRANSPOSE → PREFIX=, NAME= 활용법

FORMAT vs INFORMAT → 시험 단골 헷갈림

WHERE vs IF → 실행 시점 차이 (DATA step vs Procedure 단계)

ODS → PDF/RTF/HTML 중 시험에서 묻는 옵션

숫자형 변수 길이 → 저장 바이트수, 정밀도 시험 포인트

날짜 함수 → INTNX, INTCK, YRDIF (꼭 시험 출제)



# SAS BASE 자격시험 대비 공부노트

## 1. Access and Create Data Structures (20–25%)
### 1-1. Create temporary and permanent SAS data sets

#### **[ 개념 ]**
- 임시 데이터셋: WORK 라이브러리에 저장 → 세션 종료 시 삭제됨.
- 영구 데이터셋: 지정한 라이브러리에 저장 → 이후에도 사용 가능.

#### **[ 예제 ]**
```
/* 임시 데이터셋 생성 */

data work.temp;
	set sashelp.class;
run;

/* 영구 데이터셋 생성 */

libname mylib 'D:\sas_study\library';
data mylib.students;
	set sashelp.class;
run;
```

#### **[ 핵심 포인트 ]**

- WORK.는 생략 가능.
- 영구 데이터셋 만들 때는 반드시 LIBNAME으로 경로를 할당해야 함.

### 1-2. Investigate SAS data libraries

#### **[ 개념 ]** 
- 데이터셋의 속성(변수명, 타입, 길이 등) 확인 가능.

#### **[ 예제 ]** 
```commandline
libname mylib 'D:\sas_study\library';

/* 라이브러리 내 데이터셋 구조 확인 */
proc contents data=mylib.students;
run;
```
#### **[ 핵심 포인트 ]** 
- PROC CONTENTS는 시험에서 자주 나오는 필수 구문.
- 변수 길이, label, format, 인덱스 여부 등을 확인하는 문제 출제됨.

### 1-3. Access data

#### **[ 개념 ]**
- 다양한 데이터 불러오기 방법 활용.

#### **[ 예제 (SET statement) ]**
```
data new;
    set sashelp.class;  /* 기존 데이터셋 읽기 */
    bmi = (weight / (height*height)) * 703;
run;
```
#### **[ 예제 (PROC IMPORT – CSV/Excel) ]**
```
/* CSV 불러오기 */
proc import datafile='D:\sas_study\library\students.csv'  /* 불러올 원본 CSV 파일 경로 지정 */
	out=work.students_2                                     /* 결과로 생성할 SAS 데이터셋 이름 (work 라이브러리 안에 students_2) */
	dbms=csv replace;                                       /* DBMS 형식: csv 파일 / 기존 같은 이름 데이터셋 있으면 덮어쓰기 */
	guessingrows=100;                                       /* 앞에서 100행을 읽어 변수 유형(문자/숫자)과 길이를 추정 */
	delimiter=',';                                          /* CSV 파일의 구분자를 ','(콤마)로 지정 */
run;                                                       /* PROC IMPORT 실행 */

/* Excel 불러오기 */
proc import datafile='D:\sas_study\library\students.xlsx'
	out=work.students_3
	dbms= xlsx replace;
run;
```
#### **[ 예제 (data infile – txt) ]**
```commandline
data work.students_4;
    infile 'D:\sas_study\library\students.txt' dlm=',' dsd firstobs=2;
    /* 
        dlm=','   → 구분자를 콤마로 지정
        dsd       → 연속된 구분자 처리, 따옴표로 감싼 값 인식
        firstobs=2 → 첫 번째 줄(헤더: name, sex, age, height, weight)은 건너뛰고 2행부터 읽기
    */
    input name : $8. sex $ age height weight;
    /*  뒤에 $ 표시하면 문자열은 뒤에 반드시 & 표시를 하여야 함 ' $' 또는 ':$20.' 길이까지   
	     한글 한글자는 길이 2개 차지 */
run;

```
#### **[ 핵심 포인트 ]** 
- txt는 data infile input 구문 / csv, xlsx는 proc import 구문
- DBMS= 옵션 → csv, xlsx 지정.
- GUESSINGROWS=: 변수 타입/길이 추정 시 몇 행까지 확인할지.

### 1-4. Combine SAS data sets
#### **[ 개념 ]**
- 여러 데이터셋을 합치는 방법.

#### **[ 예제 (Concatenate / Merge) ]**
```commandline
/* 예제 데이터 생성 */
data class_info;
    input id name $ age;
    datalines;
   1 kim 20
   2 lee 21
   3 park 22
   4 choi 23
   ;
   run;
   
data class_score;
    input id major $ score;
    datalines;
    2 Stat 85
    3 CS   90
    4 Math 88
    5 Bio  92
    ;
run;

/* Concatenate : 연달아 이어 붙이기 */
data concat;
	set class_info class_score;
run;

/* Merge : 키 잡고 옆으로 붙이기 */

proc sort data=class_info; by id; run;
proc sort data=class_score; by id; run;

/* 1) Inner join */
data merged_inner;
    merge class_info (in=a) class_score (in=b);
    by id;
    if a and b;   /* inner join */
run;

/* 2) Outer join */
data merged_outer;
    merge class_info (in=a) class_score (in=b);
    by id;
    if a or b; 
run;

/* 3) Left join */
data merged_left;
    merge class_info (in=a) class_score (in=b);
    by id;
    if a; 
run;

```

#### **[ 핵심 포인트 ]** 
* MERGE 사용할 때는 반드시 BY 변수 기준으로 정렬해야 함. 
  * 'by 변수;' 없으면 모든 행이 Cartesian Product처럼 붙어버림

* (in= ) 옵션 자주 출제 (조건부 merge에서 활용).
  * if a and b; → inner join 
  * if a; → left join 
  * if b; → right join 
  * if a or b; → full outer join

* 정렬 필수
  * MERGE 전에 BY 변수 기준으로 반드시 PROC SORT 해야 함. 
  * 안 하면 오류 발생: ERROR: BY variables are not properly sorted on data set...

* Missing 값 처리 
  * 한쪽에만 있는 key는 다른 쪽 변수 값이 missing(. 또는 공백`)으로 채워짐.
  
### 1-5. Create and manipulate SAS date values

#### **[ 개념 ]** 
* SAS는 날짜를 1960-01-01 = 0으로 저장. (이전은 음수, 이후는 양수)
* 시간은 초 단위(datetime 값은 1960-01-01 00:00:00 기준)
* 날짜 상수 표현
  * (날짜) '25dec1999'd
  * (시간) '12:30't
  * (날짜+시간) '25dec1999:12:30'dt

#### **[ 날짜 관련 함수 ]** 
* TODAY(), DATE()
    * today_date = today();
    * today_date = date();

* YEAR, MONTH, DAY, QTR
    * y = year(today()) - 연도
    * m = month(today()) - 월
    * d = day(today()) - 일
    * q = qtr(today()) - 분기

* INTCK('기준',A,B) : (B-A) 기간 차이 계산
    * years = intck('year', '25dec1999'd, today())

* INTNX(interval, start, increment, alignment) 
  * alignment 옵션:
      * 'b': 구간의 시작일(beginning)
      * 'e': 구간의 끝(end)
      * 's': 시작 날짜와 동일한 위치
      * 'm': 중간(midpoint)

* yrdif(start_date, end_date, 'basis')
  * basis : 연도 계산 기준
    * 'ACT/ACT' → 실제 일수 기준
    * '30/360' → 금융권, 1개월=30일 기준
    * 'ACT/360' → 실제 일수/360 기준
    * 'ACT/365' → 실제 일수/365 기준

```commandline
data dates;
    today_date = today();             /* 오늘 날짜 */
    bday = '25dec1999'd;              /* 날짜 상수 */
    
    years = intck('year', bday, today_date);  /* 연도 차이 */
    months = intck('month', bday, today_date);  /* 월 차이 */
    days = intck('day', bday, today_date); /* 일수 차이 */
    
    next_month_same = intnx('month', today_date, 1, 'same');   /* 다음달 같은 날짜 */
    next_month_beg = intnx('month', today_date, 1, 'b');   /* 다음달 첫날 -> 기본값 */
    next_month_end = intnx('month', today_date, 1, 'e');   /* 다음달 마지막날 */
    next_month_mid = intnx('month', today_date, 1, 'middle');   /* 다음달 중간날짜 -> 31일 있는 달이면 16일 */
    
    put today_date= date9.;
    put next_month_same = date9. next_month_beg= date9. next_month_end= date9. next_month_mid= date9.;
run;
```

### 1-6. Control observations and variables

#### **[ 개념 ]**
* 원하는 행/열만 골라내기

#### **[ 예제 ]**
```
/* 행 선택 */
data teens;
    set sashelp.class;
    where age >= 13 and age <= 19;
run;

/* 열 선택 */
data subset;
    set sashelp.class (keep=name age);
run;

/* drop statement */
data dropvar;
    set sashelp.class;
    drop height weight;
run;

/* if절 활용 */
/* - 조건부 삭제*/
data teens_delete;
	set sashelp.class;
	if age < 13 or age > 19 then delete;
run;

/* - 조건부 값 변경/계산*/
data class_mod;
	set sashelp.class;
	if age >= 13 and age <=19 then group = 'teen';
	else group = 'other';

	if age >= 15 then score = 50+5;
	else score=50;
run;

/* - 조건부 출력 제어*/
data teens_out boys_out;
	set sashelp.class;
	if age >= 13 and age <= 19 then output teens_out;
	else if sex='남' then output boys_out;
run;
```

#### **[ 핵심 포인트 ]** 

* WHERE vs IF: WHERE는 데이터 읽기 전에 필터링, IF는 읽은 후 필터링.
* KEEP=, DROP= 옵션은 DATA step / SET / MERGE / PROC 어디서든 사용 가능.

## 2. Manage Data (35–40%)

### 2-1. Sort observations in a SAS data set
#### **[ 개념 ]**
* 데이터를 특정 변수 기준으로 정렬하거나 중복 제거

#### **[ 예제 ]**
```
/* 오름차순 정렬 */
proc sort data=sashelp.class out=sorted;
    by age;
run;

/* 내림차순 정렬 */
proc sort data=sashelp.class out=sorted_desc;
    by descending age;
run;

/* 중복 제거 */
proc sort data=sashelp.class out=nodup nodupkey;
    by name;
run;
```

#### **[ 핵심 포인트 ]** 

* OUT= 옵션 → 원본 보존하면서 새로운 데이터셋 생성.
* NODUP vs NODUPKEY:
  * NODUP: 모든 변수 기준 중복 제거.
  * NODUPKEY: BY 변수 기준 중복 제거.

### 2-2. Conditionally execute SAS statements

#### **[ 개념 ]**  
* 조건문으로 데이터 처리 제어

#### **[ 예제 ]** 
```
data flag;
    set sashelp.class;
    if age < 13 then group = "Child";
    else if age < 20 then group = "Teen";
    else group = "Adult";
run;

/* 여러 줄 실행 → DO-END */
data result;
	length group $20; * 미리 길이 설정 안하면 not child가 더 길어서 잘림;
	set sashelp.class;
	if age < 13 then do;
		group='child';
		flag = 0;
	end;
	else do;
		group='not child';
		flag = 1;
	end;
run;
```

#### **[ 핵심 포인트 ]** 

* IF-THEN/ELSE 기본 제어문.
* DO ... END; 블록 안에 여러 문장 실행 가능.

### 2-3. Use assignment statements in the DATA step

#### **[ 개념 ]**  
* 변수 생성/갱신

#### **[ 예제 ]** 
```
data assign;
    set sashelp.class;
    bmi = (weight / (height*height)) * 703;   /* 새로운 변수 생성 */
    age = age + 1;                            /* 기존 변수 값 갱신 */
    start_date = '01jan2025'd;                /* 날짜 상수 */
    format start_date date9.;
run;
```

#### **[ 핵심 포인트 ]** 

* = 연산자로 새로운 변수 만들거나 기존 값 수정 가능.
* 상수 날짜는 'ddmmmyyyy'd 형태.

### 2-4. Modify variable attributes

#### **[ 개념 ]**  
* 변수 이름, 길이, 라벨, 포맷 변경.

#### **[ 예제 ]** 
```
data modify;
    set sashelp.class (rename=(name=student_name));    /* 변수명 변경 */
    length newvar $20;            /* 문자 길이 지정 */
    label age="Student Age";      /* 변수 설명 추가 */
    format height 5.2;            /* 소수점 형식 */
run;
```

#### **[ 핵심 포인트 ]** 
* RENAME=, LENGTH, LABEL, FORMAT 시험 단골
* LENGTH는 문자열 변수 값 입력 전에 선언해야 적용됨
* FORMAT(출력형식) : 숫자에서 중요함. 문자열은 FORMAT 보단 LENGTH 중요
    * COMMAw.d: 천 단위 콤마
    * DOLLARw.d: $ + 콤마
    * PERCENTw.d: 100배하고 % 표시
    * BESTw.: 가장 깔끔한 숫자 자동 선택
    * Z w.: 앞자리 0 채우기
    * w.d: 기본 숫자 포맷 (콤마 없음)
    * 날짜는 숫자 → format 필수 (date9./ datetime19. / mmddyy10. / yymmdd10. / ddmmyy10.)

### 2-5. Accumulate sub-totals and totals

#### **[ 개념 ]**  
* 그룹별 집계

#### **[ 예제 ]** 
```
proc sort data=sashelp.class out=class;
	by sex;
run;

data totals1;
	set class;

	/* data문 안에서의 by절 - 우선 먼저 proc sort로 정렬 먼저 해야 사용 가능
	: SAS에게 성별 그룹별 처리하겠다고 알림
	: 이제 SAS는 각 성별 그룹의 첫 번째(first.sex), 마지막(last.sex) 관측치를 인식할 수 있음 */
	by sex;
	/* retain : DATA step이 반복되면서 변수가 초기화되지 않고 값을 유지하도록 함
					일반 변수는 DATA step 반복마다 값이 초기화됨*/
	retain total_wt 0;
	total_wt + weight; /* 누적합 */
	if first.sex then do;
		total_wt=weight; end;
run;

data totals2;
	set class;

	/* data문 안에서의 by절 - 우선 먼저 proc sort로 정렬 먼저 해야 사용 가능
	: SAS에게 성별 그룹별 처리하겠다고 알림
	: 이제 SAS는 각 성별 그룹의 첫 번째(first.sex), 마지막(last.sex) 관측치를 인식할 수 있음 */
	by sex;
	/* retain : DATA step이 반복되면서 변수가 초기화되지 않고 값을 유지하도록 함
					일반 변수는 DATA step 반복마다 값이 초기화됨*/
	retain total_wt 0;
	total_wt + weight; /* 누적합 */
	if last.sex then do;
		output; * 그 시점까지 누적된 합계를 데이터셋에 저장 -> 그룹별 마지막 사람 행만 남음;
		total_wt=0;
		end;
run;

```

#### **[ 핵심 포인트 ]** 

* BY문 + FIRST./LAST. 변수 활용.
* RETAIN + 누적 변수(+) 조합 시험에 자주 나옴.

### 2-6. Use SAS functions

#### **[ 문자 함수 ]**

```
data char_fn;
    str = "   Hello, SAS!   "; * 데이터 상에서 뒷자리 공백은 보이지 않음;
/* translate(str, 'a', 'b') -> str의 포함 문자 중 b를 a로 수정 */
	tran_str = translate(str, '_', ' '); 
/* upcase(str) -> 모두 대문자로 변경 */
    up = translate(upcase(str),'_',' ');
/* lowcase(str) -> 모두 소문자로 변경 */
    low =  translate(lowcase(str),'_',' ');
/* left(str) -> 앞공백을 뒤로 몰아넣기 */
	left = translate(left(str),'_',' ');
/* trim(str) -> 뒷공백 제거(앞공백 유지) */
	trim = translate(trim(str),'_',' ');
	no_space = translate(trim(left(str)),'_',' ');
/* strip(str) -> 앞뒤 모든 공백 제거(중간 공백은 그대로) */	
	strip = translate(strip(str),'_',' ');
/* substr(str, n, m) -> 문자열에서 n번째 문자부터 m개 추출(공백도 카운트) */	
    part63 = substr(str, 6, 3);
    part63_no_space = substr(no_space, 6, 3);

/* compress(str, 'abc') -> 문자열에서 a와 b와 c 제거 -> 구분자 없이 ' '안에 없애고 싶은 문자 다 넣으면 됨 */
    cleaned = translate(compress(str, ',!'),'_',' ');

/* scan(string, n, delimiters) -> n : n번째 문자열 가져옴(-1은 마지막문자열), delimiters : 구분자(생략하면 개수 상관없이 공백) */

	str2 = "Apple  Banana Orange";
	x = scan(str2, 2); * Banana;
	y = scan(str2, -1); * Orange;

	email = "user@example.com";
	id  = scan(email, 1, '@');
	dom = scan(email, 2, '@');

run;

```

#### **[ 숫자 함수 ]**
```commandline
data num_fn;
	x = 10.7;
	rounded = round(x, 1);
	intpart = int(x);
	random_uni = rand("uniform"); * 기본 균등분포 (0 ~ 1);
	random_normal = rand("normal", 0, 1); * 정규분포 (평균=0, 표준편차=1);
	random_binom = rand("binomial", 0.3, 10); * 이항분포 (n=10, p=0.3);
	random_pois = rand("poisson", 5); * 포아송분포(λ=5);
	random_range = 1 + rand("uniform") * 99; * 범위 지정 랜덤 숫자 (예: 1~100);
run;
```
#### **[ 날짜 함수 ]**
```
data date_fn;
	today_d = today();
	year_v = year(today_d);

/* intcx("qtr/month/year/day", start_date, end_date) */   
    years = intck('year', bday, today_date);  /* 연도 차이 */
    months = intck('month', bday, today_date);  /* 월 차이 */
    days = intck('day', bday, today_date); /* 일수 차이 */
   
/* intnx("qtr/month/year/day", start_date, n, 'b/e/same/middle') */   
    next_month_same = intnx('month', today_date, 1, 'same');   /* 다음달 같은 날짜 */
    next_month_beg = intnx('month', today_date, 1, 'b');   /* 다음달 첫날 -> 기본값 */
    next_month_end = intnx('month', today_date, 1, 'e');   /* 다음달 마지막날 */
    next_month_mid = intnx('month', today_date, 1, 'middle');   /* 다음달 중간날짜 -> 31일 있는 달이면 16일 */
	years_passed = intck("year", '01jan2000'd, today_d);
	
	format today_d next_qtr next_month date9.;

/* mdy(month, day, year) -> 월/일/연도로 SAS 날짜 만들기 */
	d = mdy(11, 24, 2025);
	format d date9.;
	put d; /* 출력: 24NOV2025 */

/* yrdif(start_date, end_date, 'basis') -> 두 날짜 사이 연수 차이 계산(나이 계산시 중요)
    '30/360' - 매달 30일 기준
    'ACT/360' - 실제일수/360
	'ACT/365' - 실제일수/365
	'ACT/ACT' - 실제일수/실제연도일수 */
	dob = '29feb1992'd;
	today = '28feb2025'd;
	age_basic = yrdif(dob, today); * 33.0;
	age_act365 = yrdif(dob, today, 'ACT/365'); * 33.021917808;
	age_actact = yrdif(dob, today, 'ACT/ACT'); * 32.997701924;   
	
run;
```

#### **[ 핵심 포인트 ]** 

* 문자: SCAN, SUBSTR, TRIM, COMPRESS, UPCASE, LOWCASE
* 숫자: SUM, MEAN, ROUND, INT, RAND
* 날짜: MDY, TODAY, YEAR, MONTH, DAY, INTCK, INTNX, YRDIF

### 2-7. Convert character ↔ numeric

#### **[ 개념 ]**  
* INPUT(문자→숫자)
* PUT(숫자→문자)

#### **[ 예제 ]** 
```
data convert;
    char_num = "123";
    real_num = input(char_num, 8.);     /* 문자 → 숫자 */

    num_val = 2025;
    str_val = put(num_val, 4.);         /* 숫자 → 문자 */
run;
```

#### **[ 핵심 포인트 ]** 

SAS는 자동 변환을 하기도 하지만 시험에서는 명시적 변환 중요.

### 2-8. Process data using DO loops

#### **[ 예제 ]** 

/* 반복 DO */
data loop_ex;
    do i = 1 to 5;
        square = i**2;
        output;
    end;
run;

/* 조건부 DO */
data loop_cond;
    x = 1;
    do while (x < 100);
        x + 10;
        output;
    end;
run;


#### **[ 핵심 포인트 ]** 

DO i = 1 to n; → 반복 loop.

DO WHILE(condition); / DO UNTIL(condition); 시험 출제.

### 2-9. Restructure with PROC TRANSPOSE

#### **[ 개념 ]**  Wide ↔ Long 변환.

#### **[ 예제 ]** 

proc transpose data=sashelp.class out=transposed prefix=col;
    var height weight;
    id name;
run;


#### **[ 핵심 포인트 ]** 

VAR: 변환할 변수.

ID: 컬럼 이름으로 바꿀 변수.

PREFIX=: 자동 생성되는 변수명 접두사.

### 2-10. Macro variables

#### **[ 개념 ]**  코드 재사용성 향상.

#### **[ 예제 ]** 

%let year = 2025;

data sales&year;
    set sales_data;
    if year = &year;
run;


#### **[ 핵심 포인트 ]** 

%LET으로 매크로 변수 정의.

호출 시 &변수명. → . 붙이면 다른 문자와 구분 가능.

## Part 3 – Error Handling (15–20%)
🎯 학습 목표

프로그래밍 논리 오류 파악 및 해결

문법 오류 식별 및 수정

SAS 로그 활용 능력

### 3-1. Identify and Resolve Programming Logic Errors

PUTLOG Statement

DATA step 실행 중 변수 값 확인 가능

문법:

putlog var1= var2=;


👉 실행 로그에 var1=값 var2=값 출력

조건부 PUTLOG

if age < 0 then putlog "ERROR: Invalid age " age=;


👉 잘못된 값이 들어왔을 때만 로그 출력

N / ERROR 변수

_N_: DATA step 반복 횟수 (레코드 번호 느낌)

_ERROR_: 오류 발생 시 1, 정상은 0

활용 예시:

data test;
  set sashelp.class;
  if _N_ <= 5 then putlog _all_;
run;

### 3-2. Recognize and Correct Syntax Errors

SAS 문법 기본 규칙

각 문장은 **세미콜론(;)**으로 끝남

변수명: 최대 32자, 알파벳/숫자/언더스코어 사용 가능

예약어는 변수명으로 불가 (예: data, set)

자주 발생하는 Syntax Error

세미콜론 누락

옵션 철자 오류 (예: DELIMTER → DELIMITER)

큰따옴표/작은따옴표 짝이 안 맞음

로그 해석 예시

data work.test
  set sashelp.class;
run;


👉 로그에 ERROR 180-322: Statement is not valid or it is used out of proper order.
→ data 문 끝에 세미콜론 빠짐

### 3-3. Debugging with the Log

NOTE / WARNING / ERROR 메시지 해석 연습 필수

NOTE: 정보 메시지 (예: 변수 길이 잘림, 자동 형변환)

WARNING: 주의 필요 (예: BY 변수 정렬 안 됨)

ERROR: 실행 불가 (예: 옵션 잘못됨, 데이터셋 없음)

✅ 연습문제
문제 1

다음 코드 실행 시 로그에 어떤 메시지가 나올까?

data new;
  set sashelp.class;
  if age = "15" then output;
run;


👉 답:

age는 numeric, "15"는 character → 자동 형 변환 발생

로그에 NOTE: Character values have been converted to numeric values 출력

문제 2

다음 코드 실행 시 오류를 찾으시오:

proc print data sashelp.class;
  var name age;
run;


👉 답:

data sashelp.class; ❌ → 세미콜론 누락

올바른 코드:

proc print data=sashelp.class;
  var name age;
run;


💡 시험 팁:

에러 자체를 고치는 문제보다는 **어떤 로그가 나올까?**를 묻는 경우가 많음!

특히 자동 형 변환, 세미콜론 빠짐, 잘못된 옵션 문제 자주 나옴.


## Part 4 – Generate Reports and Output (15–20%)
🎯 학습 목표

PROC PRINT, PROC FREQ, PROC MEANS, PROC UNIVARIATE 활용

라벨, 포맷, 타이틀, 푸터 등 보고서 꾸미기

ODS (Output Delivery System) 활용해서 PDF/Excel/HTML 출력

데이터 Export

### 4-1. Generate List Reports (PROC PRINT)

기본 문법

proc print data=sashelp.class;
  var name age height weight;
run;


주요 옵션 및 문장:

VAR: 출력 변수 선택/순서 조정

WHERE: 조건부 출력

SUM: 합계

ID: ID 변수 지정

BY: 그룹별 출력

NOOBS: 관측치 번호 제거

LABEL: 변수 라벨 사용

### 4-2. Generate Summary Reports (PROC FREQ, PROC MEANS, PROC UNIVARIATE)

PROC FREQ

proc freq data=sashelp.class;
  tables sex*age / nocol norow nopercent;
run;


일원/이원 빈도표

옵션: NLEVELS, ORDER=

PROC MEANS

proc means data=sashelp.class n mean std min max;
  class sex;
  var height weight;
run;


요약통계: 평균, 표준편차, 최소/최대

WAYS, CLASS, VAR, OUTPUT 활용

PROC UNIVARIATE

proc univariate data=sashelp.class;
  var height;
run;


이상치, 분포, 누락값 확인

### 4-3. Enhance Reports

사용자 정의 포맷 (PROC FORMAT)

proc format;
  value $gender 'M'='Male' 'F'='Female';
run;

proc print data=sashelp.class;
  format sex $gender.;
run;


TITLE / FOOTNOTE

title "학생 키와 몸무게 요약";
footnote "출처: SASHELP.CLASS 데이터";


라벨과 헤더 조정

proc print data=sashelp.class label;
  label height="키(cm)" weight="몸무게(kg)";
run;

### 4-4. ODS (Output Delivery System)

출력 형식 지정

ods pdf file="report.pdf" style=journal;
proc print data=sashelp.class;
run;
ods pdf close;


지원 포맷:

HTML

PDF

RTF

XLSX

PPTX

CSV

👉 STYLE= 옵션으로 테마 변경 가능

### 4-5. Export Data

PROC EXPORT

proc export data=sashelp.class
  outfile="class.csv"
  dbms=csv replace;
run;


SAS/ACCESS XLSX Engine

libname myxls xlsx "class.xlsx";
data myxls.classdata;
  set sashelp.class;
run;
libname myxls clear;

✅ 연습문제
문제 1

아래 코드 실행 시 어떤 출력이 나올까?

proc print data=sashelp.class noobs label;
  var name sex age;
  label sex="성별";
  where age >= 14;
run;


👉 답: 나이 14세 이상 학생의 이름, 성별, 나이 출력 / 성별 라벨 적용 / 관측번호 제거

문제 2

빈도표에서 성별과 나이를 교차표로 만들되, 퍼센트와 row/col 비율은 빼고 단순 개수만 보려면?
👉 답:

proc freq data=sashelp.class;
  tables sex*age / nocol norow nopercent;
run;


💡 시험 팁

PROC PRINT, PROC FREQ, PROC MEANS는 거의 100% 출제됨

ODS PDF/EXCEL 문법도 자주 물어봄 (특히 close 문 빠졌을 때 결과 예측 문제)


