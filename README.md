# Analytical-Insight-of-clients
Providing analytical insight for your clients for 12, 18 and 24 months on portfolio
/*
This code is a collaborative project of all team members of Team 7. 
Each team member worked with the code for their assigned customers and also gave input to improve the code.
The designer(s) of each query is specified in the comments. 
If as second team member had significant assisting role in the query design, they are mentioned as 'in collaboration with X'.
Some querries were also designed during team meetings. These are marked as 'designed collaboratively by all team members'. 

The code handed in shows examples for how each query was executed. 
As mentioned in the querries those are only examples for a multitude of customers and accounts that were analyzed.
Therefore, we as a team believe, that these examples are representative for all team members.
*/

USE invest;

-- Filtering for only adjusted prices (as shown in class)
SELECT *
FROM pricing_daily_new
WHERE price_type = 'Adjusted'
LIMIT 500
;

-- Calculate the lag of 1 (as shown in class)
SELECT *, 
	LAG(value, 1) OVER(
						PARTITION BY ticker
                        ORDER BY date
						) AS lagged_price
FROM pricing_daily_new
WHERE price_type = 'Adjusted'
LIMIT 100
;

-- Top 10 biggest accounts
-- Designed collaboratively by all team members
SELECT account_id, ROUND(SUM(value * quantity),2) AS account_value -- calculating the value of each account by multiplying the quantity of one holding held on the account times the value of the holding. Then all holding*quantity values are summed up
FROM holdings_current -- using holdings:current as it contains the most recent value of each holding and the quantity held of a holding for each account
GROUP BY account_id
ORDER BY account_value DESC
LIMIT 10;

-- Top 10 biggest customers 
-- Designed collaboratively by all team members
SELECT full_name AS `Name`,
		CONCAT('$', (ROUND(SUM(value * quantity),2))) AS accounts_value, -- same calculation as in previous query but with different group by statement
		COUNT(DISTINCT ad.account_id) AS number_of_accounts, -- counting the number of accounts a person has as additional information
        customer_location AS location -- selecting customer location as additional information
