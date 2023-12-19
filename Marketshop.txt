    create table users(
    -> id varchar(25) primary key not null,
    -> email varchar(225)  unique check (email = '^\\S+@\\S+\\.\\S+$') not null,
    -> user_role varchar(100) check (user_role in ('customer','shopkeeper','admin')) not null,
    -> name varchar(225) not null,
    -> password varchar(225) not null);

    create table pass(
    -> shopkeeper_userid varchar(25) primary key not null,
    -> expiry date not null,
    -> foreign key (shopkeeper_userid) references users(id));

    create table shops(
    -> shop_id varchar(25) primary key not null,
    -> shop_name varchar(100) not null,
    -> landmark varchar(225),
    -> monthly_rent int not null);

    create table shop_documents(
    -> license_id varchar(25) primary key not null,
    -> shopkeeper_userid varchar(25) not null,
    -> shop_id varchar(25) not null,
    -> initial_date date not null,
    -> expiry date not null,
    -> foreign key (shopkeeper_userid) references users(id),
    -> foreign key (shop_id) references shops(shop_id));

    create table transactions(
    -> payment_id varchar(25) primary key not null,
    -> license_id varchar(25) not null,
    -> payment_category varchar(20) check(payment_category in ('rent','electricity')) not null,
    -> amount int not null,
    -> penalty int not null default 0,
    -> last_date date not null,
    -> status varchar(30) check(status in ('not paid','pending approval','paid')) not null default 'not paid',
    -> paid_on date,
    -> unique key (license_id, payment_category, (YEAR(last_date)), (month(last_date))),
    -> foreign key (license_id) references shop_documents(license_id));
    

    create table remarks(
    -> license_id varchar(25) not null,
    -> customer_id varchar(25) not null,
    -> rating int not null check (rating >= -10 and rating <= 10),
    -> feedback varchar(665),
    -> primary key (license_id, customer_id),
    -> foreign key (license_id) references shop_documents(license_id),
    -> foreign key (customer_id) references users(id));

mysql> show tables;
+---------------------+
| Tables_in_dbms_mini |
+---------------------+
| pass                |
| remarks             |
| shop_documents      |
| transactions        |
| users               |
| shops               |
+---------------------+

mysql> show columns from users;
+-----------+--------------+------+-----+---------+-------+
| Field     | Type         | Null | Key | Default | Extra |
+-----------+--------------+------+-----+---------+-------+
| id        | varchar(25)  | NO   | PRI | NULL    |       |
| email     | varchar(225) | YES  |     | NULL    |       |
| user_role | varchar(100) | NO   |     | NULL    |       |
| name      | varchar(225) | NO   |     | NULL    |       |
| password  | varchar(225) | NO   |     | NULL    |       |
+-----------+--------------+------+-----+---------+-------+

mysql> show columns from shops;
+----------------+--------------+------+-----+---------+-------+
| Field          | Type         | Null | Key | Default | Extra |
+----------------+--------------+------+-----+---------+-------+
| shop_id        | varchar(25)  | NO   | PRI | NULL    |       |
| shop_name      | varchar(100) | NO   |     | NULL    |       |
| landmark       | varchar(225) | YES  |     | NULL    |       |
| monthly_rent   | int          | NO   |     | NULL    |       |
+----------------+--------------+------+-----+---------+-------+

mysql> show columns from shop_documents;
+-------------------+-------------+------+-----+---------+-------+
| Field             | Type        | Null | Key | Default | Extra |
+-------------------+-------------+------+-----+---------+-------+
| license_id        | varchar(25) | NO   | PRI | NULL    |       |
| shopkeeper_userid | varchar(25) | NO   | MUL | NULL    |       |
| shop_id           | varchar(25) | NO   | MUL | NULL    |       |
| initial_date      | date        | NO   |     | NULL    |       |
| expiry            | date        | NO   |     | NULL    |       |
+-------------------+-------------+------+-----+---------+-------+

