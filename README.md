
# SAS BASE 자격시험 대비 공부노트

## 0. PDV(Program Data Vector)이란?

* DATA step이 실행될 때, 한 줄(한 관측치)을 담아두는 임시 작업 공간
  * SAS는 데이터셋 전체를 한번에 읽지 않음
  * 한 행씩 PDV에 불러와서 코드 실행 → 출력 데이터셋에 기록 
  → PDV는 "메모리 속 임시 테이블 한 줄"

* PDV에 들어오는 것 : 3가지
  1) SET / MERGE로 읽어온 변수들 
  2) 새로 계산한 변수들 (ex)x = y + 1; 같은 로직에서 생긴 변수
  3) 자동 변수 : _N_ _ERROR_ (출력 데이터셋엔 저장되지 않음).

* PDV의 동작 흐름 (핵심)
  1) PDV 초기화  : PDV의 모든 변수 = missing(.) 로 초기화
  2) SET이나 INPUT으로 1행 읽어서 PDV 채우기
  3) DATA step의 로직 실행  : 계산, 조건문, IF문, LENGTH, FORMAT… 모두 PDV에서 처리됨
  4) OUTPUT (기본 자동 출력) : 현재 PDV의 값 → 출력 데이터셋에 1행 저장됨
  5) PDV 내용이 초기화되고 다음 행으로 이동
  6) 이 과정이 데이터가 끝날 때까지 반복

* PDV 이해하면 풀리는 개념들
  1) 왜 IF와 WHERE가 다르게 작동하는지 
    - WHERE: 데이터 읽기 전에 필터 → PDV에 들어오지도 않음 
    - IF: PDV에 들어온 뒤 제거됨
  2) RETAIN이 뭔지
    - PDV는 매 루프마다 초기화 BUT, RETAIN을 쓰면 초기화되지 않고 값이 유지
  3) MERGE 시 BY 값이 반복되면 왜 문제가 생기는지
    - PDV에 여러 소스가 동시에 들어와 정렬이 필요해서
  4) OUTPUT을 중간에 두 번 쓰면 왜 두 줄이 생기는지
    - PDV 상태를 여러 번 기록하기 때문

```
data t1;
input name $ weight;
datalines;
	a 60
	b 55
	c 40
;
run;

data t2;
	set t1;
	retain cum;
	cum+weight;
	output;
	if weight > 50 then do;
		weight = weight - 10;
		output;
	end;
run;
```

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
	getnames=yes;                                           /* 첫행을 칼럼명으로 사용 */
run;                                                       /* PROC IMPORT 실행 */

/* Excel 불러오기 */
proc import datafile='D:\sas_study\library\students.xlsx'
	out=work.students_3
	dbms= xlsx replace;
	getnames=yes; * default가 yes;
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

