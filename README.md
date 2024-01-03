# More Than You Ever Wanted to Know: Episode 1 - Relational Databases and Object Relational Mappers

# DATABASE SETUP AND MANAGEMENT USING AN ORM (SQLAlchemy)

# THE DATABASE:

## What a database is:
A binary file or collection of binary files stored in the file system. What makes these files unique is the way that we interact with them, including how we read and write to them. This is where the true defining feature of a database comes in. The DBMS!!

## What a DBMS is:
DBMS is short for DataBase Management System. In short, it is what makes a file or group of files into a database. These management systems all organize data in their own unique ways to optimize read and writes to the database. In our case, we are using Sqlite3 locally and Postgres in production. Also note that both of these DBMSs or RDBMSs (as they are used with Relational Databases), are both considered Transactional! By transactional, we mean that we interact with the database using Transactions! Transactions are a relatively simple concept with BIG implications!! Essentially a transaction defines a "Unit of Work" to be performed on the database, meaning that we can group multiple SQL statements into what is considered an aggregate command. If during the processing of these requests ANY error is thrown, the database will not make ANY changes to the database and will ROLLBACK. This simple feature prevents incomplete writes to the database and also ensures that any statements that were meant to be processed together (possibly reliant upon one another) either succeed as a whole or fail as a whole, thereby ensuring data integrity!! We will see some examples of Transactions below!!

# THE ORM:

## WHAT an ORM is:
The term ORM is short for Object Relational Mapper. An ORM performs 2 main tasks for us. 1) It provides us with a convenient Object Oriented interface, with which we can interact with our database. And 2) It writes SQL Statements for us! One of the really helpful aspects of the first feature is that it parses responses from the database into class instances that represent rows in our db! Let's see an example:

### An Example Mapping:
When we fetch the Demo user from our database, we receive the packet below as the response.<br>

<img width="955" alt="db_select_response" src="https://github.com/crespohector/System-Design-Lecture-In-Mod-7/assets/107947798/37c5345f-3826-4e0a-94c4-78584b5acad3">

This packet has been formatted by the tool Wireshark and can be parsed as follows: On the far left, we simply have numbers of bytes in hexadecimal. So 0010 is actually the number 16. Each "line" consists of 16 bytes each in the center column. These are the raw bytes received by our client in hexidecimal notation. In the rightmost column we have a text representation of the packet. Note that many of the bytes in the packet are outside of the ASCII range and do not have glyph representations. These characters have been indicated with the strings of periods in the text '.....'. If we focus on the data portion of the packet, we see that 2 of the 4 PostgreSQL Headers represent the table row we queried for. The first, describes the table columns by name and the second provides the values for those columns in the row. Using this data in tabular format, SQLAlchemy can map the values to the respective keys and instantiate a class instance using this data! If we "unwind" the text representations of these 2 Headers we can loosely correlate the 2 as being 1 Header for "keys" and 1 being a Header with the "values" for those keys (Note: I believe the 'T' and the 'D' that begin the 2 headers are special control characters used by the PostgreSQL protocol but have not actually confirmed that...): 

<img width="1146" alt="row_description_and_row_data" src="https://github.com/crespohector/System-Design-Lecture-In-Mod-7/assets/107947798/ec64d017-8761-4641-9715-1ba5d95b4058">

Now that SQLAlchemy has the keys and the values that map to them, it can instantiate an object of class User, using these values, thereby providing us with a convenient Python object for us to interact with programmatically and ultimately return to the client side of our app. A class instance which we will turn into a dictionary as shown below:

```
{
  'id': 1,
  'username': 'Demo',
  'email': 'demo@aa.io'
}
```

Also by using the builtin methods, we are having our ORM write SQL queries for us. Let's correlate some common SQLAlchemy lines into their respective SQL statements:

### Fetching a single row of data from a database table:
```
# /api/users/id
user = User.query.get(id)
```
```
BEGIN (1st command sent to db)

SELECT users.id AS users_id, users.username AS users_username, users.email AS users_email, users.hashed_password AS users_hashed_password
FROM users
WHERE users.id = %(pk_1)s (2nd command sent to db)

ROLLBACK (final command sent to db)
```

### Fetching all rows from a database table:
```
# /api/users/
users = User.query.all()
```
```
BEGIN (1st command sent to db)

SELECT users.id AS users_id, users.username AS users_username, users.email AS users_email, users.hashed_password AS users_hashed_password 
FROM users (2nd command sent to db)

ROLLBACK (final command sent to db)
```