mysql> show columns from pass;
+-------------------+-------------+------+-----+---------+-------+
| Field             | Type        | Null | Key | Default | Extra |
+-------------------+-------------+------+-----+---------+-------+
| shopkeeper_userid | varchar(25) | NO   | PRI | NULL    |       |
| expiry            | date        | NO   |     | NULL    |       |
+-------------------+-------------+------+-----+---------+-------+

mysql> show columns from transactions;
+------------------+-------------+------+-----+----------+-------+
| Field            | Type        | Null | Key | Default  | Extra |
+------------------+-------------+------+-----+----------+-------+
| payment_id       | varchar(25) | NO   | PRI | NULL     |       |
| license_id       | varchar(25) | NO   | MUL | NULL     |       |
| payment_category | varchar(20) | NO   |     | NULL     |       |
| amount           | int         | NO   |     | NULL     |       |
| penalty          | int         | NO   |     | 0        |       |
| last_date        | date        | NO   |     | NULL     |       |
| status           | varchar(30) | NO   |     | not paid |       |
| paid_on          | date        | YES  |     | NULL     |       |
+------------------+-------------+------+-----+----------+-------+

mysql> show columns from remarks;
+-------------+--------------+------+-----+---------+-------+
| Field       | Type         | Null | Key | Default | Extra |
+-------------+--------------+------+-----+---------+-------+
| license_id  | varchar(25)  | NO   | PRI | NULL    |       |
| customer_id | varchar(25)  | NO   | PRI | NULL    |       |
| rating      | int          | NO   |     | NULL    |       |
| feedback    | varchar(665) | YES  |     | NULL    |       |
+-------------+--------------+------+-----+---------+-------+

--a function to check if the given userRole is correct for the given userID
DELIMITER $$
CREATE FUNCTION check_user_role (userID INT, userRole VARCHAR(10))
RETURNS BOOLEAN DETERMINISTIC
BEGIN
DECLARE result BOOLEAN DEFAULT 0;
SELECT 1 INTO result FROM users WHERE id = userID AND user_role = userRole LIMIT 1;
RETURN result;
END $$
DELIMITER ;

--a trigger to see to it that gate passes should be issued to only shopkeepers
DELIMITER $$
CREATE TRIGGER check_gatepass
BEFORE INSERT ON pass FOR EACH ROW
BEGIN
IF NOT check_user_role(NEW.shopKeeper_userid, 'shopkeeper') THEN
SIGNAL SQLSTATE '45000'
SET MESSAGE_TEXT = 'NOT a ShopKeeper';
END IF;
END $$
DELIMITER ;

--a trigger to impose constraints on license
DELIMITER $$
CREATE TRIGGER check_license
BEFORE INSERT ON shop_documents FOR EACH ROW
BEGIN
IF NOT check_user_role(NEW.shopKeeper_userid, 'shopkeeper') THEN
SIGNAL SQLSTATE '45000'
SET MESSAGE_TEXT = 'NOT a ShopKeeper';
END IF;
END $$
DELIMITER ;

--a trigger to impose constraints on feedbacks
DELIMITER $$
CREATE TRIGGER check_feedback
BEFORE INSERT ON remarks FOR EACH ROW
BEGIN
IF NOT check_user_role(NEW.customer_id, 'customer') THEN
SIGNAL SQLSTATE '45000'
SET MESSAGE_TEXT = 'NOT a customer';
END IF;
END $$
DELIMITER ;

 LOAD DATA INFILE '"C:\Users\DELL\OneDrive\Desktop\pass.csv"' 
INTO TABLE pass 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;//

 LOAD DATA INFILE '"C:\Users\DELL\OneDrive\Desktop\shops.csv"' 
INTO TABLE shops 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;//

 LOAD DATA INFILE '"C:\Users\DELL\OneDrive\Desktop\users.csv"' 
INTO TABLE users 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;//

 LOAD DATA INFILE '"C:\Users\DELL\OneDrive\Desktop\shop_documents.csv"' 
INTO TABLE shop_documents 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;//

 LOAD DATA INFILE '"C:\Users\DELL\OneDrive\Desktop\transactions.csv.csv"' 