```
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
* 
#### **[ 핵심 포인트 ]** 

* OUT= 옵션 → 원본 보존하면서 새로운 데이터셋 생성
* NODUP vs NODUPKEY:
  * NODUP: 모든 변수 기준 중복 제거
  * NODUPKEY: BY 변수 기준 중복 제거
* DUPOUT= : 중복된 값들만 따로 뽑아서 저장하는 옵션
  * out과 dupout의 결과물 합이 전체 데이터셋

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
proc sort data=sashelp.class 
      out=nodupes_age
      dupout=dupes
      nodupkey;
    by age;
run;



```

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
```
data num_fn;
	x = 135.756;
/* 
반올림 : round(x, 0.1) 
올림 : ceil(x) -> 단일 인자
내림 : floor(x) -> 단일 인자
*/
	round_10 = round(x, 10);
	round_1 = round(x, 1);
	round_01 = round(x, 0.1);

	ceil_ex = ceil(-3.5); * -3;
	floor_ex = floor(-3.5); * -4;

/* 정수형 : int */
	intpart = int(x);

/* 난수생성 : rand("분포", ~) */
	random_uni = rand("uniform"); * 기본 균등분포 (0 ~ 1);
	random_normal = rand("normal", 0, 1); * 정규분포 (평균=0, 표준편차=1);
	random_binom = rand("binomial", 0.3, 10); * 이항분포 (n=10, p=0.3);
	random_pois = rand("poisson", 5); * 포아송분포(λ=5);
	random_range = 1 + rand("uniform") * 99; * 범위 지정 랜덤 숫자 (예: 1~100);

/* 크기 비교 : smallest(n, of a b c) : a b c 중 n번째로 큰 값 / largest(n, of x1-x3) : x1 x2 x3 중 n번째로 큰 값 */
	x1 = 2;
	x2 = 5;
	x3 = 1;
	a = 6;
	b = 3;
	c = 10;
	
	s = smallest(1, of a b c);
	l = largest(2, of x1-x3);
	
run;
```
#### **[ 날짜 함수 ]**
```
data date_fn;

/* 기본 날짜 함수
오늘 날짜 : today(), date() -> format date9.
현재 시간 : time() -> format time8.
현재 날짜 + 시간 : datetime() -> format datetime20. 
요일(1=일 ~ 7=토) : weekday(date)
연월일 추출 : year(date), month(date), day(date)
날짜 생성 : mdy(m,d,y)
시간 생성 : hms(h,m,s)
*/

	today_date = today();
	today_date2 = date();
	now = datetime();
	only_time = time();
	year_v = year(today_date);
	
	format today_date today_date2 date9. now datetime20. only_time time8.;
	
/* mdy(month, day, year) -> 월/일/연도로 SAS 날짜 만들기 */
	bday = mdy(10,9, 1996);
	format bday date9.;
	put bday; /* 출력: 09OCT1996 */
	time_n = hms(09,52,00);
	format time_n time8.;
	put time_n;
	
/* intcx("qtr/month/year/day", start_date, end_date) */   
    years = intck('year', bday, today_date);  /* 연도 차이 */
   	years_passed = intck("year", '01jan2000'd, today_date);
    months = intck('month', bday, today_date);  /* 월 차이 */
    days = intck('day', bday, today_date); /* 일수 차이 */
   
/* intnx("qtr/month/year/day", start_date, n, 'b/e/same/middle') */   
    next_month_same = intnx('month', today_date, 1, 'same');   /* 다음달 같은 날짜 */
    next_month_beg = intnx('month', today_date, 1, 'b');   /* 다음달 첫날 -> 기본값 */
    next_month_end = intnx('month', today_date, 1, 'e');   /* 다음달 마지막날 */
    next_month_mid = intnx('month', today_date, 1, 'middle');   /* 다음달 중간날짜 -> 31일 있는 달이면 16일 */

	format next_month_same next_month_beg next_month_end next_month_mid date9.;

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
* 숫자: SUM, MEAN, ROUND, INT, RAND, SMALLEST, LARGEST, ROUND
* 날짜: MDY, TODAY, DATE, TIME, YEAR, QTR, MONTH, DAY, INTCK, INTNX, YRDIF

### 2-7. Convert character ↔ numeric

#### **[ 개념 ]**  
* INPUT(문자→숫자) : input(char_var, format.)
* PUT(숫자→문자) : put(num_var, format.)

#### **[ 예제 ]** 
```
data convert;
    char_num = "123";
    real_num = input(char_num, 8.);     /* 문자 → 숫자(8자리, 정수형) */

    num_val = 2025;
    str_val = put(num_val, 4.);         /* 숫자 → 문자(길이를 정해줌) */
run;
```

### 2-8. Process data using DO loops

#### **[ 개념 ]**  
* DO i = 1 to n; END; → 반복 loop.
* DO WHILE(condition); END; : TRUE이면 일해라 -> 최초 1회 실행 보장하지 않음
* DO UNTIL(condition); END; : 일단 시작하고 TRUE면 그만해 -> 최초 1회 실행 무조건

#### **[ 예제 ]** 
```
data loop_ex;
	do i=1 to 10;
		square = i*i;
		output;
	end;
run;

data loop_ex2;
	i = 1;
	do while(i<10); * i가 1부터 9일때까지 돌아감;
		square = i*i;
		output;
		i = i + 1; * 순서 중요 : output 전에 i가 들어가면 첫행에 square은 1, i는 2 이렇게 찍힘;
	end;
run;

data loop_ex3;
	i = 1;
	do until(i>10); * i가 1부터 10일때까지 돌아감;
		square = i*i;
		output;
		i = i + 1;
	end;
run;

/* SAS의 DO UNTIL 반복문은 루프 내부의 문장이 실행된 후에 조건을 루프의 마지막에서 확인하기 때문에, 최소한 한 번은 실행된다 */
data loop_ex4;
	i = 1;
					/* i는 10보다 작기 때문에 조건은 TRUE여서 반복문 종료이지만, 
					until은 조건을 마지막에 확인해서 반복문이 최초 1회는 실행되어야함*/
	do until(i<10);
		square = i*i;
		i = i + 1;
		output;
	end;
run;
```

### 2-9. Restructure with PROC TRANSPOSE

#### **[ 개념 ]**  
* Wide ↔ Long 변환 : 행과 열의 변환

#### **[ 핵심 포인트 ]** 
* ID → 열 이름 생성: ID 변수는 반드시 고유값(unique)이 있어야 깔끔하게 나옴(고유값이 아니면 같은 칼럼명 충돌)
* VAR → 회전할 값 선택: 여러 개 VAR 지정하면 행이 여러 줄로 생성
* NAME 자동 생성: 원래 VAR 변수명이 자동으로 _NAME_에 들어감 
  * 변경하고 싶다면 name = option 사용. _name_의 라벨은 name of former variable 
* NAME 자동 생성: 원래 VAR 변수명이 자동으로 _NAME_에 들어감
* BY 문 사용 가능: 그룹별로 transpose 가능(단, 미리 PROC SORT 필요)
* Wide → Long은 proc transpose 1번
* Long → Wide는 id group 등을 적절히 이용해 해결 가능

#### **[ 예제 ]** 
```

