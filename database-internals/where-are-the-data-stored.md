# Where Are They Stored?

It's a no-brainer that our databases exist to store data. But it begs the question, where is the data stored? Well, a number of these DBMS store them on the disk, some store them in memory, but it's beyond the scope of this writeup.

In this writeup we will be focusing on DBMS like MySQL and PostgreSQL, and how they store our data. In relational DBMS, our data are stored as **Rows** in a table. These data also have other definitions like **Columns**, **indexes** and a few other properties. See the users table below for your reference.

### Users Table

| Column         | Type           | Description                          |
|----------------|----------------|--------------------------------------|
| id             | UUID / INT     | Primary key, unique identifier       |
| first_name     | VARCHAR(100)   | User's first name                    |
| last_name      | VARCHAR(100)   | User's last name                     |
| email          | VARCHAR(255)   | User's email (unique)                |
| password_hash  | TEXT           | Hashed password                      |
| created_at     | TIMESTAMP      | When the user was created            |
| updated_at     | TIMESTAMP      | Last update timestamp                |

## **How It Saves Them**

In MySQL InnoDB, properties like our column names, foreign key references, and the rest of the metadata from the table above are stored in system tables in the InnoDB directory.

**NOTE:** So the more columns you add, the more storage you take up, and this is why designing database tables well is so important in my opinion.

The actual rows which contain our data are stored in **Pages**. _In MySQL, when using file-per-table mode, these are files that have the .ibd extension. In system tablespace mode, tables are stored in shared ibdata files._

So what are these pages? Well, for starters, they are structured differently in each DBMS, though they share some conceptual similarities. They have a fixed size based on the DBMS and can be tuned. _In MySQL the default page size is 16KB and in PostgreSQL the default page is 8KB._

This is why I can't over-emphasize the importance of designing our tables well, because we need to make sure data fits these blocks nicely to avoid fragmentation as much as possible.

While both MySQL and PostgreSQL use pages, their internal structures differ:

**PostgreSQL Heap Page Structure:**
```
+----------------------------+
| Page Header                |
+----------------------------+
| Line Pointer Array (slot0)|
| Line Pointer Array (slot1)|
| Line Pointer Array (slot2)|
| ...                        |
+----------------------------+
| Free Space                 |
| ...                        |
+----------------------------+
| Tuples (Row Data)          |
| (grow from bottom up)      |
| ...                        |
+----------------------------+
```

**MySQL InnoDB Page Structure:**
```
+----------------------------+
| Page Header                |
+----------------------------+
| Infimum Record             |
| Supremum Record            |
+----------------------------+
| User Records (Row Data)    |
| (stored in PK order)       |
| ...                        |
+----------------------------+
| Free Space                 |
| ...                        |
+----------------------------+
| Page Directory (Slots)     |
| ...                        |
+----------------------------+
``` 

### The Page Header

This section contains metadata information for the page itself. In both systems, it includes properties like the Page ID, checksum, and free space information. The exact structure differs between MySQL and PostgreSQL.

### Row Location Mechanisms

**In PostgreSQL:** Line Pointers (also called Item Pointers) are stored in an array at the top of the page. Each line pointer contains the offset and length of a tuple (row) in the page. They help quickly locate rows and track when rows are moved or updated.

**In MySQL InnoDB:** The Page Directory contains slots that point to records. Records are stored in primary key order, and the directory helps with binary search within the page. Records also contain next/previous pointers for the linked list structure.

### Free Space

Free space in pages is not necessarily continuous—it can become fragmented as rows are inserted, updated, and deleted. When you issue an insert, the database finds available space (which may be fragmented) to fit the new row. This space will keep storing more rows until it can no longer fit. In MySQL, when a page can no longer fit new rows, a page split occurs. In PostgreSQL, when a page becomes full, a new page is simply allocated—there's no page splitting mechanism.

### Row Data

This is where our actual data and row metadata live. This is where our user information is stored.

In MySQL, these data are stored in a clustered index, which means they are physically ordered by the primary key. For auto-incrementing primary keys, our data are stored in ascending order by the primary key value, which allows for efficient sequential inserts and range scans.

In PostgreSQL, the data are stored as a heap. Unlike MySQL, PostgreSQL does not cluster data by the primary key, so the physical order of rows is independent of the primary key order.

## Page Splits

Remember I mentioned earlier about page splits, so let's add more context to it. In the case of MySQL we have a default page size of 16KB, and a default page size of 8KB in PostgreSQL. 

Page splits in MySQL InnoDB primarily occur during INSERT operations. When inserting a new row, if the target page is full, MySQL performs a page split. Typically, the page is split roughly 50/50—half the rows stay in the original page, and half move to a new page. This maintains the clustered index order. If the split causes the parent index page to overflow, the split can propagate up the B-tree structure, potentially creating new levels in the index.

When a row is UPDATED and grows beyond its current space, InnoDB handles it differently: the old row is marked as deleted, and a new row version is inserted in the appropriate location (maintaining primary key order). A page split only occurs if the target page for inserting this new row version is full—the split is caused by the INSERT operation, not directly by the UPDATE.

In PostgreSQL, the data are stored as heaps, and it handles full pages differently. When a page becomes full, PostgreSQL simply allocates a new page to store additional rows—there's no page splitting like in MySQL's B-tree structure. 

Separately, for individual column values that exceed approximately 2KB, TOAST (The Oversized-Attribute Storage Technique) kicks in. Columns like TEXT, VARCHAR, BYTEA, and a few others are TOASTABLE. TOAST takes those large values and moves them to a separate storage location (the TOAST table), storing only a pointer in the heap. This reduces the row size significantly and is independent of page allocation—TOAST is about handling large individual column values, not about managing full pages.

## What Is The Impact Of Page Split?

So one of the worst things that can happen to a DB is a lot of page splits. In MySQL this is especially true due to its clustered nature. When we do an insert with an AUTO_INCREMENT primary key, MySQL inserts rows in primary key order. Since new rows have higher primary key values, they're inserted at the "end" of the clustered index structure, which means they typically go into the rightmost leaf pages. This is very efficient because splits are minimal (only the rightmost pages split), and we get predictability and good sequential disk writes.

In PostgreSQL, the whole thing is a heap, so PostgreSQL stores data in the heap and uses the Free Space Map (FSM) to track available space in pages. The FSM is a separate data structure (a tree of pages) that maintains information about free space in heap pages, allowing PostgreSQL to quickly find pages with enough space for new inserts without scanning all pages.

But what's the problem with this really? For tables like MySQL, when we use a primary key that is random (like a UUID), we lose those brilliant performance benefits out of the box, because our data is scattered everywhere in the page, and there's no order. It's inserted randomly, and this makes querying very expensive because the performance is not predictable. 


### Take Home And Conclusion

Yes, our database stores these tables on disk, but it's so important to know how these things are designed so we can be sure we design our schemas right and get the best out of them. In subsequent writeups, we'd talk about how our database handles CRUD operations under the hood, and how it relates to this topic. Till then, Sayonara!