INTO TABLE transactions 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;//

 LOAD DATA INFILE '"C:\Users\DELL\OneDrive\Desktop\remarks.csv.csv"' 
INTO TABLE remarks 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;//

--Current shop details of different areas of the campus
SELECT name,shop_name, landmark, monthly_rent as rent, initial_date, expiry
FROM shop_documents
INNER JOIN users ON shop_documents.shopKeeper_userid = users.id
INNER JOIN shops ON shop_documents.shop_id = shops.shop_id
WHERE initial_date <= CURDATE() AND CURDATE() <= expiry;

+-----------+----------------+-------------------+-------+------------+------------+
| name      | shop_name      | landmark          | rent  | initial_date | expiry   |
+-----------+----------------+-------------------+-------+------------+------------+
| user2     | shop1          | incubation center | 10000 | 2020-01-01 | 2023-12-31 |
| user5     | shop2          | admin             |  8000 | 2021-01-02 | 2023-06-30 |
| user7     | shop4          | kalam             | 12000 | 2020-03-04 | 2023-01-31 |
| user12    | shop5          | asima             |  9500 | 2022-03-02 | 2023-12-01 |
| user13    | shop6          | gymkhana          |  8000 | 2021-08-02 | 2023-09-20 |
+-----------+----------------+-------------------+-------+------------+------------+
5 rows in set (0.02 sec)

--Details of shop keepers and their security pass validity
SELECT * FROM users
JOIN pass ON users.user_role = 'shopkeeper'
AND pass.shopKeeper_userid = users.id;

+----------+---------------------+------------+-----------+----------+-------------------+------------+
| id       | email               | user_role  | name      | password | shopkeeper_userid | expiry     |
+----------+---------------------+------------+-----------+----------+-------------------+------------+
| roll08   | roll08@yahoo.com    | shopkeeper | user5     | passcode | roll08            | 2023-02-01 |
| roll10   | roll10@yahoo.com    | shopkeeper | user7     | beats123 | roll10            | 2022-12-01 |
| roll07   | roll07@yahoo.com    | shopkeeper | user2     | secret   | roll07            | 2022-12-31 |
| roll12   | roll12@yahoo.com    | shopkeeper | user13    | abhi123  | roll12            | 2023-01-15 |
| roll09   | roll09@yahoo.com    | shopkeeper | user6     | hello    | roll09            | 2022-12-15 |
| roll11   | roll11@yahoo.com    | shopkeeper | user12    | abcd123  | roll11            | 2023-02-28 |
+----------+---------------------+------------+-----------+----------+-------------------+------------+
6 rows in set (0.01 sec)

--Reminders for expiring license agreement period
SELECT * FROM shop_documents
JOIN users ON shop_documents.shopKeeper_userid = users.id
AND initial_date <= CURDATE() AND CURDATE() <= expiry
WHERE DATEDIFF(expiry, CURDATE()) <= 300;

+------------+-------------------+---------+--------------+------------+--------+---------------------+------------+--------+----------+
| license_id | shopkeeper_userid | shop_id | initial_date | expiry     | id     | email               | user_role  | name   | password |
+------------+-------------------+---------+------------+------------+----------+---------------------+------------+--------+----------+
| license_2  | roll08            | store_2 | 2021-01-02   | 2023-06-30 | roll08 | roll08@yahoo.com    | shopkeeper | user5  | passcode |
| license_4  | roll10            | store_4 | 2020-03-04   | 2023-01-31 | roll10 | roll10@yahoo.com    | shopkeeper | user7  | beats123 |
+------------+-------------------+---------+------------+------------+----------+---------------------+------------+--------+----------+
2 rows in set (0.01 sec)


--Pending charges from each shop
SELECT shop_documents.shop_id, shops.shop_name,
SUM(amount) as 'Pending Charges' FROM transactions
JOIN shop_documents ON transactions.status != 'paid'
AND transactions.license_id = shop_documents.license_id
JOIN shops ON shop_documents.shop_id = shops.shop_id
GROUP BY shop_id;