proc sort data=sashelp.class out=class; by sex; run;
proc transpose data=class out=transposed prefix=col_ name=varname; 
	by sex;
run;

/* LONG -> WIDE 버전 */

proc sort data=sashelp.class out=class; by sex; run;

proc transpose data=class out=transposed prefix=col_; 
	by sex;
    var height weight;
    id name; 
run;

/* WIDE -> LONG 버전 */

proc sort data=sashelp.class out=class; by name sex age; run;
proc transpose data=class out=class_trans(rename=(_NAME_=hw)) prefix=var; 
	by name sex age; 
	var height weight; 
run;

proc sort data=sashelp.cars nodupkey out=cars; by drivetrain make; run;
proc transpose data=cars out=transposed prefix=o_; 
	by drivetrain make;
    var invoice;
    id origin; 
run;

/* notsorted 정렬 없이. 그냥 연속된것들을 그룹으로 침 */
proc transpose data=sashelp.cars out=transposed; 
	by drivetrain make notsorted;
    var invoice;
run;

```

### 2-10. Macro variables

#### **[ 개념 ]** 
* 코드 재사용성 향상

#### **[ 핵심 포인트 ]** 
* %LET으로 매크로 변수 정의
  * 매크로변수를 만들때 큰따옴표를 넣는 것보다, 코드 내 필요할때마다 추가하기 (권장)
* 호출 시 &변수명. → . 붙이면 다른 문자와 구분 가능
  * " " (큰따옴표) → 매크로 변수 &var. 치환됨 (중요)
  * ' ' (작은따옴표) → 매크로 변수 그대로 문자 &var.로 인식됨
* 매크로 관련 옵션
  * MPRINT	: 매크로가 실행될 때, 치환된 실제 SAS 코드를 로그에 보여줌
  * SYMBOLGEN : 매크로 변수(&var)가 어떤 값으로 치환되는지 로그에 표시
  * MLOGIC : 매크로 로직(조건문, %IF/%DO 등)의 흐름을 로그에 표시 

#### **[ 예제 ]** 

```
option mprint symbolgen mlogic;

/* 방법1 : 매크로변수 설정 + data 스텝 이용 -> symbolgen 작동 */

%let gender = M;

data class_&gender.;
    set sashelp.class;
    if sex = "&gender.";
run;

/* 방법2 : 매크로코드 이용 -> mprint, symbolgen 작동 */
%macro filter_class(gender);
    data class_&gender.;
        set sashelp.class;
        if sex = "&gender.";
    run;
%mend;

%filter_class(M);

/* 예제 : 매크로변수 안에 매크로변수 사용 */
%let titleX = PROC MEANS Of Only &cyl_cnt. Cylinder Vehicles; * title 안에 cyl_cnt 있음;
%let cyl_cnt = 12;

title "&titleX"; * 문자열 매크로 변수는 큰따옴표 필수;
proc means data=sashelp.cars;
	where cylinders=&cyl_cnt;
	var msrp;
run;



```


## Part 3 – Error Handling (15–20%)
#### **[ 학습 목표 ]** 
* 논리 오류(program logic bugs) 파악 및 해결
* 문법 오류(syntax errors) 식별 및 수정
* SAS 로그 활용 능력
#### **[ 출제 포인트 ]** 
* 흔한 문법 실수(세미콜론 누락·따옴표 불일치·예약어 오용 등)
* 로그의 NOTE/WARNING/ERROR 의 해석법
  * NOTE = 정보 메시지 (예: 변수 길이 잘림, 자동 형변환, missing value 등) 
  * WARNING = 주의사항 : 실행은 되었지만 잠재적 문제 (예: BY 변수 정렬 안 됨)
  * ERROR = 실행중단 원인 : 실행 불가 (예: 옵션 잘못됨, 데이터셋 없음)
* PUTLOG, _N_, _ERROR_ 같은 디버깅 도구 사용법

### 3-1. Identify and resolve programming logic errors
#### **[ 개념 ]** 
* 문법은 맞지만 원하는 출력이 나오지 않는 오류.
* 주된 원인 : 조건식 / 데이터 타입 / 필터링 위치(IF vs WHERE) / 자동형변환이 주된 원인
* DATA step 반복 횟수(_N_)와 오류 플래그(_ERROR_)로 디버깅 가능
* PUTLOG 활용 : DATA step 내부에서 변수값과 메시지를 로그에 출력할 때 사용(조건부 출력 가능)
* 매크로 치환/흐름 추적 : OPTIONS MPRINT SYMBOLGEN MLOGIC -> 2-10 참고

#### **[ 핵심 포인트 ]** 
* _N_ : DATA step 반복 횟수
* _ERROR_ : 오류 발생 시 1
* 자동형변환은 NOTE로 표시됨 → 반드시 INPUT/PUT 사용
* WHERE vs IF
  * WHERE: 데이터 읽기 전에 필터 / 프로시져(proc) 에서도 사용 가능
  * IF: 읽은 후 필터 → 결과가 달라질 수 있음(논리 오류) -> putlog 로 디버깅 / data step에만 사용 가능

#### **[ PUTLOG 예제 ]** 
```
/* 
 putlog "문구삽입 가능" 
 putlog var=; : 변수명과 값 출력
 putlog _all_; : 모든 변수 출력
 if … then putlog …; : 조건부 디버깅
 문자/숫자 형식 지정 가능 (putlog age 4.2)
*/
data debug;
  set sashelp.class;
  putlog "DEBUG:" _N_= name= age=;
  if sex="M" then putlog name= "weight=" weight 10.2 ; * format 형태 출력 가능;
