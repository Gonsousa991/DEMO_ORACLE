# Instructions
- Create account in https://apex.oracle.com/en/
- Request Workspace
- Run the following to create the data model

```sql
create table item(
    item varchar2(25) not null,
    dept number(4) not null,
    item_desc varchar2(25) not null
);

create table loc(
    loc number(10) not null,
    loc_desc varchar2(25) not null
);

create table item_loc_soh(
item varchar2(25) not null,
loc number(10) not null,
dept number(4) not null,
unit_cost number(20,4) not null,
stock_on_hand number(12,4) not null
);


--- in average this will take 1s to be executed
insert into item(item,dept,item_desc)
select level, round(DBMS_RANDOM.value(1,100)), translate(dbms_random.string('a', 20), 'abcXYZ', level) from dual connect by level <= 10000;

--- in average this will take 1s to be executed
insert into loc(loc,loc_desc)
select level+100, translate(dbms_random.string('a', 20), 'abcXYZ', level) from dual connect by level <= 1000;

-- in average this will take less than 120s to be executed
insert into item_loc_soh (item, loc, dept, unit_cost, stock_on_hand)
select item, loc, dept, (DBMS_RANDOM.value(5000,50000)), round(DBMS_RANDOM.value(1000,100000))
from item, loc;


commit;
```
- in the Apex App Builder import the application StockApplication.sql. When opening the application for login will be the same email and password as registered at apex.oracle.com.


**NOTE: If you fail to complete any challenge please still reply with what was the thinking and why you were not able to complete.**


[INFO]
> After finishing the problems, create a public repository in Github, push your code and send us the Github Public URL.
> If for some reason this is not possible, send us a zip folder containing a install script with all the required solution identified.
> The source contains a StockApplication.zip Apex application that can be deployed to better understand the challenge from user perspective.

# Context
Item loc stock an hand represents a snapshot table of stock in a specific moment for all items in all stores/warehouses for a retailer. In scenario where you have an Apex application that enables a view of stock per store/warehouse please consider the following:
 - this application has an very high user concurrency access during the entire day
 - the access to the application data is per store/warehouse
 - one of the attributes that most store/warehouse users search is by dept
 
# Challenge
## Must Have
### Data Model



1. Primary key definition and any other constraint or index suggestion

Answer:

    alter table "WKSP_ORACLEDEMO"."DEPTS" add constraint
    "DEPTS_CON_PK" primary key ( "DEPT" );
    
    alter table "WKSP_ORACLEDEMO"."ITEM" add constraint
    "ITEM_CON" primary key ( "ITEM" );
    
    alter table "WKSP_ORACLEDEMO"."LOC" add constraint
    "LOC_CON" primary key ( "LOC" );
    
    alter table "WKSP_ORACLEDEMO"."ITEM_LOC_SOH" add constraint
    "ITEM_LOC_SOH_FK_ITEM" foreign key ( "ITEM" ) references "ITEM" ( "ITEM" );
    
    alter table "WKSP_ORACLEDEMO"."ITEM_LOC_SOH" add constraint
    "ITEM_LOC_SOH_LOC_FK" foreign key ( "LOC" ) references "LOC" ( "LOC" );
    
    alter table "WKSP_ORACLEDEMO"."ITEM_LOC_SOH" add constraint
    "ITEM_LOC_SOH_PK" primary key ( "ITEM", "LOC", "DEPT" );
    
    CREATE INDEX idx_item_loc_soh_dept ON item_loc_soh(dept);

--If this will be used a lot in the where clause we can also create this:

    CREATE INDEX idx_unit_cost ON item_loc_soh(unit_cost);
    
    CREATE INDEX idx_stock ON item_loc_soh(stock_on_hand);





2. Your suggestion for table data management and data access considering the application usage

Answer:

--Normalize the data in all tables.

--Partition the tables if they contain a lot of rows with the correct partition key.

--Indexing the columns that are used frequently in the where clause. 



3. Your suggestion to avoid row contention at table level parameter because of high level of concurrency

Answer: Indexing and Partitioning




4. Create a view that can be used at screen level to show only the required fields


Answer:

      CREATE OR REPLACE FORCE EDITIONABLE VIEW "SHOW_REQUIRED_FIELDS" ("ITEM", "LOC", "DEPT", "UNIT_COST", "STOCK_ON_HAND") AS 
      select "ITEM","LOC","DEPT","UNIT_COST","STOCK_ON_HAND" from item_loc_soh;





5. Create a new table that associates user to existing dept(s)

--Answer:

        create table DEPTS(
            Dept varchar2(25) not null,    
            Dept_desc varchar2(25) not null
        );
        
        create table Users_APP(
            User_app number(4) not null,    
            Name varchar2(25) not null,
            Dept varchar2(25) not null
        
        );

