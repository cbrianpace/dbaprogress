---
layout: post
read_time: true
show_date: true
title:  From Code to SQL -- Mortgage Amortization Schedule
date:   2024-11-25 12:00:00 -0400
description: Can SQL do more than you think?  Take a look at the power of recurssive queries in Postgres.
img: posts/banners/house_640.jpg
tags: [postgres, sql, cte]
author: Brian Pace
---

## Introduction

A recent conversation with a teammate brought me back to my roots as a developer in the mortgage loan industry.  Thinking back to some of my old code, I wondered if it is possible to convert lines of old code used to generate a mortgage amortization schedule into a simple SQL statement.

In my journey in the PostgreSQL world from Oracle, I realized that PostgreSQL is more than a database.  It is a data platform.  PostgreSQL provides a host of features that enable some creative solutions when dealing with data.  To help in my code to SQL challenge the recursive queries feature solves the problem.

## Common Table Expressions (WITH Clause)

All of the database platforms that I have worked with in my career provides the user with the common table expression (CTE) feature, or WITH clause.  Think of the CTE as a temporary table that only lives for the life of the query.  Here is an example:

## Recursive Queries

Recursive queries build on the CTE feature in that there is a non-recursive part (section in blue  in the below example) and a recursive part (section in red).

The non-recursive part is our starting information for our mortgage loan.  In this example, $255,000 loan with an interest rate of 2.49% for 180 months (15 years).  The principal and interest amount is $1,699.11.  This value will only be used for the initial pass thru the query.

The recursive part now is executed continually until no more results are returned (in our case, when the payment number is less than the term (pn < term).  Each pass the remaining balance is updated (bal) based on the amount of principal paid for that scheduled payment (total principal and interest - interest due for that payment cycle).

## Putting it All Together

To see it in action, or maybe run some what if's with your own mortgage, start by calculating the principal and interest payment amount (if unknown).  Here is a query to help you do that:

```sql
SELECT bal, ir, term, 
       round(bal*((ir/12)* 
           (((ir/12)+1)^term))/((((ir/12)+1)^term)-1),2) pni 
FROM (select 255000 bal, .0249 ir, 180 term) x;

bal   |ir    |term|pni    |
------+------+----+-------+
255000|0.0249| 180|1699.11|
```

In the above query, enter the mortgage loan amount for bal ($255,000 in the example), interest rate (ir), and term in months.  The pni column is the principal and interest amount.

Plug the information about the loan into the below query to generate the amortization schedule.

```sql
WITH RECURSIVE t(bal, ir, term, pn, pni, intpmt, prinpmt) AS (
    VALUES (255000::numeric, .0249, 180, 0, 1699.11::numeric, 0::numeric, 0::numeric)
  UNION ALL
    SELECT bal - (pni-(trunc((((bal*ir/360)*30)+.005)*100)/100)) bal, ir, term, pn+1, 
           pni ,
           TRUNC((((bal*ir/360)*30)+.005)*100)/100 intpmt,
           pni-(TRUNC((((bal*ir/360)*30)+.005)*100)/100) prinpmt
    FROM  t        
    WHERE pn <term
)
SELECT pn payment_nbr, trim_scale(prinpmt) principal_payment, 
     trim_scale(intpmt) interest_payment, 
     trim_scale(bal) principal_balance
FROM t      
WHERE pn>0
order BY pn;

payment_nbr|principal_payment    |interest_payment    |principal_balance      |
-----------+---------------------+--------------------+-----------------------+
          1|              1169.98|              529.13|              253830.02|          
          2|              1172.41|              526.70|              252657.61|          
          3|              1174.85|              524.26|              251482.76|          
          4|              1177.28|              521.83|              250305.48|          
          5|              1179.73|              519.38|              249125.75|          
          6|              1182.17|              516.94|              247943.58|          
          7|              1184.63|              514.48|              246758.95|
...
```

Ever wondered how much could be saved if you paid a little extra each month on your mortgage?  Adjust the principal and interest amount in the query to find out.  Take a look at this:

```sql
WITH RECURSIVE t(bal, ir, term, pn, pni, intpmt, prinpmt) AS (
    VALUES (255000::numeric, .0249, 180, 0, 1699.11::numeric, 0::numeric, 0::numeric)
  UNION ALL
    SELECT bal - (pni-(trunc((((bal*ir/360)*30)+.005)*100)/100)) bal, ir, term, pn+1,
           pni ,
           TRUNC((((bal*ir/360)*30)+.005)*100)/100 intpmt,
           pni-(TRUNC((((bal*ir/360)*30)+.005)*100)/100) prinpmt
    FROM  t
    WHERE pn <term
)
SELECT trim_scale(sum(intpmt)) total_interest
FROM t
WHERE pn>0;

total_interest
--------------
      50840.31
      
WITH RECURSIVE t(bal, ir, term, pn, pni, intpmt, prinpmt) AS (
    VALUES (255000::numeric, .0249, 180, 0, 1899.11::numeric, 0::numeric, 0::numeric)
  UNION ALL
    SELECT bal - (pni-(trunc((((bal*ir/360)*30)+.005)*100)/100)) bal, ir, term, pn+1,
           pni ,
           TRUNC((((bal*ir/360)*30)+.005)*100)/100 intpmt,
           pni-(TRUNC((((bal*ir/360)*30)+.005)*100)/100) prinpmt
    FROM  t
    WHERE pn <term
)
SELECT trim_scale(sum(intpmt)) total_interest
FROM t
WHERE pn>0;

total_interest
--------------
      43250.35
```

Look at that! By paying $200 extra each month you save over $7,000 dollars.

The recursive query feature in PostgreSQL allowed me to take lines of code and transform it into a single SQL statement.  Have fun!!!
