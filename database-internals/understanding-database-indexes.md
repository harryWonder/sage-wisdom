# Database Indexes

# **Database Indexes How MySQL and PostgreSQL Implement Them Internally**

A database index is a secondary data structure the DB uses to quickly locate rows. Without an index, the database must scan the table’s heap pages(This is where all of those data lives) directly, which is slow and IO-heavy.

Although different databases expose similar APIs, their internal index structures work very differently.

Let’s compare **MySQL InnoDB** and **PostgreSQL** using a simple `Users` table:

- `id`
- `first_name`
- `last_name`
- `email`

Assume we create an index on the `email` column.

---

## **How MySQL InnoDB Stores Indexes**

MySQL InnoDB uses:

- A **clustered index** for the table data (ordered by the primary key)
- **Secondary indexes** for other columns

When you create an index on `email`, InnoDB builds a **B+-tree** that stores:

- the email value
- the **primary key value**

This means secondary indexes do not contain the full row.

So when you run:

```sql
SELECT * FROM users WHERE email = ?
```

InnoDB performs a **double lookup**:

1. Search the secondary index and  retrieve the primary key for that index.
2. Use the retrieved primary key to look up the full row in the clustered index.

This behaviour is what we term a double read.

---

## **How PostgreSQL Stores Indexes**

PostgreSQL tables are stored as a **heap**, but they are not clustered by any index in the case of Mysql.

When you create an index on `email`, PostgreSQL builds a B-tree that stores:

- the email value
- a **CTID** which is a pointer to the row in the heap

When you run:

```sql
SELECT * FROM users WHERE email = ?
```

PostgreSQL:

1. Searches the index
2. Follows the CTID of that Index to fetch the row from the heap

This is functionally also a double read, but the pointer used is a physical location, not a primary-key value. 

## So They Both Perform Double Lookups?

Both Mysql and Pgsql both perform double lookups but they can avoid this when the query is an index only query. A good example of an index only query is the one below.  

```sql
SELECT email FROM users WHERE email = ?
```

In this case, both Dbms (especially postgres) can avoid going to the clustered index or heap to retrieve the row because all the data it needs for that query can be retrieved from the index.

In theory, both databases *can* avoid touching the full row and simply return the value directly from the index.

But in practice, they behave very differently:

### **In MySQL (InnoDB)**

Even if the index contains all the columns we need, MySQL usually still has to read the clustered index to check things like:

- whether the row is visible
- whether it has been deleted
- whether the record version is current

So MySQL *rarely eliminates the double lookup*.

It almost always does:

1. Read secondary index
2. Read clustered index (double read)

Meaning: MySQL *almost never* achieves a true index-only scan.

---

### **In PostgreSQL**

PostgreSQL can actually avoid the second lookup, but *only* if two conditions are met:

1. The index contains all the columns the query needs
2. The visibility map says the heap page is “all-visible”

When these conditions are satisfied, Postgres will:

1. Read index
2. **Skip** reading the heap entirely

Indexes are one of the most powerful tools in your database toolbox. They can make queries blazing fast, but the underlying behavior differs across the different DBMS we have.

Understanding these differences can help you **design better indexes, write more efficient queries, and avoid unnecessary IO.**