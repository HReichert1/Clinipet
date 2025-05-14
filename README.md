# Clinipet
In this project data is loaded to BigQuery n the Google Cloud Platform, cleaned and new tables are created and then loaded to Looker Studio. The data is originiated from the vet clinic Healthtail.

Healthtail provided 3 tables in .csv format 
![image](https://github.com/user-attachments/assets/67af0182-8b14-4c67-ab9d-76c6e514dc5b)

## 1. ETL Plan
- Load the .csv file to a project in BigQuery.
- clean issues in the healthtail_reg_cards table using SQL code (as this is a manually entered table inconsistencies occur) and save the clean data in registration_clean.
- create an aggregated new table to track the monthly movement of medication, i.e. purchases and medication used in clinic. For this task data from invoice and visits have to be joined. The new table is named med_audit.

## 2. Carry out the ETL plan
- Clean data in registration form and create registration_clean
```
SELECT patient_id, owner_id, owner_name, pet_type, 
       ifnull(breed,'Unknown') as breed,
       CONCAT(
		     UPPER(SUBSTRING(patient_name, 1,1)),
		    LOWER(SUBSTRING(patient_name, 2))
	    ) AS patient_name,
      gender, patient_age, date_registration,
      REGEXP_REPLACE(owner_phone, '[^0-9]', '') as owner_phone
FROM `clinipet-459308.Raw.reg_cards` 
```
- create med_audit
```
with stock_out as (
SELECT DATE_TRUNC(visit_datetime,month) as month , med_prescribed as med_name, 
       round(sum(med_dosage),2) as total_packs,
       sum(med_cost) as total_value, 
       'stock out' as stock_movement
FROM `clinipet-459308.Raw.visits` 
group by 1, 2
),

stock_in as (
select month_invoice as month, med_name, round(sum(packs),2) as total_packs,round(sum(total_price),2) as total_value, 'stock in' as stock_movement
from `clinipet-459308.Raw.invoices`
group by 1,2
)


select * from stock_in
union all
select * from stock_out
```
The results table has to be saved as a BigQuery table in the project and dataset.

- Answer reseach questions of Healthtail
```
--- 1. What med did we spend the most money on in total?
Select med_name, sum(total_value) as tot_money_spend
FROM `clinipet-459308.Raw.med_audit` 
where stock_movement ='stock in' ---only purchased meds
group by med_name  
order by 2 desc
limit 1
--- answer: Vetmedin (Pimobendan) with 1035780.0

---2. What med had the highest monthly total_value spent on patients? At what month?
Select med_name, month, sum(total_value) as tot_money_spend
FROM `clinipet-459308.Raw.med_audit` 
where stock_movement ='stock out' ---only meds used in clinic
-- I decided to use stock out in where condition as this is the medication used in patients.  
-- if both stock in and stock out has to be used the where statment has to be removed
group by med_name, month 
order by 3 desc, 1 asc 
limit 1

---answer: 	
--Palladia (Toceranib Phosphate) in 2024-11-01 with 50000

---3. What month was the highest in packs of med spent in vet clinic?
Select month, sum(total_packs) as tot_packs
FROM `clinipet-459308.Raw.med_audit` 
where stock_movement ='stock out' ---only meds used in clinic
-- same problem as above, as only stock out medication is used in clinic. This is again contrary to advice
group by month 
order by 2 desc 
limit 1

---answer: 2024-12-01 with 3861.62 packs

---4. What's on average monthly spent in packs of the med that generated the most revenue?

select med_name, avg(total_packs) as avg_packs 
FROM `clinipet-459308.Raw.med_audit`
where stock_movement ='stock in' and med_name =
     (Select med_name FROM `clinipet-459308.Raw.med_audit` where stock_movement ='stock out' group by med_name  
      order by sum(total_value) desc limit 1)
 
group by med_name 


--- answer: Palladia (Toceranib Phosphate)generates the most revenue (stock-out), avg spent of packs of this med: 108.125
```
##3. Create interactive report in Looker Studio
Registration_clean and visits are loaded to Looker Studio and joined via a left join on patient_id.

On 2 pages the interactive dashboard uses two filters for the users to select the pet type (cat, dog, hamster) and/or the dates which need to get analyzed.

Page 1 describes the population on the top half of the screen and the visit dependent analysis for the diagnoses in pet types and breed as well as the changes of diagnoses over time and the average medication costs over time (only costs of medication are considered which are used in the clinic)
![image](https://github.com/user-attachments/assets/4a3ca07c-342a-4ba0-bfce-60208682c889)

On Page 2 Diagnoses per pet types and breeds, average age of patients and medication costs per pet type are displayed.

![image](https://github.com/user-attachments/assets/575203c4-c8be-47c9-b24a-88c9308aecca)
