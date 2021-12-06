# E-Wallet Service

## Index

- [E-Wallet Service](#e-wallet-service)
  - [Index](#index)
  - [Introduction](#introduction)
  - [Business process and assumptions](#business-process-and-assumptions)
  - [Declare the grain and the thought process behind](#declare-the-grain-and-the-thought-process-behind)
  - [Main component in e-wallet service (Identify the facts and dimensions)](#main-component-in-e-wallet-service-identify-the-facts-and-dimensions)
  - [Data model](#data-model)
    - [ERD](#erd)
    - [Relationships](#relationships)
  - [Questions](#questions)
    - [1) What is the daily balance of credit in the e-wallet service?](#1-what-is-the-daily-balance-of-credit-in-the-e-wallet-service)
    - [2) How much credit will expire in the next month?](#2-how-much-credit-will-expire-in-the-next-month)
    - [3) What is the outcome (i.e. % used, % expired, % left) of credit given in a particular month?](#3-what-is-the-outcome-ie--used--expired--left-of-credit-given-in-a-particular-month)

## Introduction

One of the online retail company’s features is an e-wallet service, that holds credit that can be used to
pay for products purchased on the platform.
Users can receive credit in three different ways:
• When a product purchase that is paid for is canceled, the money is refunded as cancellation
credit.
• Users can receive gift card credit as a gift.
• If a user has a poor service experience, credit may be provided.
Credit in the e-wallet expires after 6 months if it is gift card credit, but in 1 year if it is cancellation credit.

## Business process and assumptions

Let's define the business process of the e-wallet service, starting from the user point of view of the e-wallet service, the business process is as follows:

1) Users **receive** credit in their e-wallet account. (which will be recorded in the database as **positive** credit, with expiry in the future and category to identify it)

2) Users can **use** the available credit to pay for products, as long as the credit didn’t expire. (which will be recorded in the database as **negative** credit)

- Assuming the e-wallet service is available in *single currency* let's call it USD.
- We also assume that the e-wallet service allows purchases to use the available credit, and the consumptions of the credit are *from the soonest expiry credit to the longest*.

## Declare the grain and the thought process behind

Starting from the design of the e-wallet service, we would be interested to keep track of the **credits** that are available in the e-wallet service daily per category.

## Main component in e-wallet service (Identify the facts and dimensions)

1) Fact Table:
So we would choose to have a **credit table** as a fact table, which keeps track of any debit or credit balance that is available in the e-wallet service on daily basis per category.

2) Dimensions:

- **User table** Identify the credit owner. (which will hold user information)
- **Time dimension** Identify time for credit creation or expiration time. (which will help to identify the date in different granularity)
- **Credit Category**: identify the type of credit, which can be one of the following:
  - cancellation credit
  - gift card credit
  - service credit

## Data model

### ERD

Following is an ERD representation of the E-wallet system:

![ewallet_erd](ewallet_erd.png)

- 🔑 Represent Primary key (yellow key)
- 🗝 Represent Foreign key (red key)
- ◆ White diamond symbol represent a column that allows **Null** (due to the event not being completed yet)

### Relationships

- **one-to-one**: between credit and credit_category (assuming each credit can have a single category only)
- **one-to-one**: between credit and date dimension (assuming each credit has a single date dimension)
- **one-to-many**: between user and credit (assuming each user can have multiple credits, but any credit can be owned by only one user)

## Questions

### 1) What is the daily balance of credit in the e-wallet service?

```sql
--It's the sum of the credits and debits balance in the credit table, which haven't been expired.
SELECT SUM(amount)
  FROM credit
 WHERE expiration_date > CURRENT_DATE
```

### 2) How much credit will expire in the next month?

```sql
--It's the sum of the credits and debits balance in the credit table, which will expire in the next month.
SELECT SUM(amount)
  FROM credit
 WHERE date_trunc('month', expiration_date) > date_trunc('month', CURRENT_DATE) + INTERVAL '1 month'
   AND expiration_date < date_trunc('month', CURRENT_DATE) + INTERVAL '2 month'
```

### 3) What is the outcome (i.e. % used, % expired, % left) of credit given in a particular month?

```sql
with balance_credit as (
    SELECT SUM(IF(amount > 0, amount, 0)) AS deposit_credit
         , SUM(IF(amount < 0, abs(amount), 0)) AS withdrawal_credit
         , SUM(IF(NOT is_used AND NOT is_expired) ,amount, 0) AS left_credit
         , SUM(IF(amount > 0 AND NOT is_used AND is_expired, amount, 0)) AS expired_credit
      FROM credit
     WHERE created_time >= START_DATE
       AND created_time < START_DATE + INTERVAL '1 month'
)
SELECT (withdrawal_credit / deposit_credit) * 100 AS withdrawal_percent
     , (left_credit / deposit_credit) * 100 AS left_percent
     , (expired_credit / deposit_credit) * 100 AS expired_percent
  FROM balance_credit;
```