--We can also create a table to manage this connections like Depts_Users_Con that have an id, the Dept and User_app values.

    create table Depts_Users_Con(
        ID number not null,    
        User_app number(4) not null,    
        Dept varchar2(25) not null
    
    );



### PLSQL Development


6. Create a package with procedure or function that can be invoked by store or all stores to save the item_loc_soh to a new table that will contain the same information plus the stock value per item/loc (unit_cost*stock_on_hand)

Answer:

First create the table:

    create table item_loc_soh_2(
    item varchar2(25) not null,
    loc number(10) not null,
    dept number(4) not null,
    unit_cost number(20,4) not null,
    stock_on_hand number(12,4) not null,
    stock_value number(12,4) not null
    );

Then the package:

    create or replace package "LIB_STOCK" as
    
    Procedure stores_information(p_item varchar2 ,p_dept number , p_loc number, p_all varchar2 default 'NO');
    
    end "LIB_STOCK";
    /
    
    create or replace package body "LIB_STOCK" as
    
    
      procedure stores_information(p_item varchar2 ,p_dept number , p_loc number, p_all varchar2 )
      is
        v_item varchar2(50);
        v_loc number;
        v_dept number;
        v_unit_cost number;
        v_stock number;
        v_stock_value number;
      begin
    
    If p_all = 'NO' then
    
    select item, loc, dept, unit_cost, stock_on_hand, (unit_cost*stock_on_hand) 
    into v_item, v_loc, v_dept, v_unit_cost, v_stock, v_stock_value 
    from item_loc_soh
      where item=p_item
            and dept= p_dept
            and loc = p_loc;
    
    
      insert into item_loc_soh_2(item, loc, dept, unit_cost, stock_on_hand, stock_value) values
      (v_item, v_loc, v_dept, v_unit_cost, v_stock, v_stock_value );
    
    
      elsif p_all= 'ALL' then 
    
      for i in (select item, loc, dept, unit_cost, stock_on_hand from item_loc_soh)
    
      loop
    
    insert into item_loc_soh_2(item, loc, dept, unit_cost, stock_on_hand, stock_value) values
      (i.item, i.loc, i.dept, i.unit_cost, i.stock_on_hand, (i.unit_cost*i.stock_on_hand)  );
    
    
      end loop;
      end if;
      end;
    end "LIB_STOCK";
    /

--If pass the value 'ALL' it is created on the other table with all values.






7. Create a data filter mechanism that can be used at screen level to filter out the data that user can see accordingly to dept association (created previously)

Answer: Don´t understand the question but for data filter mechanism we can use a view to get the important data that we want





8. Create a pipeline function to be used in the location list of values (drop down)

Answer:

-- First create a type:

        CREATE TYPE locs AS OBJECT (
          loc number,
          loc_desc VARCHAR2(100)
        );



  --Then the type table:

        CREATE TYPE locs_table AS TABLE OF locs;
    


-- Then the function:

        CREATE OR REPLACE FUNCTION Locs_List
          RETURN locs_table PIPELINED
        AS
          
        BEGIN
           FOR loc_rec IN (SELECT loc, loc_desc FROM loc)
          LOOP
            PIPE ROW(locs(loc_rec.loc, loc_rec.loc_desc));
          END LOOP;
          
          
          RETURN;
        END;


--The query to list the locs : 

    select * from table(locs_list)
    



## Should Have
### Performance


9. Looking into the following explain plan what should be your recommendation and implementation to improve the existing data model. Please share your solution and the corresponding explain plan. Please take in consideration the way that user will use the app.
```sql

 Plan Hash Value  : 1697218418 

------------------------------------------------------------------------------
| Id  | Operation           | Name         | Rows | Bytes | Cost  | Time       |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |              | 100019 | 40760 | 10840 | 00:00:03 |
| * 1 |   TABLE ACCESS FULL | ITEM_LOC_SOH | 100019 | 40760 | 10840 | 00:00:03 |
------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 1 - filter("LOC"=652 AND "DEPT"=68)


Notes
-----
- Dynamic sampling used for this statement ( level = 2 )

```

