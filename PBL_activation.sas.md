![SAS](https://upload.wikimedia.org/wikipedia/commons/thumb/1/10/SAS_logo_horiz.svg/320px-SAS_logo_horiz.svg.png)  

# SAS - Analysis of the activations of a new service payment over a year 

This code is in **SQL** language in a **SAS** environment.  
For this reason the SQL code is in between the 
**```proc sql;```** and  **```run;```** statements.  
  
  
## Objective of the analysis:

In early 2020 the company launched a new online service that customers could activate independently for payments service.  
The service had some difficulties in IT releasing, but overall the feedback from customers who had used it was positive.  
At the end of the year it was necessary to understand 
>  - how the **activations** were distributed **over the banks** 
>  - how **activation** were distributed **over customare segmentation**   
>     *the customer are clustered by their year transaction, the higher the transaction the bigger the customare  
>     the analysis confirmed that the product was very appealing to micro merchant (51% of the customer are micro merchant)*  
>  - **how much the service is used after activation**  
>      *the analysis indicates that only the 30% of the merchant used the service after the activation*
>  - **who really uses the service**  
>     *Resturants, Bar, Apparel, Retail (Drugtore, Jewelery,  Florist, Optical, Food (Bakeries , pastry shop, wines and spirit) , Hotel, Doctors and dentist   
>     are the top 10 clients who activated the service  and also used. 
>     Apparel is the best performer, followed Hotel, Food and Retail; unexpected Restaurant and Bar do not use it.*
>  - how was the **region distribution**
>     

  
***


## The code generating the report 

A new structure of the industries was defined by the team, but non still available in the datawarehose.  
So first is necessary to import the new table for decoding the merchant category codes into the new  
industries map. This is done by SAS IDE, importing the excel file into a table named as *mcc_pbl*   


Then from the database are reported the   
- merchants 
- activation date 
- industry code, 
- banks group, 
- region

```sql
PROC SQL;
   CREATE TABLE WORK.PBL AS 
   SELECT DISTINCT 
		  pv.K_PIVA,pv.CO_BOOK,		  
 		  pv.TE_SOC, CO_SIS_FONTE, CO_ESE_PA, CHIAVE_PV,  TE_INSEGNA_ESE, 
		  FL_PBL, TE_STATO_PBL, FL_ESE_INTERNET, FL_ESE_MOTO, 
          mcc.MCC_PBL, pv.co_ese_sicc, pv.TE_ESE_SICC, pv.TE_ESE_MACRO_CTG_2, pv.TE_ESE_CTG_2, co_prov_ese,te_prov_Ese, TE_REG_ESE, TE_ANIELSEN_ese,
		  pv.FL_INSIEME_ACQ_GT_1, pv.FL_INSIEME_ACQ_GT_4A, pv.FL_INSIEME_ACQ_GT_6A, 
    	  piva.TE_M_CLASSE_TRX_GRP_PIVA,pv.CHIAVE_PV_PBL,  
          (datepart( pv.DT_ESE_ATT)) FORMAT=DATE9. AS data_attivazione
      FROM DMCVM_RO.TBCVM_ACQ_PV_DD pv
           LEFT JOIN DMCVM_RO.TBCVM_ACQ_PIVA_DD piva ON (pv.K_PIVA = piva.K_PIVA)
		   left join mcc_pbl mcc on (trim(pv.TE_ESE_SICC)=trim(mcc.TE_ESE_SICC))
       WHERE pv.FL_PBL = 'S'
      order by pv.k_piva,pv.chiave_pv ;

	  /*check that there are no duplicates */
	 select count(chiave_pv) as pv, count(distinct chiave_pv) as distinct_pv from PBL;
QUIT;
```

Retrive transactions, i/e usage of the service  

```sql
PROC SQL;
   CREATE TABLE WORK.PBL_TRX_2020 AS 
   SELECT 
          pos.CHIAVE_PV, 
		  POS.CO_ABI_NEG_ATTUALE AS ABI,
		  POS.TE_ENTE_NEG_ATTUALE AS BANCA_NEG,
            (SUM(VA_MOV_CREDITO_TOT)) FORMAT=COMMAX19. AS CREDITO_TOTALE,
            (SUM(VA_MOV_CONSUMER)) FORMAT=COMMAX19. AS CREDITO_CONSUMER ,
            (SUM(VA_MOV_CORPORATE)) FORMAT=COMMAX19. AS CREDITO_CORPORATE,
            (SUM(VA_MOV_EXTRA_EA)) FORMAT=COMMAX19. AS CREDITO_EEA,
            (SUM(NU_MOV_CREDITO_TOT)) FORMAT=COMMAX11. AS NUM_OPE_CREDITO_TOT, 
            (SUM(VA_MOV_pbt)) FORMAT=COMMAX19. AS PBT 
  FROM WORK.PBL PV
           LEFT JOIN DMCVM_RO.TBCVM_KPI_ACQ_POS_AAMM pos ON (pos.CHIAVE_PV = pv.CHIAVE_PV)
      WHERE pos.TE_AA = '2020'
      GROUP BY    pos.CHIAVE_PV, pos.CO_ABI_NEG_ATTUALE, pos.TE_ENTE_NEG_ATTUALE;

;QUIT;
```

Join product information with transactions

```sql
proc sql;
create table pbl_2020_abi_neg as
select p.*, t.* from pbl p left join PBL_TRX_2020 t on p.chiave_pv = t.chiave_pv
order by p.k_piva, p.chiave_pv;

/*check  that there are no duplicates*/
select count(chiave_pv) as pv, count(distinct chiave_pv) as distinct_pv from pbl_2020_abi_neg;
quit;
```

Gruping by merchants for semplify the report

```sql
PROC SQL;
   CREATE TABLE PBL_2020 AS 
   SELECT K_PIVA,CO_BOOK, TE_SOC, CO_SIS_FONTE,CO_ESE_PA, CHIAVE_PV, TE_INSEGNA_ESE, FL_PBL, TE_STATO_PBL, FL_ESE_INTERNET, FL_ESE_MOTO, 
          MCC_PBL,CO_ESE_SICC, TE_ESE_SICC, TE_ESE_MACRO_CTG_2, TE_ESE_CTG_2, CO_PROV_ESE, TE_PROV_ESE, TE_REG_ESE, 
          FL_INSIEME_ACQ_GT_1, FL_INSIEME_ACQ_GT_4A, FL_INSIEME_ACQ_GT_6A, 
          TE_M_CLASSE_TRX_GRP_PIVA, CHIAVE_PV_PBL, data_attivazione, 
            (SUM(CREDITO_TOTALE)) FORMAT=COMMAX19. AS CREDITO_TOTALE, 
            (SUM(CREDITO_CONSUMER)) FORMAT=COMMAX19. AS CREDITO_CONSUMER, 
            (SUM(CREDITO_CORPORATE)) FORMAT=COMMAX19. AS CREDITO_CORPORATE, 
            (SUM(CREDITO_EEA)) FORMAT=COMMAX19. AS CREDITO_EEA, 
            (SUM(PBT)) FORMAT=COMMAX19. AS PBT
      FROM PBL_2020_ABI_NEG 
      GROUP BY K_PIVA,CO_BOOK,TE_SOC,CO_SIS_FONTE,CO_ESE_PA,CHIAVE_PV,TE_INSEGNA_ESE,FL_PBL,TE_STATO_PBL,FL_ESE_INTERNET,FL_ESE_MOTO,
               MCC_PBL,CO_ESE_SICC,TE_ESE_SICC,TE_ESE_MACRO_CTG_2,TE_ESE_CTG_2,CO_PROV_ESE,TE_PROV_ESE,TE_REG_ESE,FL_INSIEME_ACQ_GT_1,
               FL_INSIEME_ACQ_GT_4A,FL_INSIEME_ACQ_GT_6A,TE_M_CLASSE_TRX_GRP_PIVA,CHIAVE_PV_PBL,data_attivazione;

	select count(chiave_pv) as pv, count(distinct chiave_pv) as distinct_pv from PBL_2020;
QUIT;
```
![Excel](https://upload.wikimedia.org/wikipedia/commons/thumb/7/73/Microsoft_Excel_2013-2019_logo.svg/110px-Microsoft_Excel_2013-2019_logo.svg.png)    

After this , the table is exported in Excel format, and analyzed to create the document below.   

![PowerPoint](https://upload.wikimedia.org/wikipedia/commons/1/16/Microsoft_PowerPoint_2013-2019_logo.svg)



![Activataion](/PBL_20201114_1.png)
![Activataion](/PBL_20201114_2.png)
![Activataion](/PBL_20201114_3.png)

