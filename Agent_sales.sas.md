![SAS](https://upload.wikimedia.org/wikipedia/commons/thumb/1/10/SAS_logo_horiz.svg/320px-SAS_logo_horiz.svg.png)
![Excel](https://upload.wikimedia.org/wikipedia/commons/thumb/7/73/Microsoft_Excel_2013-2019_logo.svg/110px-Microsoft_Excel_2013-2019_logo.svg.png)
![mail](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f7/Microsoft_Outlook_2013-2019_logo.svg/110px-Microsoft_Outlook_2013-2019_logo.svg.png)
# SAS REPORT VIA EXCEL

This code is in **SQL** language in a **SAS** environment.  
For this reason the SQL code is in between the 
**```proc sql;```** and  **```run;```** statements.  
The other lines of code, out of those statements, are in **SAS** language.  
The code with the **%** is a bit of **SAS Macro** language.

  
  
## Objective of the procedure:

This procedure create a report of onboarderd P.O.S. ( i.e. point of sales terminals ) 
by a network of sales agents of a big Bank .  
After report is created, is then sent in excel format by email.  

The excel  has 4 sheet
***
>    - sheet 1 is the report where for each terminal ID the information detailed are  
>      	
>  		merchant information like *VAT, name, adress, industry*  
>       terminal information like *ID, type, installation/close/change date*  
>       *transaction amount and number* by *month* in a column view mode  
>    - sheet 2 is a summary of total merchant, total store, total terminal , total transaction, total terminal
>    - sheet 3 is a summary of #terminal per store
>    - sheet 4 is a summary of #terminal per merchant
  
***


## The code generating the report 

Firts, in SAS, the code define a structure
for format the months in a more comprehensive way when displayed. 

```sql
proc format library = work ;
	value MeseIt    
		1='January'    
		2='February'    
		3='March'    
		4='April'    
		5='May'    
		6='June'    
		7='July'    
		8='August'    
		9='September'    
		10='October'    
		11='November'    
		12='Dicember'  
		OTHER =' '
;run;   
options  fmtsearch=(work);
```

Loads all terminal that are onboarded by the sales network


```sql
proc sql;
create table a4u as
select * 
	,datepart(dt_pos_att) as attivazione format=date9.
	,co_term_id as termid
	,co_ese_pa as Codice_Punto_Vendita
	,t2.co_canale 
	,t2.te_CANALE from  MKT_RO.TBMK2_ACQ_POS_ISP_DD t1
	LEFT JOIN MKT_RO.TBMK2_TP_CANALE_CONV_ISP t2 ON (t1.ID_TP_CANALE_CONV = t2.ID_TP_CANALE_CONV)
	where t1.ID_TP_CANALE_CONV=14;
quit;
```

Simple check of the number of terminals by type , only for check pourpose

```sql
proc freq data=a4u order=internal ;
	tables te_tipologia_pos  /nocol nocum norow nopercent missing   ;
	format attivazione monyy.
;run;
```

Display number of terminal activated by month/year and channel , only for checking that the channel are the good ones.

```sql
proc freq data=a4u order=internal ;
	tables attivazione*te_CANALE  /nocol nocum norow nopercent missing out=attivazioni ;
	format attivazione monyy.;
;run;
```

Creates a table with the mercant categories codes (mcc) vs. industries.  
This information will be diplayed into the final report 
where the mcc will be substituted by the more comprehensive industries 

```sql
PROC SQL;
   CREATE TABLE WORK.MCC AS 
   SELECT DISTINCT t1.CO_ESE_SICC, 
          t1.TE_ESE_SICC, 
          t1.TE_ESE_SICC_CVM,
		  vert.MACROCATEGORIA_VERTICALI
      FROM DMCVM_RO.TBCVM_ACQ_PV_DD t1
	  left join cvm_acq.br_MCC_VERTICALI vert on vert.mcc=t1.co_Ese_sicc
      ORDER BY t1.CO_ESE_SICC;
QUIT;
```

Table with customer master data and POS info 
like  merchant name/store/address 
like  activation/closing/substitution date  

```sql
proc sql;
	create table work.start_pos as 
		select distinct  t2.co_soc_piva as Cliente__Partita_IVA
				,t2.co_soc_cf as Cliente__Codice_fiscale
				,t2.te_soc as Cliente__Nome_anagrafica
				,t2.co_ese_sicc as Codice_MCC_L3
				,mcc.te_ese_sicc_cvm
				,t2.TE_INSEGNA_ESE as  Insegna
				,t2.te_ind_ese as Punto_Vendita__Indirizzo
				,t2.te_prov_ese as Punto_vendita_PR
				,t1.co_canale as Agent_Code
				,t2.k_piva,t1.CO_ESE_PA,t1.CO_TERM_ID 
  			   	,case 
 					when dwh.te_soluzione_macro is not missing 
					then dwh.te_soluzione_macro 
					else "" end as Tipo_POS
				,t1.co_book
			   ,"" as insieme /*T3.INSIEME*/
			   ,t2.TE_INSEGNA_ESE
			   ,cats( t2.co_sia_ese ,t2.co_sia_stab) as CODICESIA 
			   ,datepart(t2.dt_ese_att) as DT_ESE_ATT 	FORMAT=DDMMYY10.
			   ,datepart(t2.dt_ese_ann) as DT_ESE_ANN 	FORMAT=DDMMYY10.

			   ,DATEPART(t1.DT_POS_ATT) as DT_ATT_POS_GT 		FORMAT=DDMMYY10. /*TBMK2_ACQ_POS_ISP_DD*/
			   ,DATEPART(t1.DT_POS_ANN) as DT_ANN_POS_GT 		FORMAT=DDMMYY10. /*TBMK2_ACQ_POS_ISP_DD*/

			   ,year(datepart(t2.dt_ese_att))  as ANNO_ATT_ESE
			   ,put(month(datepart(t2.dt_ese_att)),MeseIt.) as  MESE_ATT_ESE
			   ,year(datepart(t2.dt_ese_ann)) as ANNO_ANN_ESE
			   ,put(month(datepart(t2.dt_ese_ann)),MeseIt.) as  MESE_ANN_ESE

    		   ,year(datepart(t1.DT_POS_ATT))  as ANNO_ATT_POS_GT
			   ,put(month(datepart(t1.DT_POS_ATT)),MeseIt.) as  MESE_ATT_POS_GT
			   ,year(datepart(t1.DT_POS_ANN)) as ANNO_ANN_POS_GT
			   ,put(month(datepart(t1.DT_POS_ANN)),MeseIt.) as  MESE_ANN_POS_GT
			   ,intck('day',datepart(t2.dt_ese_att), datepart(t1.dt_pos_att)) as gg_delay_ese_pos
		from WORK.A4U t1
				left join MKT_RO.TBMK2_ACQ_MERCHANT_ISP_DD T2 on (t1.CO_ESE_PA = t2.CO_ESE_PA)
				left join DMCVM_RO.TBCVM_ACQ_POS_DD dwh on (dwh.co_term_id =t1.co_term_id) 
				left join MCC mcc on t2.co_Ese_sicc = mcc.co_Ese_sicc
		order by t1.DT_POS_ATT;
quit;
```

Retrive terminals types

```sql
proc sql;
create table tipo_pos as 
	select distinct dwh.co_term_id , dwh.te_soluzione_macro from DMCVM_RO.TBCVM_ACQ_POS_DD dwh , start_pos pos
	where (dwh.co_term_id =pos.co_term_id) 
;quit;

/*check the total terminal by book and merchant, to see if there are inconsistencies between tables*/
proc sql;
create table posA4U_input as select distinct 
	co_book,count(co_Ese_pa) as ese, count (distinct co_Ese_pa) as ese_dist,count(co_term_id) as tid, count (distinct co_term_id) as tid_dist
from a4u group by co_book order by co_book;
create table posA4U_join_merchant as select distinct 
	co_book,count(distinct k_piva) as k_piva_dist, count(co_Ese_pa) as ese, count (distinct co_Ese_pa) as ese_dist,count(co_term_id) as tid, count (distinct co_term_id) as tid_dist
from start_pos group by co_book order by co_book;
;quit;

proc print data=posA4U_input;run; 
proc print data=posA4U_join_merchant;run; 
```

 Create table with terminals transaction  

```sql
proc sql;
	create table work.start_mov as 
		select 	t1.CO_ESE_PA
               ,t1.CO_TERM_ID 
			   ,t1.TE_INSEGNA_ESE
		    ,t1.K_PIVA
			,time.CO_AAMM as mese
			,t2.trx as va_transato
			,t2.num as nu_transazioni
		from WORK.start_pos t1
			/*left join DB.MOVIMENTI_ISP_AAMM_21 t2 on (t1.CO_ESE_PA = t2.CO_ESE_PA) AND (t1.CO_TERM_ID = t2.CO_TERM_ID)*/
			left join CVM_ACQ.BRUNI_ISP_MM_2021 t2 on (t1.CO_ESE_PA = t2.CO_ESE_PA) AND (t1.CO_TERM_ID = t2.CO_TERM_ID)
			left join MKT_RO.TBMK2_TIME time on time.id_aamm =t2.id_aamm
		where co_aamm > 202012 and co_aamm<= 202204
	group by t1.co_ese_pa, t1.co_term_id, t1.te_insegna_ese,t1.k_piva,mese
;quit;
```

Star building code for shifting transaction 

> from row view 
>
>> POS month1 total_transaction  
>> ID1 **month1** total_transaction  
>> ID1 **month2** total_transaction  
>> ID1 **monthn** total_transaction  
> 
> to column VIEW
> 
>> POS  **month1** **month2** **monthn**  
>> ID1  total_trx  total_trx  total_trx   

```sql
proc sort data=start_mov  ;
	by   K_PIVA co_ese_pa  co_term_id mese   ;
;run;

proc transpose data=start_mov out=mov_transp prefix=m;
	by k_piva co_ese_pa co_term_id    ;
	id mese;
	var  nu_transazioni va_transato ;
	format nu_transazioni va_transato  commax20.;
;run;
```

... prepare the COLUMNS sort 

```sql
proc sql;
describe table mov_transp;
;quit;
proc sql ;
   select cats(' ',name)
          into :list_colonne
          separated by ' '
          from dictionary.columns
          where libname = 'WORK' and memname = 'MOV_TRANSP' and upcase(name) like "M%";
quit;

data correggi_colonne;
	colonne_non_ordinate="&list_colonne.";
	array col(50) $20 _temporary_;
	call missing(of col[*]); 
	do i=1 to dim(col) until (p eq 0);
		call scan(colonne_non_ordinate, i,p,l);
		col[i]= substrn(colonne_non_ordinate,p,l);
		end;
	call sortc(of col[*]);
	length colonne_ordinate $150;
	colonne_ordinate =catx(' ' , of col[*]);
	drop i p l;
;run;
```

end column sort.

Start building transactions kpi 

```sql
data _mov _num;
	SET mov_transp;
	if _NAME_='va_transato' then output _mov;
	if _NAME_='nu_transazioni' then output _num;
;run;
```

then retrive transaction and operation columns structure information

```sql
proc sql;
describe table _num;
;quit;
proc sql ;
   select cats(name,'= ope_',substr( name, 2, 6))
          into :list_operazioni
          separated by ' '
          from dictionary.columns
          where libname = 'WORK' and memname = '_NUM' and upcase(name) like "M%";
quit;

proc datasets library = work nolist;
   modify _num;
   rename &list_operazioni;
quit;
proc sql;
describe table _mov;
;quit;
proc sql ;
   select cats(name,'= neg_',substr( name, 2, 6))
          into :list_movimenti
          separated by ' '
          from dictionary.columns
          where libname = 'WORK' and memname = '_MOV' and upcase(name) like "M%";
quit;

proc datasets library = work nolist;
   modify _mov;
   rename &list_movimenti;
quit;
```

Put transaction and operation toghether

```sql
proc sql;
	create table _totale as 
	select a.*, b.* from _mov a inner join _num b on a.k_piva=b.k_piva and a.co_ese_pa=b.co_ese_pa and a.co_term_id=b.co_term_id
	order by k_PIVA, co_ese_pa, co_term_id;
;run;
```

Staring some make up for new table structure  

Retrive sorted operation columns  

```sql
proc sql; select tranwrd(colonne_ordinate,"m","ope_" ) into :lista_operazioni_ordinate from correggi_colonne;quit;
```

Retrive sorted transaction columns

```sql
proc sql; select tranwrd(colonne_ordinate,"m","neg_" ) into :lista_negoziato_ordinate from correggi_colonne;quit;

/*for simple check of the ordered  columns,  printing on console log */
%put &lista_operazioni_ordinate; 
/*for simple check of the ordered  columns,  printing on console log */
%put &lista_negoziato_ordinate; 

data totale_movimenti;
	retain k_piva co_ese_pa co_term_id &lista_operazioni_ordinate &lista_negoziato_ordinate;
	set _totale ;
	drop _name_;
;run;
```

End building code for the column view of transactions

```sql
proc sql;
	create table work.unione as 
		select pos.*, kpi.* 
		from work.start_pos pos
			left join work.totale_movimenti kpi 
            on (pos.CO_ESE_PA = kpi.CO_ESE_PA)  and (pos.CO_TERM_ID = kpi.CO_TERM_ID)  and (pos.k_piva = kpi.K_PIVA)
;quit;
```

If there are terminal with transaction then  ...

```sql
proc sql noprint;   select    count(*)   into   :kpiCount  from   work.totale_movimenti   ;  quit;
```

... create a total sum over all months

```sql
	data POS_A4U;
	set unione ;
	%if &kpiCount. > 0 %then %do; 
		/*if no transaction, skip*/
		totale_negoziato = sum ( OF neg: );
		totale_operazioni = sum (OF ope:);
		if totale_negoziato > 0 then Negozia= 'SI';
							else Negozia= 'NO';
		format totale: commax20.;
	%END;
;run;
```

final step, bring all together

```sql
proc sql  noprint;
	select count (distinct co_ese_pa) into :ese_movimenti  from totale_movimenti;
	select count (distinct co_term_id) into :termid_movimenti  from totale_movimenti;
	select count (distinct k_PIVA) into :merchisp_piva_trovate from POS_A4U;
	select count (distinct co_ese_pa) into :merchisp_coesepa_trovate from POS_A4U;
	select count (distinct co_term_id) into :merchisp_cotermid_trovati from POS_A4U;
;run;
```

Collecting some main values of the report

```sql
data controlli ;
	merchisp_piva_trovate=&merchisp_piva_trovate.; 
	merchisp_coesepa_trovate=&merchisp_coesepa_trovate.; 
	merchisp_cotermid_trovati=&merchisp_cotermid_trovati.; 
	ese_movimenti= &ese_movimenti.;
	termid_movimenti=&termid_movimenti.; 
run;
proc print data=controlli;run;
```

... and finally start to build the excel file with all the reports in separted sheets

```sql
%let fileTimeStamp = %sysfunc(date(), yymmddn8.)%sysfunc(putc(%sysfunc(time(), b8601TM6.), $4.)) ;
%put &fileTimeStamp.;

libname x xlsx "/sasdata_MA/workspace/Nexi_Acq_Workspace/dataroot/Ale/_A4U/a4upos_&fileTimeStamp..xlsx";
%put la versione SAS &sysvlong;
options validvarname=v7;
proc datasets library=work details;
   copy out=x;
      select  pos_a4u CONTROLLI posA4U_input posA4U_join_merchant;
quit;
```

Here send the email with the attached excel report

```sql
filename piccione  email
to= ("alessandro.bruni@nexi.it" "bruni.alessandro@gmail.com")
replyto= ("alessandro.bruni@nexi.it" )
subject= "Report you requested for A4U"
type="text/html"
attach=("/sasdata_MA/workspace/Nexi_Acq_Workspace/dataroot/Ale/_A4U/a4upos_&fileTimeStamp..xlsx" content_type="excel");

data _null_;
file piccione;
put "<ul>Calcolo kpi per DB Agenti.</ul>";
put "<ul>Numero terminali dal canale A4U in TBMK2_ACQ_POS_ISP_DD: <strong>&merchisp_cotermid_trovati.</strong></li></ul>";
put "<ul>Numero punti vendita dal canale A4U in TBMK2_ACQ_POS_ISP_DD : <strong>&merchisp_coesepa_trovate.</strong></li></ul>";
put "<ul>Numero punti vendita transanti dal canale A4U     : <strong>&ese_movimenti.</strong></ul>";
put "<ul>Numero terminali transanti dal canale A4U   : <strong>&termid_movimenti.</strong></ul>";
put "<ul>Numero piva elaborate dalla TBMK2_ACQ_MERCHANT_ISP_DD : <strong>&merchisp_piva_trovate.</strong></ul>";

;run;


quit;

```