run;

/* 로그창 결과: 
     DEBUG:_N_=1 Name=Alfred Age=14
     Name=Alfred weight=    112.50
     DEBUG:_N_=2 Name=Alice Age=13
     DEBUG:_N_=3 Name=Barbara Age=13
     DEBUG:_N_=4 Name=Carol Age=14
     DEBUG:_N_=5 Name=Henry Age=14
     Name=Henry weight=    102.50
     DEBUG:_N_=6 Name=James Age=12
*/

/* putlog _all_ : 해당 행의 모든 변수와 값, ERROR, N 모두 출력 */
data test;
  set sashelp.class;
  if _n_ <=5 then putlog _all_;
run;

/* 로그창 결과: 
     Name=Alfred Sex=M Age=14 Height=69 Weight=112.5 _ERROR_=0 _N_=1
     Name=Alice Sex=F Age=13 Height=56.5 Weight=84 _ERROR_=0 _N_=2
     Name=Barbara Sex=F Age=13 Height=65.3 Weight=98 _ERROR_=0 _N_=3
     Name=Carol Sex=F Age=14 Height=62.8 Weight=102.5 _ERROR_=0 _N_=4
     Name=Henry Sex=M Age=14 Height=63.5 Weight=102.5 _ERROR_=0 _N_=5
*/

```

### 3-2. Recognize and Correct Syntax Errors
#### **[ 개념 ]** 
* 프로그램이 실행조차 되지 않는 오류 -> SAS가 "ERR0R:"로 중단함
* SAS 주요 문법 규칙
  * 모든 문장은 **세미콜론(;)**으로 끝남
  * 변수명: 최대 32자, 알파벳/숫자/언더스코어 사용 가능
  * 예약어는 변수명으로 불가 (예: data, set)
  * DATA step과 PROC에는 RUN; / QUIT;

* 자주 발생하는 Syntax Error 
  * 세미콜론 누락 
  * 옵션 철자 오류 (예: DELIMTER → DELIMITER)
  * 큰따옴표/작은따옴표 짝이 안 맞음

#### **[ 로그 해석 예시 ]** 
1. 세미콜론 빠짐(문법 오류)
```
/* 잘못된 코드 */
data bad
  set sashelp.class;
run;
/* (에러 로그) ERROR 56-185: DATASTMTCHK=COREKEYWORDS 옵션이면, SET은(는) DATA 문에서 허용되지 않습니다. 
DATA 문에서 찾을 수 없는 세미콜론을 확인하거나 DATASTMTCHK=NONE을 사용합니다. */

/* 수정 */
data good;
  set sashelp.class;
run;
```
2. 문자와 숫자 자동 변환(논리 오류)
```
/* 비권장 방식 */
data conv;
  set sashelp.class;
  /* age는 numeric인데 "15" 문자와 비교하면 automatic conversion 발생 */
  if age = "15" then flag=1; /* 로그에 NOTE 나옴 - 비권장 */
run;

/* 로그에 NOTE가 나오면서 자동형변환(char -> num) 하여 실행 */
NOTE: 다음의 위치에서 문자형 값이 숫자형 값으로 변환되었습니다. (행):(칼럼). 72:12   
NOTE: 19개의 관측값을 데이터셋 SASHELP.CLASS.에서 읽었습니다.
NOTE: 데이터셋 WORK.CONV은(는) 19개 관측값과 6개 변수를 가집니다.

/* 명확한 방식 (권장) */
data conv_fix;
  set sashelp.class;
  if age = input("15",8.) then flag=1; /* 혹은 if age = 15; */
run;
```
3. 등호 빠짐(문법 오류)
```
 69         proc print data sashelp.class;
                            _____________
                            73
 ERROR 73-322: =이(가) 요구됩니다.
 70           var name age;
 71         run;