FROM holdings_current AS hc
JOIN account_dim AS ad -- joining account_dim as bridge betweeen customer_details and holdings_current
ON hc.account_id = ad.account_id
JOIN customer_details AS cd -- joining customer_details to get customer details
ON ad.client_id = cd.customer_id
GROUP BY full_name -- grouping by full_name as unique identifier (works here because there is not that many customers in the dataset
ORDER BY SUM(value * quantity) DESC -- ordering to see the biggest customers only
LIMIT 10; -- limiting results to top 10 custoemers

-- Return rate per stock over a 12 month period 
-- Designed collaboratively by all team members
SELECT a.date, a.ticker, (a.value - a.lagged_price)/a.lagged_price AS returns -- calculating return rate by using (current value - 12 month ago value)/12 month ago value
FROM(
SELECT *, 
	LAG(value, 250) OVER(					-- A lag of 250 days roughly represent a year on the stock market (not open on weekends and public holidays)
						PARTITION BY ticker
                        ORDER BY date
						) AS lagged_price
FROM pricing_daily_new
WHERE price_type = 'Adjusted') as a 		-- Only using adjusted price as recommended by Prof Thomas due to better accuracy in the results
WHERE (a.value - a.lagged_price)/a.lagged_price IS NOT NULL -- Filtering out NULL values to make results easier to analyze
AND a.date LIKE '%-09-09' -- Only cosidering one date in each year to improve query efficiency
;

-- Return rate per stock over a 18 month period 
-- Designed collaboratively by all team members
SELECT a.date, a.ticker, (a.value - a.lagged_price)/a.lagged_price AS returns
FROM(
SELECT *, 
	LAG(value, 378) OVER(					-- Same query as for 12 month but with lag now adapted for roughly 18 month period
						PARTITION BY ticker
                        ORDER BY date
						) AS lagged_price
FROM pricing_daily_new
WHERE price_type = 'Adjusted') as a
WHERE (a.value - a.lagged_price)/a.lagged_price IS NOT NULL
AND a.date LIKE '%-09-09'
;

-- Return rate per stock over a 24 month period
-- Designed collaboratively by all team members
SELECT a.date, a.ticker, (a.value - a.lagged_price)/a.lagged_price AS returns
FROM(
SELECT *, 
	LAG(value, 500) OVER(					-- Same query as for 12 month but with lag now adapted for roughly 24 month period
						PARTITION BY ticker
                        ORDER BY date
						) AS lagged_price
FROM pricing_daily_new
WHERE price_type = 'Adjusted') as a
WHERE (a.value - a.lagged_price)/a.lagged_price IS NOT NULL
AND a.date LIKE '%-09-09'
;

-- Diving into the analysis

-- Current Holdings all accounts for 'Moritz Petasch' (one example out of top 10 customers that were analyzed)
-- Designed collaboratively by all team members
SELECT *
FROM holdings_current AS hc -- getting information for all accounts of a specific person in holdings_current
JOIN security_masterlist AS sm -- getting more information on the holdings from the security_masterlist
ON hc.ticker = sm.ticker
WHERE hc.account_id IN (SELECT ad.account_id 
						FROM account_dim AS ad
                        JOIN customer_details AS cd
                        ON ad.client_id = cd.customer_id
                        WHERE cd.full_name = 'Moritz Petasch') -- filtering for accounts of only one customer (can be exchanged for any name in database)
ORDER BY hc.account_id
;

-- Calculating the value of each account when opened and now and calculating the return rate for account 82801 (as one example out of appr 35 analyzed like this)
-- Designed by Tim Hahne in cooperation with Zaynab Zennour and adapted by each Team member to analyze the accounts of their customers (each team member was assigned 2-3 customers with 3-4 accounts each)
SELECT (SELECT SUM(pd.value * hc.quantity) -- As previously, the quantity of a holding in an account is multiplied by the value of the holding and then summed up for all holdings in an account
			FROM holdings_current AS hc
            JOIN pricing_daily_new AS pd 
            ON hc.ticker = pd.ticker
			WHERE hc.account_id = 82801 -- filtering for a specific account
            AND pd.price_type = 'Adjusted' -- only using adjusted prices
            AND pd.ticker IN (SELECT ticker
								FROM holdings_current
                                WHERE account_id = 82801) -- for all holdings that are held by the specified account
			AND pd.date = (SELECT (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end) 
            -- Filtering for holding values of the day when the specified account was opened
            -- Some accounts are opened on the weekend. this case statement makes sure that there is an entry in the pricing_daily_new for the holdings by looking up the value of the holding on the following Monday
								FROM account_dim
                                WHERE account_id = 82801)) AS opening_value_account, -- opening value of the account 
                                
			(SELECT SUM(hc.value * hc.quantity)
			FROM holdings_current AS hc
            WHERE account_id = 82801) AS current_value_account, -- current value of the account by using the holding values from the holdings_current table
            
            (((SELECT SUM(hc.value * hc.quantity)
			FROM holdings_current AS hc
            WHERE hc.account_id = 82801) -- filtering for a specific account
            -
            (SELECT SUM(pd.value * hc.quantity)
			FROM holdings_current AS hc
            JOIN pricing_daily_new AS pd
            ON hc.ticker = pd.ticker
			WHERE hc.account_id = 82801 -- filtering for a specific account
            AND pd.price_type = 'Adjusted' -- only using adjusted prices
            AND pd.ticker IN (SELECT ticker
								FROM holdings_current
                                WHERE account_id = 82801)
			AND pd.date = (SELECT (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
								FROM account_dim
                                WHERE account_id = 82801)))
			/
            (SELECT SUM(pd.value * hc.quantity)
			FROM holdings_current AS hc
            JOIN pricing_daily_new AS pd
            ON hc.ticker = pd.ticker
			WHERE hc.account_id = 82801 -- filtering for a specific account
            AND pd.price_type = 'Adjusted' -- only using adjusted prices
            AND pd.ticker IN (SELECT ticker
								FROM holdings_current
                                WHERE account_id = 82801)
			AND pd.date = (SELECT (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
								FROM account_dim
                                WHERE account_id = 82801)))
			/(SELECT (DATEDIFF(a.currentdate, a.opendate))/365 
            -- to calculate the yearly growth rate since the account was opened the current date is substracted from the account opening date. Since result is in days, it is divided by 365 to calculate yearly rate
				FROM (SELECT pd.date AS opendate, hc.date AS currentdate
						FROM pricing_daily_new AS pd
						JOIN holdings_current AS hc
						ON pd.ticker = hc.ticker
						WHERE pd.date = (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
								FROM account_dim
                                WHERE account_id = 82801)
				LIMIT 1) AS a)	AS growth_rate -- using the previous querries the growth rate of the specified account is calculated by (current value - opening value)/opening value
FROM holdings_current AS hc
JOIN pricing_daily_new AS pd
ON hc.ticker = pd.ticker
LIMIT 1
;

-- After identifying underperforming accounts, now holding performances in these underperforming accounts are compared
-- Analyzing average monthly return rates of holdings in account 59402 (as one example out of 9 accounts analyzed like this)
-- Designed by Tim Hahne in cooperation with Zaynab Zennour and adapted by each Team member to analyze the accounts of their customers (each team member had 2-3 underperforming accounts)
SELECT a.ticker, CONCAT('$',FORMAT(a.opening_value, 2)) AS opening_value, 
		CONCAT('$',FORMAT(b.current_value, 2)) AS current_value, 
		CONCAT(ROUND((((b.current_value - a.opening_value)/a.opening_value) * 100), 2), '%') AS return_rate, -- return rate per holding is calculated here for the period the account has existed 
		ROUND((b.ticker_value/b.total_value), 4) AS weight_holding -- weight of the holding in the account
FROM (SELECT pd.value AS opening_value, pd.ticker as ticker
			FROM pricing_daily_new AS pd
            JOIN security_masterlist as sm
            ON pd.ticker = sm.ticker
            WHERE pd.price_type = 'Adjusted'
            AND pd.date = (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
			-- Filtering for holding values of the day when the specified account was opened
            -- Some accounts are opened on the weekend. this case statement makes sure that there is an entry in the pricing_daily_new for the holdings by looking up the value of the holding on the following Monday
								FROM account_dim
                                WHERE account_id = 59402)
			AND sm.ticker IN (SELECT ticker
					FROM holdings_current
                    WHERE account_id = 59402)) AS a -- as previously the opening value for each ticker is calculated by filtering for the opening date for the account and for tickers in the account
JOIN (SELECT value AS current_value, ticker, quantity, 
		(SELECT SUM(value * quantity)
			FROM holdings_current
            WHERE account_id = 59402) AS total_value, 
		(value * quantity) AS ticker_value
			FROM holdings_current
            WHERE account_id = 59402) AS b -- current value is again retrieved from holdings_current and filtered for holdings in the speicified account
ON a.ticker = b.ticker
;     

-- Comparing holdings that are in the same categories to choose best performing one
-- Analyzing average monthly return rates of holdings in account 59402 (as one example out of 9 accounts analyzed like this)
-- Designed by Tim Hahne in cooperation with Zaynab Zennour and adapted by each Team member to analyze the accounts of their customers (each team member had 2-3 underperforming accounts with 1-2 underperforming holdings)
SELECT a.ticker, CONCAT('$',FORMAT(a.opening_value, 2)) AS opening_value, -- opening value of holding is calculated in the same way as before
		CONCAT('$',FORMAT(b.current_value, 2)) AS current_value, -- current value is calculated in the same way as before 
		CONCAT(ROUND((((b.current_value - a.opening_value)/a.opening_value) * 100), 2), '%') AS return_rate, -- return rate per holding calculated as before
		CONCAT(FORMAT(c.mu * 100, 2), '%') AS mu, -- mu of each holding
        CONCAT(FORMAT(c.sigma * 100, 2), '%') AS sigma, -- Sigma of each holding 
        CONCAT(FORMAT(c.risk_adj_returns * 100, 2), '%') AS risk_adj_returns -- risk adjusted return for each holding
FROM (SELECT pd.value AS opening_value, pd.ticker as ticker
			FROM pricing_daily_new AS pd
            JOIN security_masterlist as sm
            ON pd.ticker = sm.ticker
            WHERE pd.price_type = 'Adjusted'
            AND pd.date = (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
			-- Filtering for holding values of the day when the specified account was opened
            -- Some accounts are opened on the weekend. this case statement makes sure that there is an entry in the pricing_daily_new for the holdings by looking up the value of the holding on the following Monday
								FROM account_dim
                                WHERE account_id = 594)
			AND sm.sec_type = (SELECT sec_type -- Filtering for holdings with same sec_type as the one that should be replaced
							FROM security_masterlist
                            WHERE ticker = 'UCO')
			AND sm.major_asset_class = (SELECT major_asset_class -- Filtering for holdings with same major_asset_class as the one that should be replaced
							FROM security_masterlist
                            WHERE ticker = 'UCO')
			AND sm.minor_asset_class = (SELECT minor_asset_class -- Filtering for holdings with same minor_asset_class as the one that should be replaced
							FROM security_masterlist
                            WHERE ticker = 'UCO')) AS a
JOIN (SELECT pd.value AS current_value, pd.ticker as ticker
			FROM pricing_daily_new AS pd
            JOIN security_masterlist as sm
            ON pd.ticker = sm.ticker
            WHERE pd.price_type = 'Adjusted'
            AND pd.date = '2022-09-09'
			AND sm.sec_type = (SELECT sec_type -- Filtering for holdings with same sec_type as the one that should be replaced
							FROM security_masterlist
                            WHERE ticker = 'UCO')
			AND sm.major_asset_class = (SELECT major_asset_class -- Filtering for holdings with same major_asset_class as the one that should be replaced
							FROM security_masterlist
                            WHERE ticker = 'UCO')
			AND sm.minor_asset_class = (SELECT minor_asset_class -- Filtering for holdings with same minor_asset_class as the one that should be replaced
							FROM security_masterlist
                            WHERE ticker = 'UCO')) AS b
ON a.ticker = b.ticker
JOIN (SELECT AVG((a.value - a.lagged_price)/a.lagged_price) AS mu, -- mu calculated as explained by Prof Thomas
		STD((a.value - a.lagged_price)/a.lagged_price) AS sigma, -- sigma calculated as explained by Prof Thomas
        AVG((a.value - a.lagged_price)/a.lagged_price)/STD((a.value - a.lagged_price)/a.lagged_price) AS risk_adj_returns, -- calculated as explained by Prof Thomas
		a.ticker as ticker
		FROM(SELECT pd.value AS value, pd.ticker AS ticker, LAG(value, 250) OVER(
						PARTITION BY pd.ticker ORDER BY pd.date) AS lagged_price
                        FROM pricing_daily_new AS pd
                        JOIN security_masterlist as sm
            ON pd.ticker = sm.ticker
            WHERE pd.price_type = 'Adjusted'
            AND pd.date >= (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
								FROM account_dim
                                WHERE account_id = 594)
			AND sm.sec_type = (SELECT sec_type
							FROM security_masterlist
                            WHERE ticker = 'UCO')
			AND sm.major_asset_class = (SELECT major_asset_class
							FROM security_masterlist
                            WHERE ticker = 'UCO')
			AND sm.minor_asset_class = (SELECT minor_asset_class
							FROM security_masterlist
                            WHERE ticker = 'UCO')) AS a
			WHERE ((a.value - a.lagged_price)/a.lagged_price) IS NOT NULL
			GROUP BY ticker) AS c
ON a.ticker = c.ticker
GROUP BY a.ticker
;

-- Querries for graphical representation in final presentation to show recommended holding compared to original holding
-- Designed by Tim Hahne
SELECT value,ticker, `date`
FROM pricing_daily_new AS pd
WHERE pd.price_type = 'Adjusted'
AND pd.date >= (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
				FROM account_dim
				WHERE account_id = 59402)
AND ticker = 'KOLD'
;

SELECT value,ticker, `date`
FROM pricing_daily_new AS pd
WHERE pd.price_type = 'Adjusted'
AND pd.date >= (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
				FROM account_dim
				WHERE account_id = 59402)
AND ticker = 'AAAU'
;


-- Additional insights not used in the main presentation but as additional backup data in the appendix
-- CREATING a view with returns account 59402 since opening (as one example of accounts analyzed)
-- Designed by Zaynab Zennour
create view test10 as 
SELECT a.sec_type, a.ticker, a.opening_value, b.current_value, ((b.current_value - a.opening_value)/a.opening_value) * 100 AS return_rate
FROM (SELECT pd.value AS opening_value, pd.ticker as ticker, sec_type
			FROM pricing_daily_new AS pd
            JOIN security_masterlist as sm
            ON pd.ticker = sm.ticker
            WHERE pd.price_type = 'Adjusted'
            AND pd.date = (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
								FROM account_dim
                                WHERE account_id = 59402)
			AND sm.ticker IN (SELECT ticker
					FROM holdings_current
                    WHERE account_id = 59402)) AS a
JOIN (SELECT value AS current_value, ticker
			FROM holdings_current
            WHERE account_id = 59402) AS b
ON a.ticker = b.ticker
GROUP BY a.sec_type, a.ticker
having return_rate is not null
;
-- Calculate the correlation between the etf and common share assets types for account_id =59402 to check adversity portfolio for account_id =54902 (as one example of accounts analyzed)
-- Designed by Zaynab Zennour
Select (Avg(a.x * a.y)-(Avg(a.x) * Avg(a.y))) / (STDDEV(a.x) * STDDEV(a.y)) AS correlation_type
from 
(select 
(select return_rate  from test10 where sec_type ='etf') as x
,(select return_rate  from test10 where sec_type like 'com%') as y
from test10)
 as a ;

-- calculating the correlation between security types among account_id = 54902
-- Designed by Zaynab Zennour
Select (Avg(a.x * a.y)-(Avg(a.x) * Avg(a.y))) / (STDDEV(a.x) * STDDEV(a.y)) AS correlation_type
from 
(select 
(select return_rate  from test10 where sec_type ='etf') as x
,(select return_rate  from test10 where sec_type like 'common_share') as y
from test10)
 as a ;
 
 -- return rate per security type for correlation analysis
 -- Designed by Zaynab Zennour
SELECT a.sec_type, a.ticker, a.opening_value, b.current_value, ((b.current_value - a.opening_value)/a.opening_value) * 100 AS return_rate
FROM (SELECT pd.value AS opening_value, pd.ticker as ticker, sec_type
			FROM pricing_daily_new AS pd
            JOIN security_masterlist as sm
            ON pd.ticker = sm.ticker
            WHERE pd.price_type = 'Adjusted'
            AND pd.date = (select (case when WEEKDAY(acct_open_date) > 4 then DATE_ADD(acct_open_date, INTERVAL 2 DAY) else acct_open_date end)
								FROM account_dim
                                WHERE account_id = 59402)
			AND sm.ticker IN (SELECT ticker
					FROM holdings_current
                    WHERE account_id = 59402)) AS a
JOIN (SELECT value AS current_value, ticker
			FROM holdings_current
            WHERE account_id = 59402) AS b
ON a.ticker = b.ticker
GROUP BY a.sec_type, a.ticker
having return_rate is not null
;

-- Average Return rate per stock over a 12 month period
-- Designed by Zaynab Zennour
SELECT a.ticker, AVG(a.value - a.lagged_price)/a.lagged_price AS avg_returns
FROM(
SELECT *, 
	LAG(value, 250) OVER(
						PARTITION BY ticker
                        ORDER BY date
						) AS lagged_price
FROM pricing_daily_new
WHERE price_type = 'Adjusted') as a
WHERE (a.value - a.lagged_price)/a.lagged_price IS NOT NULL
ORDER BY AVG(a.value - a.lagged_price)/a.lagged_price
;

-- avg price per month
-- Designed by Zaynab Zennour
SELECT a.ticker, STDDEV(a.lagged_price) AS avg_price
FROM(
SELECT *, 
	LAG(value, 250) OVER(
						PARTITION BY ticker
                        ORDER BY date
						) AS lagged_price
FROM pricing_daily_new
WHERE price_type = 'Adjusted') as a
WHERE a.lagged_price IS NOT NULL
AND a.date LIKE '%-09-09'
LIMIT 1
;

-- Calculate the risk of stocks over the years : std and variance
-- Designed by Zaynab Zennour
select  a.ticker, round(stddev(a.quantity*a.value),0) as stock_risk 
from 
(select h.quantity as quantity, h.ticker as ticker, year(H.date), H.value as value
from 
pricing_daily_new p
join holdings_current as h on h.ticker = p.ticker) as a 
group by a.ticker 
order by stock_risk asc; 

-- variance 
-- Designed by Zaynab Zennour
select  a.ticker, round(variance(a.quantity*a.current_value),0) as stock_risk 
from 
(select h.quantity as quantity,H.ticker as ticker , year(H.date), h.value as current_value
from 
pricing_daily_new p
join holdings_current as h on h.ticker = s.ticker) as a 
group by a.ticker 
order by stock_risk asc; 

-- Calculate the risk of stock classes over the years : std and variance 
-- Designed by Zaynab Zennour
select  a.major_asset_class, a.minor_asset_class, round(stddev(a.quantity*a.value),0) as class_risk 
from 
(select h.quantity as quantity, h.ticker as ticker, year(p.date), p.value, major_asset_class, minor_asset_class
from 
pricing_daily_new as p
join security_masterlist as s on p.ticker = s.ticker
join holdings_current as h on h.ticker = s.ticker) 
as a 
group by a.ticker 
order by class_risk  asc; 

select  a.major_asset_class, a.minor_asset_class, round(variance(a.quantity*a.value),0) as class_risk 
from 
(select h.quantity as quantity, h.ticker as ticker, year(p.date), p.value, major_asset_class, minor_asset_class
from 
pricing_daily_new as p
join security_masterlist as s on p.ticker = s.ticker 
join holdings_current as h on h.ticker = s.ticker) 
as a 
group by a.ticker 
order by class_risk  desc; 

Select (Avg(a.x * a.y)-(Avg(a.x) * Avg(a.y))) / (STDDEV(a.x) * STDDEV(a.y)) AS correlation_type
from 
(select case when p.ticker = 'KOLD' then h.quantity*p.value end as x, case when p.ticker = 'KOLD' then h.quantity*p.value end as y 
from 
pricing_daily_new as p
join security_masterlist as s on p.ticker = s.ticker 
join holdings_current as h on h.ticker = s.ticker) 
as a 