+-----------+----------------+-----------------+
| shop_id   | shop_name      | Pending Charges |
+-----------+----------------+-----------------+
| store_1   | shop1          |            7000 |
| store_2   | shop2          |           22000 |
| store_3   | shop3          |            9000 |
| store_4   | shop4          |            6000 |
| store_5   | shop5          |            9500 |
| store_6   | shop6          |           16000 |
+-----------+----------------+-----------------+
6 rows in set (0.01 sec)

--Summary of performances of each shop
SELECT shop_documents.shop_id, shop_name, COUNT(*) as count,
IFNULL(SUM(rating), NULL) / COUNT(*) as rating
FROM shop_documents
LEFT JOIN remarks ON shop_documents.license_id = remarks.license_id
LEFT JOIN users ON shop_documents.shopKeeper_userid = users.id
LEFT JOIN shops ON shop_documents.shop_id = shops.shop_id 
GROUP BY shop_id;

+-----------+----------------+-------+--------+
| shop_id   | shop_name      | count | rating |
+-----------+----------------+-------+--------+
| store_1   | shop1          |     1 | 6.0000 |
| store_2   | shop2          |     1 | 7.0000 |
| store_3   | shop3          |     1 |   NULL |
| store_4   | shop4          |     2 | 5.0000 |
| store_5   | shop5          |     2 | 6.0000 |
| store_6   | shop6          |     1 |   NULL |
+-----------+----------------+-------+--------+
6 rows in set (0.02 sec)

SELECT * FROM remarks;
+----------------+-------------+--------+-------------------------------------------------+
| license_id     | customer_id | rating | feedback                                        |
+----------------+-------------+--------+-------------------------------------------------+
| license_1      | roll01      |      6 | Apple juice is very good but less quantity      |
| license_1      | roll06      |      5 | Juices taste bitter sometimes.                  |
| license_2      | roll04      |      7 | quantity is less                                |
| license_2      | roll01      |      7 | very tasty maggie and patties                   |
| license_3      | roll03      |     10 | awesome                                         |
| license_4      | roll04      |      4 | shops is not open on holidays                   |
| license_4      | roll02      |      4 | all necessary items are not available           |
| license_4      | roll03      |      2 | Quality and hygiene is not maintained properly. |
| license_4      | roll05      |      6 | less staff                                      |
| license_5      | roll02      |      8 | I get all the max required items                |
| license_5      | roll03      |      3 | rice is not good                                |
| license_5      | roll06      |      9 | great                                           |
| license_6      | roll04      |      8 | fruits are good                                 |
| license_6      | roll05      |      4 | Only limited quantity of fruits are available.  |
+----------------+-------------+--------+-------------------------------------------------+
14 rows in set (0.00 sec)


-- to find the reason why a shop got less rating
SELECT license_id,feedback FROM remarks s1 
WHERE s1.rating IN 
    (SELECT min(rating) FROM remarks s2 
        WHERE s1.license_id=s2.license_id);
        
+----------------+-------------------------------------------------+
| license_id     | feedback                                        |
+----------------+-------------------------------------------------+
| license_1      | Juices taste bitter sometimes.                  |
| license_2      | quantity is less                                |
| license_2      | very tasty maggie and patties                   |
| license_3      | awesome                                         |
| license_4      | Quality and hygiene is not maintained properly. |
| license_5      | rice is not good                                |
| license_6      | Only limited quantity of fruits are available.  |
+----------------+-------------------------------------------------+
7 rows in set (0.01 sec)
    
-- to find the total rent to be paid in their tenure
SELECT shops.shop_id,(TIMESTAMPDIFF(MONTH,initial_date,expiry))*monthly_rent AS total_rent 
FROM shops INNER JOIN shop_documents 
    ON shops.shop_id=shop_documents.shop_id;
+-----------+------------+
| shop_id   | total_rent |
+-----------+------------+
| store_1   |     470000 |
| store_2   |     232000 |
| store_3   |     234000 |
| store_4   |     408000 |
| store_5   |     190000 |
| store_6   |     200000 |
+-----------+------------+
6 rows in set (0.00 sec)