```

## Part 4 – Generate Reports and Output (15–20%)
#### **[ 학습 목표 ]**
* PROC PRINT, PROC FREQ, PROC MEANS, PROC UNIVARIATE 활용
* 라벨, 포맷, 타이틀, 푸터 등 보고서 꾸미기
* ODS (Output Delivery System) 활용해서 HTML/PDF/XLSX/RTF/PPTX 등으로 출력
  * ods pdf file="x.pdf"; ODS CLOSE 잊지 말기
* 데이터 Export
  * proc export data=... outfile="..." dbms=csv replace; run; 또는 libname xlsx로 영구저장
* 출력 포맷과 데이터형 구분: FORMAT은 표시(출력)용, INFORMAT은 읽기용. 실제 값은 변하지 않는다

#### **[ 핵심 포인트 ]**
* PROC PRINT: 행·열을 그대로 보고, VAR, ID, WHERE, LABEL, NOOBS, DOUBLE, SUM 등 자주 사용.
* PROC FREQ: 1-way/2-way 빈도표. TABLES var*var / nocol norow nopercent; 옵션으로 표 형태 제어. NLEVELS, ORDER= 등.
* PROC MEANS / SUMMARY: 평균·합계·표준편차 등. CLASS vs BY 차이, OUTPUT OUT=로 결과 데이터셋 생성.
* PROC UNIVARIATE: 분포·이상치·기술통계(백분위·정규성검정).
* PROC FORMAT: VALUE·CNTLIN=로 사용자 포맷 정의 → 보고서 가독성 향상.

### 4-1. Generate List Reports(PROC PRINT)
#### **[ 개념 ]**
* 기본 보고서(List report) 생성
* 변수 선택/순서/라벨/합계 등 커스터마이징 가능

#### **[ 주요 옵션 ]**
* VAR: 출력 변수 선택/순서 조정
* WHERE: 조건부 출력
* SUM: 합계
* ID: ID 변수 지정
* BY: 그룹별 출력
* NOOBS: 관측치 번호 제거
* LABEL: 변수 라벨 사용

#### **[ 예제 ]**
```

* name을 id로 취급하는지, var로 취급하는지에 따라 색이 다름;

proc print data=sashelp.class label noobs;
/* 	id name; */
	var name sex age height weight;
	label height="Height(cm)";
	sum height weight;  * 총합계 표시됨;
	where sex="F";
RUN;


proc print data=sashelp.class label noobs;
	id name;
	var name sex age height weight;
	label height="Height(cm)";
	sum height weight;
	where sex="F";
RUN;

```

### 4-2. Generate Summary Reports (PROC FREQ, PROC MEANS, PROC SUMMARY, PROC UNIVARIATE)

#### [ PROC DELETE : 데이터셋 지우기 ]
* 한번에 한 개만 지울 수 있음
```
proc delete data=work.mydata;
run;
```

#### [ PROC MEANS : 수치형 변수의 기본 통계량 요약 ]

* 평균(mean), 합계(sum), 최소/최대(min/max), 표준편차(std), 개수(n) 등 수치형 요약
* 주요 사용 옵션
  * class : 그룹별 요약
  * var : 분석할 변수
  * ways n1 n2 n3 : 그룹 조합 수준 지정
  * nway : CLASS 조합 중 가장 구체적인 조합만 출력
  * maxdec= 소숫점 자리수
  * output out= : 결과를 데이터로 저장
  * noprint : 결과물 안 나오게 설정
  * autoname : 자동변수명 생성(ex. var1_mean) -> / autolabel도 있음

```
/* 예제1) 문자열 변수들의 통계량 추출 */
proc means data=sashelp.class n mean std min max;
	class sex;
	var _numeric_; * _all / _character_ / _numeric_ ;
run;

/* 예제2) 제조국(Origin), 구동방식(DriveTrain) 조합별
      평균/최소/최대/표준편차를 계산, WAYS로 조합 단계 제어, 출력 데이터셋 저장 */
      
proc means data=sashelp.cars n mean std min max;
	class origin drivetrain make;
	ways 2 3; * 2개씩 뭉쳐진 조합과, 3개씩 뭉쳐진 조합만 출력;
	var mpg_city mpg_highway weight;
	output out=car_summary
		mean= std= min= max= / autoname; * 생성되는 데이터에 실릴 집계값들; * 
run;

/* 예제3) 나이 그룹(10~13, 14~16)으로 포맷을 만들고, 성별별(weight 기준) 최고치 뽑아내기 */

proc format;
    value agegrp
        10-13 = '10-13'
        14-high = '14+';
run;

proc means data=sashelp.class nway noprint;
    where age >= 12;              /* 조건 필터 */
    class sex age;
    format age agegrp.;           /* 포맷으로 그룹 조정 */
    var weight;
    output out=class_max max(weight)=max_wt / autoname;
run;


/* 예제4) 가중평균 - 판매량을 가중한 inventory와 returns의 평균값 */
proc means data=sashelp.shoes mean; *std도 가능하지만 너무 심화버전;
	class product;
	var inventory returns;
	weight sales;
run;

/* 예제5) 여러 OUTPUT문으로 다른 통계 2세트 생성 / where 조건문 */

proc means data=sashelp.cars noprint;
	where cylinders>=6;
	class type origin;
	var msrp invoice;
	
	output out=m1 mean= / autoname; * m1 에 평균값 저장;
	output out=m2 median= / autoname;  * m2 에 중앙값 저장;