Answer: Indexing and Partitioning





 10. Run the previous method that was created on 6. for all the stores from item_loc_soh to the history table. The entire migration should not take more than 15s to run (don't use parallel hint to solve it :)) 
 
Answer: 

--Another way to make it

    PROCEDURE stores_information2 (
      p_item  VARCHAR2,
      p_dept  NUMBER,
      p_loc   NUMBER,
      p_all   VARCHAR2
    ) IS
      TYPE item_loc_soh_type IS TABLE OF item_loc_soh%ROWTYPE;
      v_item_loc_soh   item_loc_soh_type;
      
    BEGIN
      IF p_all = 'NO' THEN
        SELECT item, loc, dept, unit_cost, stock_on_hand
        BULK COLLECT INTO v_item_loc_soh
        FROM item_loc_soh
        WHERE item = p_item
          AND dept = p_dept
          AND loc = p_loc;
    
        FORALL i IN 1..v_item_loc_soh.COUNT
          INSERT INTO item_loc_soh_2 (item, loc, dept, unit_cost, stock_on_hand, stock_value)
          VALUES (v_item_loc_soh(i).item, v_item_loc_soh(i).loc, v_item_loc_soh(i).dept,
                  v_item_loc_soh(i).unit_cost, v_item_loc_soh(i).stock_on_hand,
                  v_item_loc_soh(i).unit_cost * v_item_loc_soh(i).stock_on_hand);
    
      ELSIF p_all = 'ALL' THEN
        SELECT item, loc, dept, unit_cost, stock_on_hand
        BULK COLLECT INTO v_item_loc_soh
        FROM item_loc_soh;
    
        FORALL i IN 1..v_item_loc_soh.COUNT
          INSERT INTO item_loc_soh_2 (item, loc, dept, unit_cost, stock_on_hand, stock_value)
          VALUES (v_item_loc_soh(i).item, v_item_loc_soh(i).loc, v_item_loc_soh(i).dept,
                  v_item_loc_soh(i).unit_cost, v_item_loc_soh(i).stock_on_hand,
                  v_item_loc_soh(i).unit_cost * v_item_loc_soh(i).stock_on_hand);
      END IF;
    END;



-- Another way is to generate an id for the table item_loc_soh and we can do it as following:


    CREATE OR REPLACE PACKAGE lib_stock IS
      PROCEDURE stores_information3(p_item_loc_soh_id IN item_loc_soh.item_loc_soh_id%TYPE DEFAULT NULL);
    END lib_stock;
    /
    
    CREATE OR REPLACE PACKAGE BODY lib_stock IS
      PROCEDURE stores_information3(p_item_loc_soh_id IN item_loc_soh.item_loc_soh_id%TYPE DEFAULT NULL) IS
      BEGIN
        INSERT INTO item_loc_soh_2 (item, loc, dept, unit_cost, stock_on_hand, stock_value)
        SELECT item, loc, dept, unit_cost, stock_on_hand, (unit_cost * stock_on_hand) AS stock_value
        FROM item_loc_soh
    
    -- If is null will get 'all' otherwise get the store with the id that is passed as argument in the procedure
        WHERE p_item_loc_soh_id IS NULL OR item_loc_soh_id = p_item_loc_soh_id;
      END stores_information3;
    END lib_stock;
    /



11. Please have a look into the AWR report (AWR.html) in attachment and let us know what is the problem that the AWR is highlighting and potential solution.


Answer: First time i look at a report like this but for what i can search indicate that the corresponding metrics or statistics have exceeded a certain threshold or are outside the expected range.

For solution maybe Tuning the SQL queries, improving their performance and analyze if the resources of the database are enough in terms of memory and CPU allocation for example.



## Nice to have
### Performance


11. Create a program (plsql and/or java, or any other language) that can extract to a flat file (csv), 1 file per location: the item, department unit cost, stock on hand quantity and stock value.
Creating the 1000 files should take less than 30s.
 
 
Answer:

-- Create directory object 

    CREATE DIRECTORY output_dir AS '<Output Directory>'; 


-- Create the logic to generate a file for location

    DECLARE
      CURSOR c_locations IS
        SELECT DISTINCT loc FROM loc; 
        
      v_file UTL_FILE.file_type;
    
    BEGIN
      FOR rec IN c_locations LOOP
        -- Open the file for writing ( will have one file per location dynamically)
        v_file := UTL_FILE.fopen('OUTPUT_DIR', 'location_' || rec.loc || '.csv', 'w', 32767);
    
        -- Write header 
        UTL_FILE.put_line(v_file, 'Item,Department,Unit Cost,Stock Quantity,Stock Value');
    
        -- Fetch data and write to the file for the one location at a time
        FOR item_rec IN (SELECT item, dept, unit_cost, stock_on_hand, stock_value
                         FROM item_loc_soh_2
                         WHERE loc = rec.loc) LOOP
          UTL_FILE.put_line(v_file,
                            item_rec.item || ',' ||
                            item_rec.dept || ',' ||
                            item_rec.unit_cost || ',' ||
                            item_rec.stock_on_hand || ',' ||
                            item_rec.stock_value);
        END LOOP;
    
        -- Close the file
        UTL_FILE.fclose(v_file);
      END LOOP;
      
      
    EXCEPTION
      WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('An Error has occurred during data extraction: ' || SQLERRM);
    END;
    /