### Fetching posts through a relationship:
```
# /api/users/id
posts = user.posts
# By simply keying into our relationship from our user instance, SQLAlchemy generates the SQL Statement below and retrieves the posts from the db!
```
```
SELECT posts.id AS posts_id, posts.user_id AS posts_user_id, posts.content AS posts_content 
FROM posts
WHERE %(param_1)s = posts.user_id
```
### Posting a new user to the db:
```
# Creating a new user row in db and returning that newly created row to the caller.
# In the SQL below, note the RETURNING keyword which is returning the new user id after creation.
# That id is then used to query the database for the newly created user to send it back to us
# so that we can assign it to the variable new_user.
new_user = User(
  username=form.data['username'],
  email=form.data['email'],
  password=form.data['password']
)
db.session.add(user)
db.session.commit()
```
```
// Transaction 1
BEGIN (implicit)
INSERT INTO users (username, email, hashed_password) VALUES (%(username)s, %(email)s, %(hashed_password)s) RETURNING users.id
COMMIT

// Transaction 2
BEGIN (implicit)
SELECT users.id AS users_id, users.username AS users_username, users.email AS users_email, users.hashed_password AS users_hashed_password
FROM users
WHERE users.id = %(pk_1)s
SELECT posts.id AS posts_id, posts.user_id AS posts_user_id, posts.content AS posts_content
FROM posts
WHERE %(param_1)s = posts.user_id
ROLLBACK
```
### Updating a record in the db:
```
# /api/users/update/id
user_to_update = User.query.filter(User.id == id).first()
user_to_update.email = body['email']
user_to_update.username = body['username']
db.session.commit()
```
```
BEGIN (implicit)
SELECT users.id AS users_id, users.username AS users_username, users.email AS users_email, users.hashed_password AS users_hashed_password
FROM users
WHERE users.id = %(id_1)s
 LIMIT %(param_1)s
UPDATE users SET username=%(username)s, email=%(email)s WHERE users.id = %(users_id)s
INFO sqlalchemy.engine.Engine COMMIT
```
### Deleting a record from the db:
```
# /api/users/delete/id
user_to_delete = User.query.get(id)

if user_to_delete != None:
  db.session.delete(user_to_delete)
  db.session.commit()
```
```
BEGIN (implicit)
SELECT users.id AS users_id, users.username AS users_username, users.email AS users_email, users.hashed_password AS users_hashed_password 
FROM users 
WHERE users.id = %(pk_1)s
DELETE FROM users WHERE users.id = %(id)s
COMMIT
```

# THE NETWORK:
Note that, while we are accustomed to communicating with our backend server from the client using HTTP Request / Responses,<br>
the way that our backend communicates with the database is a little different. Notably, it does NOT use the HTTP Protocol and instead<br>
each DBMS uses it's own proprietary protocol on top of the TCP / IP Protocol, which HTTP is also built on top of. In production we will see packets marked with PostgreSQL Headers! I also want to make a general note about how the TCP Protocol works. As we'll see in the screenshot below, EVERY packet that is recieved on either end of the communication will be ACKNOWLEDGED by sending an ACK TCP packet in response. This tells the sender of the packet that it was recieved successfully. As TCP has methods in place to ensure data integrity, this extra response allows us to check for the condition where a packet was lost or even malformed at some point in it's journey and allows the sender to re-send the packet when necessary. Also note that the sender will set a timeout when sending data. If an ACK packet isn't received by the expiry of the timer, it will automatically resend the packet! TCP offers assurance of data transmission!!

### TCP Acknowledgment Packets:
<img width="1958" alt="tcp_ack" src="https://github.com/crespohector/System-Design-Lecture-In-Mod-7/assets/107947798/cd4c4cca-fade-4cde-98d5-5121547fcfd5">

### PostgreSQL Transaction:
<img width="1953" alt="db_transaction" src="https://github.com/crespohector/System-Design-Lecture-In-Mod-7/assets/107947798/aa9b1052-63b1-4ad8-bdf4-0fd2e48df5d6">



# SQL:

As mentioned above, the SQL Language comprises a number of sub-classes of commands (DDL, DCL, DML, TCL). Or rather, all SQL commands fall into at least 1 of 4 possible categories. Below we shall see an example of the 4 categories and common SQL commands that belong to those categories. One note I'd like to make here is to mention the category of commands called DDL or Data Definition Language. These commands are used when creating, emptying, destroying, altering, and commenting database TABLES!! For this reason, we often see the line: "Will assume transactional DDL", when we are migrating our database schema to the database (flask db upgrade). As we are CREATING tables, we are using DDL commands and SQLAlchemy is kind enough to point this out to us!! Below is a list generated by chatGPT to outline these command categories:

<img width="708" alt="SQL_Sub-categories" src="https://github.com/crespohector/System-Design-Lecture-In-Mod-7/assets/107947798/bd81de1c-05aa-416d-890a-23ef7978b87e">

# GOTCHAS:
1. Note that if you are watching your Flask terminal and seeing a lot of unexpected SQL, it may be coming from your validation Form, not the visible endpoint code...