run;

* 결과 데이터셋에는 _TYPE_ , _FREQ_ 칼럼이 자동 생성됨
  _TYPE_ = 0 : 전체 / 1 : origin만 / 2 :  type만 / 3 : origin*type 조합 ;
  
/* 예제6) 멀티라벨 포멧 사용(mlf 옵션) */

proc format;
	 value mlf_age (multilabel) * 범위가 겹치면 (multilabel) 꼭 표시;
	 0-12 = 'Child'
	 13-18 = 'Teen'
	 15-25 = 'Youth';
run;

proc means data=sashelp.class n mean;
	class age / mlf;
	format age mlf_age.; * 15~18세는 teen, youth 중복으로 들어가서 계산됨;
	var height;
run;



```

#### [ PROC SUMMARY : PROC MEANS와 거의 동일하지만 ‘출력 안 나오는’ 버전 ]
* proc means의 noprint 버전
* 집계 결과만 데이터셋으로 생성하는 데 최적화, 성능이 빠르고 빅데이터 집계에 적합
* 엑셀 피벗테이블처럼 "그룹별 합계 데이터셋" 만들 때 자주 사용
* 주요 사용 옵션
  * nway : class 조합의 "완전 매칭" 집계만 출력
  * ways : 다단계 집계
  * output out= : 결과 저장
  * print : 결과 화면 출력(기본값이 noprint)

```
/* 예제1) 가장 상세한 조합으로 sum data를 내보내기 */

proc summary data=mydata nway;
  class sido item;
  var prem1 prem2 prem3;
  output out=sum_prem sum=;
run;

/* 예제2) 2개씩의 조합으로 mean data 내보내기 */

proc summary data=sashelp.cars;
  class origin type drivetrain;
  var msrp;
  ways 2;
  output out=way1 mean= / autoname;
 run;
 
/* 예제4) format + types로 세밀 계층 제어 + output 2개 */
proc format;
	value classfmt
		low-14 = 'junior'
		15-high = 'senior';
run;

proc summary data=sashelp.class;
	class sex age ;
	format age classfmt.;
	var height weight;
	types age age*sex;
	output out=fmtstat_m1 mean= / autoname;
	output out=fmtstat_m2 median=/autoname;
run;
```

#### [ PROC FREQ : 범주형 데이터 요약(빈도표) ]
* 범주형 변수의 빈도, 비율, 크로스탭(교차표) 생성
* 데이터 검증용으로 많이 씀(고유값 확인, 이상값 탐지)
* 1-way(일원) / 2-way(이원) / n-way 교차표 가능
* 연산이 매우 빠름
* 주요 사용 옵션(옵션은 보통 슬래시 치고 쓴다)
  * table A; : 단일 변수 빈도
  * table A B; A, B 각각 빈도표
  * table A*B; : 교차표
  * norow nocol nopercent : 비율 제거(순수 count만 보기 좋음)
    * Row Pct : 행 기준 비율 / Col Pct : 칼럼 기준 / Percent : 매트릭스 기준
  * nlevels : 고유값 개수만 요약(중요)
  * order=???(ex. freq) : 정렬 기준 변경
  * missing : 결측치도 하나의 카테고리로 취급

```
/* 예제1) nocol norow nopercent 옵션 : A*B 형태로 볼때 사용 */
proc freq data=sashelp.class;
  tables sex*age / nocol norow nopercent;
run;

/* 예제2) order=freq, list 옵션 */
proc freq data=sashelp.cars order=freq;
	tables type*drivetrain / list; * 기본형이 매트릭스인데, 리스트로 나열;
run;

/* 예제3) 출력하기 */
proc freq data=sashelp.cars;
	tables type*origin drivetrain*type / out=freq_out; * 교차표는 2개지만, 출력물은 마지막 교차표만!;
run;	

/* 예제4) nocum nopercent 옵션 */	
proc freq data=sashelp.class;
	tables age / nocum nopercent; * 누적빈도, 누적비율 안나옴;
run;

/* 예제5) 결측값 확인 nlevels 과 missing 옵션 */
data tmp; set sashelp.class; if _n_ < 3 then sex=""; run; * 임의로 null 값 생성;

proc print data=tmp; run;

proc freq data=tmp; tables sex; run;
	
proc freq data=tmp; tables sex / missing; run;

proc freq data=tmp nlevels; tables sex; run;

/* 예제6) weight으로 가중 빈도 계산(proc means sum과 동일함) */
proc freq data=sashelp.class;
	tables sex;
	weight height;
run;

proc means data=sashelp.class sum;
	class sex;
	var height;
run;

/* 예제7) 포맷(FORMAT)을 적용한 그룹 빈도 */
proc format;
	value agefmt low-13='A' 14-high='B'; 
run;

proc freq data=sashelp.class;
	tables age ;
	format age agefmt.;
run;
```

#### [ PROC UNIVARIATE : 단일 변수의 상세 통계, 분포, 이상값 및 누락치 탐지 ]
* 평균, 표준편차, 백분위수, 비대칭도(Skewness), 첨도(Kurtosis), 극단값 등 확인
* 분포 형태를 보고 이상/극단값((etreme obs) 탐지
* missing 개수, 5-number summary, normality test(shapiro-wilk 등)
* 주요 옵션
  * histogram : 히스토그램
  * qqplot : 정규 Q-Q 플롯
  * ods select : 출력 제어(시험 출제)

```
proc univariate data=sashelp.class;
  var height;
  histogram height / kernel; * kernel : 히스토그램의 선 그리기;
  qqplot height / normal(mu=est sigma=est);
run;

/* 다양한 옵션 : normal / plot / mu0=x / cipctldf
normal : 정규분포 검정
plot : Q-Q plot과 BoxPlot
mu0=x : 평균값 검정. 만약 이 옵션이 없으면 default는 mu0=0
cipctldf : (옵션 없으면, 각 분위수의 값만 표시) 분포 가정 없이 rank 기반 분위수의 95% 신뢰구간 표출 */

proc univariate data=sashelp.cars normal plot mu0=20 cipctldf;
	var mpg_city;
	output out=univ_mpg;
run;

/* 그룹별 상세통계 출력 : sort+by / class 둘 다 가능 */
proc sort data=sashelp.cars out=cars; by type; run;
proc univariate data=cars; by type; var length; run;

proc univariate data=sashelp.cars; class type; var length; run;

/* 출력1 ) output out=data_name val=vn; */
proc univariate data=sashelp.cars normal plot mu0=20 cipctldf;
	var mpg_city;
	output out=univ_mpg mean=m; * 지정을 해줘야함;
run;

/* 출력2 ) ods 이용 */
ods output Moments      = univ_moments      /* 평균, 분산 등 */
           BasicMeasures = univ_basic       /* 평균/중앙값 등 */
           Quantiles     = univ_quantiles   /* 분위수 */
           TestsForNormality = univ_normal  /* 정규성 검정 */
           ExtremeObs    = univ_extreme;    /* 최댓값/최솟값 관측치 */

proc univariate data=sashelp.class normal;
    var height;
run;

ods output close;

```

### 4-3. Enhance Reports (PROC FORMAT, Titles, Footnotes, SAS System reporting options 사용)

#### 사용자 정의 포맷 (PROC FORMAT)
* 데이터 값 자체를 바꾸는 것이 아니라 '보여주는 형태'만 바꾸는 도구
* 실제 값은 그대로 유지되며, 출력, 보고서, 라벨링에서 자주 사용
* CNTLIN = 옵션 (연습 충분히 하기)
  * 포맷을 일일이 value로 작성하는 대신 테이블 형태로 포맷 생셩(대량 코드, 범주가 많을 때 사용)
  * 데이터셋에는 fmtname / type / start / end / label 설정 필요
  * HLO='H' (High open) / 'L' (Low open) / 'O' (other)
  * 추가학습 연속형 숫자
```
* 문자형 포멧(value $) / 숫자 범위 포맷;

proc format;
  value $gender 'M'='Male' 'F'='Female';
  value agegrp low-12 = "Child" 
               13-19 = "Teenager"
               20-high = "Adult" ;
run;

proc print data=sashelp.class;
  format sex $gender. 
         age agegrp.;
run;
```
```
* cntlin;
data fmt_src;
  fmtname = 'agegroup'; * 포맷 이름 (문자포맷은 $ 붙임);
  type = 'N'; * N = 숫자, C = 문자 ;
  start = 0; end = 12; label = 'teen1'; output;
  start = 13; end = 15; label = 'teen2'; output;
  start = 16; end = 16; label = 'teen3'; output;
  /* end = 16 : dummy 값은 의미 없음, 
     CNTLIN 방식에서 무한대/오픈엔드(open-ended) 범위를 만들 때는
     HLO='H' (High open) / 'L' (Low open) / 'O' (other) 사용 */     
run;
  
proc format cntlin=fmt_src;
run;

proc print data=sashelp.class;
  format age agegroup.;
run;

```
#### 라벨(label)과 헤더 조정
* LABEL은 변수 자체의 설명(컬럼 헤더)을 바꾸는 기능
* 보고서, PRINT, MEANS, REPORT 등에서 표에 표시되는 이름만 변경
* PROC PRINT에서 LABEL 표시하려면 반드시 옵션 label; 필요
  * 옵션에서 label split='구분자'로 긴 라벨은 줄바꿈 가능
```
proc print data=sashelp.class label split='~';
  label height="키(cm)~주요정보" weight="몸무게(kg)~민감정보"; 
run;
```

#### TITLE / FOOTNOTE

* 1~10까지 여러줄의 제목/주석을 구조적으로 관리, 특정 줄만 변경, 삭제 가능
* title; : 전체 초기화
```
title1 "제목1";
title2 "제목2";
title4 "제목4";
title3 "제목3";
footnote2 "주석2";
title4;
proc print data=sashelp.class label;
run;
```

#### SAS System reporting options
* obs= : 출력할 행 수 제한
* pagesize= : 페이지 길이 지정
* linesize= : 한 줄 너비
* date / nodate : 날짜 출력 여부
* number / nonumber : 페이지 번호 출력 여부
```
options nodate nonumber linesize=120 pagesize=40;

title "Custom Report Settings";

proc print data=sashelp.cars (obs=10) label;
run;

title;
options date number;     * 초기화;
```


### 4-4. ODS (Output Delivery System)
* SAS 출력물을 다양한 형식으로 저장/포맷팅할 수 있는 시스템
* 지원 포맷 : HTML / PDF / RTF(워드) / XLSX / PPTX / CSVALL
* STYLE= 옵션으로 테마 변경 가능(statistical, journal, minimal, htmlblue 등)
* REPORT 뿐만 아니라 PROC PRINT, PROC MEANS, PROC FREQ, PROC SUMMARY, PROC UNIVARITE 등 모든 프로시저와 활용 가능
* 항상 OPEN → PROC → CLOSE 구조 : CLOSE 안 닫으면 파일 손상 가능
* 데이터만 저장하는 PROC EXPORT와 달리, 보고서 형태 그대로 출력하고 스타일 적용이 가능하다

```
/* HTML 파일 생성 */

ods html file= "d:\test2_html.html" style=statistical; 
/* html만 유난히 오류 생김 -> D 드라이브 경로를 절대경로로 인식 못 하고, WORK 밑에 하위 폴더처럼 붙여버림. path를 별도 설정 해주기 */;

ods html path="d:" file="test_html2.html" style=htmlblue;
proc print data=sashelp.class noobs label;
  title "Class Report";
  footnote "end";
run;
ods html close;  * 반드시 close 해야 파일이 완성됨;

/* pdf 파일 생성 */

ods pdf file="d:/test_pdf.pdf" style=journal;
proc means data=sashelp.cars n mean max min;
    class type;
    var mpg_city mpg_highway;
    title "Cars MPG Summary by Type";
run;
ods pdf close;

/* rtf 파일 생성 */

ods rtf file="d:/output.rtf" style=meadow; * style 지정으로 색상/폰트 변경 가능;
proc freq data=sashelp.cars;
    tables type*origin / nocol nopercent;
    title "Car Type by Origin";
run;
ods rtf close;

/* xlsx 파일 생성 */

ods excel file= "d:/test_labels.xlsx" ;
proc print data=sashelp.class noobs label;
run;
ods excel close;

/* CSV 파일 생성 */

ods csvall file="D:/output.csv";
proc print data=sashelp.class label;
run;
ods csvall close;
```

### 4-5. Export Data
* SAS에서 외부 파일로 데이터를 내보내는 방법은 2가지
  * PROC EXPORT -> 간단하고 안정적 / csv, txt, tab, jmp, excel 등 다양한 포맷을 자동 생성 가능
  * DATA step + FILE/PUT을 이용한 수작업 내보내기

#### PROC EXPORT
* data = : 내보낼 sas 데이터셋 지정
* outfile= : 생성할 외부 파일 경로
* dbms= : csv(콤마 구분 텍스트), tab(탭 구분 텍스트), dlm(원하는 구분자 지정), jmp(jmp 통계 프로그램 파일), xlsx(엑셀) 등 파일 형식 지정
  * dlm의 경우 마지막에 delimiter='|' 등 구분자 설정 필요함
* replace : 같은 이름 파일이 있으면 덮어쓰기
* label : 변수명 대신 라벨을 사용
* sheet= : 시트명 지정

```
proc export data=sashelp.class
  outfile="class.csv"
  dbms=csv 
  replace;
run;

proc export data=sashelp.class
  outfile="d:/class.txt"
  dbms=dlm
  replace  label; * 한글이면 라벨에 "" 씌워짐;
  delimiter='|';
run;

proc export data=sashelp.class
  outfile="d:/class.xlsx"
  dbms=xlsx
  replace  label;
  sheet="데이터추출";
run;
```

#### SAS/ACCESS XLSX Engine
* xlsx 엔진 : sas에서 excel을 드라이버 없이 읽고 쓰게 해주는 엔진 / 코드 간단 / 시트명 지정 가능
* clear로 초기화 하기 
  * SAS는 해당 XLSX 파일(혹은 DB, 폴더 등)을 세션 종료 전까지 점유(lock) 함
  * 다른 프로그램에서 접근이 막힐 수 있으며, 엑셀 열기/삭제/이름변경/덮어쓰기 불가
* ods, proc export로 라벨명을 사용할 수 있지만, libname xlsx 엔진으로는 불가능

```
libname myxls xlsx "d:/output.xlsx";
data myxls.classdata; set sashelp.class;  run;
data myxls.cars; set sashelp.cars; run;
libname myxls clear;
```