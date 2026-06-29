# Public Library Consortium — Full SQL Source 



---

## 📄 `sql/01_member1_schema_books_copies.sql`
**Owner: M.Waqar Altaf — Schema Foundation, Constraints, CRUD, JOINs**

```sql
USE PublicLibraryConsortium;
GO

-- ===== Books Table =====
CREATE TABLE Books (
    book_id INT IDENTITY(1,1) PRIMARY KEY,
    isbn VARCHAR(20) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    author VARCHAR(150) NOT NULL,
    genre VARCHAR(100) NULL,
    publish_year INT NULL
);
GO

-- ===== Book_Copies Table =====
CREATE TABLE Book_Copies (
    copy_id INT IDENTITY(1,1) PRIMARY KEY,
    book_id INT NOT NULL,
    barcode_label VARCHAR(30) NOT NULL UNIQUE,
    copy_condition VARCHAR(20) NOT NULL DEFAULT 'Good'
        CONSTRAINT CK_Copies_Condition CHECK (copy_condition IN ('New','Good','Worn','Damaged')),
    availability_status VARCHAR(20) NOT NULL DEFAULT 'Available'
        CONSTRAINT CK_Copies_Availability CHECK (availability_status IN ('Available','On Loan')),
    CONSTRAINT FK_Copies_Books FOREIGN KEY (book_id)
        REFERENCES Books(book_id) ON DELETE NO ACTION ON UPDATE NO ACTION
);
GO

-- ===== CRUD Demonstration =====

-- INSERT
INSERT INTO Books (isbn, title, author, genre, publish_year)
VALUES ('978-1234567897','Brave New World','Aldous Huxley','Dystopian',1932);

-- UPDATE
UPDATE Book_Copies SET copy_condition = 'Worn' WHERE copy_id = 8;

-- DELETE (expected to fail once loans exist — proves FK protection)
DELETE FROM Book_Copies WHERE copy_id = 3;

-- ===== JOIN Queries =====

-- INNER JOIN: every copy with its book title and status
SELECT b.title, bc.barcode_label, bc.availability_status
FROM Books b
INNER JOIN Book_Copies bc ON b.book_id = bc.book_id;

-- LEFT JOIN: every copy, with borrower shown only if currently on loan
SELECT b.title, bc.barcode_label, m.first_name + ' ' + m.last_name AS borrower
FROM Books b
INNER JOIN Book_Copies bc ON b.book_id = bc.book_id
LEFT JOIN Book_Loans bl ON bc.copy_id = bl.copy_id AND bl.return_date IS NULL
LEFT JOIN Members m ON bl.member_id = m.member_id;

-- ===== Constraint Tests =====

-- Duplicate ISBN test — should FAIL (UNIQUE violation)
INSERT INTO Books (isbn, title, author) VALUES ('978-1234567897','Duplicate Test','Anon');

-- Delete blocked by FK test — should FAIL (referential integrity)
DELETE FROM Books WHERE book_id = 2;
```

---

## 📄 `sql/02_member2_members_procedures.sql`
**Owner: Moarij Irfan — Members Table, Stored Procedures, Transactions**

```sql
USE PublicLibraryConsortium;
GO

-- ===== Members Table =====
CREATE TABLE Members (
    member_id INT IDENTITY(1,1) PRIMARY KEY,
    first_name VARCHAR(80) NOT NULL,
    last_name VARCHAR(80) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    phone VARCHAR(20) NULL,
    member_status VARCHAR(20) NOT NULL
        CONSTRAINT CK_Members_Status CHECK (member_status IN ('Active','Suspended','Expired')),
    join_date DATE NOT NULL DEFAULT GETDATE()
);
GO

-- ===== sp_CheckoutBookCopy =====
CREATE PROCEDURE sp_CheckoutBookCopy
    @MemberID INT, @CopyID INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
        DECLARE @MemberStatus VARCHAR(20), @CopyStatus VARCHAR(20);
        SELECT @MemberStatus = member_status FROM Members WHERE member_id = @MemberID;
        SELECT @CopyStatus = availability_status FROM Book_Copies WHERE copy_id = @CopyID;

        IF @MemberStatus IS NULL RAISERROR('Member not found.', 16, 1);
        IF @MemberStatus <> 'Active' RAISERROR('Checkout denied: member is not Active.', 16, 1);
        IF @CopyStatus IS NULL RAISERROR('Copy not found.', 16, 1);
        IF @CopyStatus <> 'Available' RAISERROR('Checkout denied: copy is not Available.', 16, 1);

        INSERT INTO Book_Loans (copy_id, member_id, loan_date, due_date)
        VALUES (@CopyID, @MemberID, GETDATE(), DATEADD(DAY, 14, GETDATE()));

        UPDATE Book_Copies SET availability_status = 'On Loan' WHERE copy_id = @CopyID;

        COMMIT TRANSACTION;
        PRINT 'Checkout successful.';
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
        PRINT 'Checkout failed: ' + ERROR_MESSAGE();
    END CATCH
END;
GO

-- ===== sp_ReturnBookCopy =====
CREATE PROCEDURE sp_ReturnBookCopy
    @LoanID INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
        DECLARE @CopyID INT, @AlreadyReturned DATE;
        SELECT @CopyID = copy_id, @AlreadyReturned = return_date
        FROM Book_Loans WHERE loan_id = @LoanID;

        IF @CopyID IS NULL RAISERROR('Loan record not found.', 16, 1);
        IF @AlreadyReturned IS NOT NULL RAISERROR('This loan has already been returned.', 16, 1);

        UPDATE Book_Loans SET return_date = GETDATE() WHERE loan_id = @LoanID;
        UPDATE Book_Copies SET availability_status = 'Available' WHERE copy_id = @CopyID;

        COMMIT TRANSACTION;
        PRINT 'Return processed successfully.';
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
        PRINT 'Return failed: ' + ERROR_MESSAGE();
    END CATCH
END;
GO

-- ===== Procedure Test Cases =====
EXEC sp_CheckoutBookCopy @MemberID = 1, @CopyID = 30;   -- valid → succeed
EXEC sp_CheckoutBookCopy @MemberID = 4, @CopyID = 29;   -- Suspended member → rejected
EXEC sp_CheckoutBookCopy @MemberID = 2, @CopyID = 3;    -- copy already On Loan → rejected
EXEC sp_ReturnBookCopy @LoanID = 1;                     -- valid unreturned loan → succeed
EXEC sp_ReturnBookCopy @LoanID = 1;                     -- already returned → rejected
```

---

## 📄 `sql/03_member3_loans_views_subqueries.sql`
**Owner: M.Sami Ullah Yousaf — Book_Loans, Views, Subqueries/Aggregates, Integration**

```sql
USE PublicLibraryConsortium;
GO

-- ===== Book_Loans Table =====
CREATE TABLE Book_Loans (
    loan_id INT IDENTITY(1,1) PRIMARY KEY,
    copy_id INT NOT NULL,
    member_id INT NOT NULL,
    loan_date DATE NOT NULL DEFAULT GETDATE(),
    due_date DATE NOT NULL,
    return_date DATE NULL,
    CONSTRAINT FK_Loans_Copies FOREIGN KEY (copy_id) REFERENCES Book_Copies(copy_id),
    CONSTRAINT FK_Loans_Members FOREIGN KEY (member_id) REFERENCES Members(member_id)
);
GO

-- ===== Views =====

CREATE VIEW vw_OverdueLoanFulfillment AS
SELECT b.title AS book_title, bc.barcode_label,
       m.first_name + ' ' + m.last_name AS borrower_name,
       bl.due_date, DATEDIFF(DAY, bl.due_date, GETDATE()) AS days_overdue
FROM Book_Loans bl
INNER JOIN Book_Copies bc ON bl.copy_id = bc.copy_id
INNER JOIN Books b ON bc.book_id = b.book_id
INNER JOIN Members m ON bl.member_id = m.member_id
WHERE bl.return_date IS NULL AND bl.due_date < CAST(GETDATE() AS DATE);
GO

CREATE VIEW vw_CatalogCirculationSummary AS
SELECT b.book_id, b.title,
       COUNT(DISTINCT bc.copy_id) AS total_copies_owned,
       COUNT(DISTINCT CASE WHEN bl.return_date IS NULL THEN bl.loan_id END) AS active_loans,
       COUNT(bl.loan_id) AS lifetime_total_loans
FROM Books b
LEFT JOIN Book_Copies bc ON b.book_id = bc.book_id
LEFT JOIN Book_Loans bl ON bc.copy_id = bl.copy_id
GROUP BY b.book_id, b.title;
GO

-- View outputs
SELECT * FROM vw_OverdueLoanFulfillment;
SELECT * FROM vw_CatalogCirculationSummary;

-- ===== Subqueries =====

-- Members who never borrowed
SELECT first_name, last_name FROM Members
WHERE member_id NOT IN (SELECT DISTINCT member_id FROM Book_Loans);

-- Books with zero available copies
SELECT title FROM Books
WHERE book_id IN (
    SELECT book_id FROM Book_Copies GROUP BY book_id
    HAVING SUM(CASE WHEN availability_status = 'Available' THEN 1 ELSE 0 END) = 0
);

-- Correlated subquery: loan count per member
SELECT m.first_name, m.last_name,
       (SELECT COUNT(*) FROM Book_Loans bl WHERE bl.member_id = m.member_id) AS total_loans
FROM Members m ORDER BY total_loans DESC;

-- ===== Aggregate Function =====

-- Genres with more than one book
SELECT genre, COUNT(*) AS book_count FROM Books
GROUP BY genre HAVING COUNT(*) > 1;

-- ===== Integration Consistency Check =====

-- Should return 0 rows — no copy on loan to more than one member at once
SELECT copy_id, COUNT(*) AS open_loans
FROM Book_Loans
WHERE return_date IS NULL
GROUP BY copy_id
HAVING COUNT(*) > 1;
```

---

## 📄 `sql/seed_data.sql`
**30 rows per table — run after all 4 tables are created, in this order**

```sql
USE PublicLibraryConsortium;
GO

-- ===== Books (30 rows) =====
INSERT INTO Books (isbn, title, author, genre, publish_year) VALUES
('978-1234567897','Brave New World','Aldous Huxley','Dystopian',1932),
('978-0451524935','1984','George Orwell','Dystopian',1949),
('978-0141439518','Pride and Prejudice','Jane Austen','Romance',1813),
('978-0061120084','To Kill a Mockingbird','Harper Lee','Fiction',1960),
('978-0743273565','The Great Gatsby','F. Scott Fitzgerald','Fiction',1925),
('978-0307277671','The Road','Cormac McCarthy','Fiction',2006),
('978-0142437247','Moby-Dick','Herman Melville','Adventure',1851),
('978-1400079988','War and Peace','Leo Tolstoy','Historical',1869),
('978-0486282114','Crime and Punishment','Fyodor Dostoevsky','Classic',1866),
('978-0142424179','The Catcher in the Rye','J.D. Salinger','Fiction',1951),
('978-0156012195','The Little Prince','Antoine de Saint-Exupery','Fantasy',1943),
('978-0747532699','Harry Potter and the Philosophers Stone','J.K. Rowling','Fantasy',1997),
('978-0618640157','The Lord of the Rings','J.R.R. Tolkien','Fantasy',1954),
('978-0544003415','The Hobbit','J.R.R. Tolkien','Fantasy',1937),
('978-0345391803','The Hitchhikers Guide to the Galaxy','Douglas Adams','Science Fiction',1979),
('978-0553293357','Foundation','Isaac Asimov','Science Fiction',1951),
('978-0441172719','Dune','Frank Herbert','Science Fiction',1965),
('978-0316769488','The Catcher Diaries','Salinger Estate','Fiction',1962),
('978-0671027032','A Game of Thrones','George R.R. Martin','Fantasy',1996),
('978-0099908401','Gone with the Wind','Margaret Mitchell','Historical',1936),
('978-0141441146','Jane Eyre','Charlotte Bronte','Romance',1847),
('978-0141439556','Wuthering Heights','Emily Bronte','Romance',1847),
('978-0140268867','The Odyssey','Homer','Classic',-800),
('978-0141439471','Frankenstein','Mary Shelley','Horror',1818),
('978-0141439846','Dracula','Bram Stoker','Horror',1897),
('978-0141442464','The Picture of Dorian Gray','Oscar Wilde','Classic',1890),
('978-0062316097','Sapiens','Yuval Noah Harari','Non-Fiction',2011),
('978-0399590504','Educated','Tara Westover','Non-Fiction',2018),
('978-0735211292','Atomic Habits','James Clear','Self-Help',2018),
('978-0061122415','The Alchemist','Paulo Coelho','Fiction',1988);

-- ===== Book_Copies (30 rows) =====
INSERT INTO Book_Copies (book_id, barcode_label, copy_condition, availability_status) VALUES
(1,'BC-1001','Good','Available'),
(1,'BC-1002','New','Available'),
(2,'BC-1003','Good','On Loan'),
(2,'BC-1004','Worn','Available'),
(3,'BC-1005','Good','Available'),
(4,'BC-1006','New','On Loan'),
(5,'BC-1007','Good','Available'),
(6,'BC-1008','Good','Available'),
(7,'BC-1009','Worn','Available'),
(8,'BC-1010','Good','Available'),
(8,'BC-1011','New','Available'),
(3,'BC-1012','Good','Available'),
(9,'BC-1013','Good','On Loan'),
(10,'BC-1014','Good','Available'),
(10,'BC-1015','New','Available'),
(11,'BC-1016','Good','Available'),
(12,'BC-1017','Good','On Loan'),
(12,'BC-1018','New','Available'),
(13,'BC-1019','Good','Available'),
(13,'BC-1020','Worn','Available'),
(14,'BC-1021','Good','On Loan'),
(15,'BC-1022','Good','Available'),
(16,'BC-1023','Good','Available'),
(17,'BC-1024','New','Available'),
(18,'BC-1025','Good','Available'),
(19,'BC-1026','Good','On Loan'),
(20,'BC-1027','Good','Available'),
(21,'BC-1028','Worn','Available'),
(22,'BC-1029','Good','Available'),
(23,'BC-1030','Good','Available');

-- ===== Members (30 rows) =====
INSERT INTO Members (first_name, last_name, email, phone, member_status) VALUES
('Ali','Khan','ali.khan@example.com','03001234567','Active'),
('Sara','Ahmed','sara.ahmed@example.com','03007654321','Active'),
('Bilal','Iqbal','bilal.iqbal@example.com',NULL,'Active'),
('Hina','Malik','hina.malik@example.com',NULL,'Suspended'),
('Usman','Raza','usman.raza@example.com',NULL,'Expired'),
('Fatima','Sheikh','fatima.sheikh@example.com',NULL,'Active'),
('Zain','Tariq','zain.tariq@example.com',NULL,'Active'),
('Ayesha','Noor','ayesha.noor@example.com',NULL,'Suspended'),
('Hamza','Saeed','hamza.saeed@example.com',NULL,'Active'),
('Mariam','Yousaf','mariam.yousaf@example.com',NULL,'Active'),
('Omar','Farooq','omar.farooq@example.com',NULL,'Expired'),
('Noor','Fatima','noor.fatima@example.com',NULL,'Active'),
('Imran','Sheikh','imran.sheikh@example.com',NULL,'Active'),
('Sana','Riaz','sana.riaz@example.com',NULL,'Active'),
('Tariq','Mehmood','tariq.mehmood@example.com',NULL,'Suspended'),
('Rabia','Aslam','rabia.aslam@example.com',NULL,'Active'),
('Faisal','Hussain','faisal.hussain@example.com',NULL,'Active'),
('Amna','Siddiqui','amna.siddiqui@example.com',NULL,'Active'),
('Kamran','Abbasi','kamran.abbasi@example.com',NULL,'Expired'),
('Lubna','Anwar','lubna.anwar@example.com',NULL,'Active'),
('Saad','Jamil','saad.jamil@example.com',NULL,'Active'),
('Zara','Hashmi','zara.hashmi@example.com',NULL,'Suspended'),
('Adeel','Murtaza','adeel.murtaza@example.com',NULL,'Active'),
('Maham','Qureshi','maham.qureshi@example.com',NULL,'Active'),
('Junaid','Aziz','junaid.aziz@example.com',NULL,'Active'),
('Iqra','Bashir','iqra.bashir@example.com',NULL,'Active'),
('Waqas','Shah','waqas.shah@example.com',NULL,'Expired'),
('Nida','Akram','nida.akram@example.com',NULL,'Active'),
('Asad','Latif','asad.latif@example.com',NULL,'Active'),
('Sidra','Naveed','sidra.naveed@example.com',NULL,'Active');

-- ===== Book_Loans (30 rows) =====
INSERT INTO Book_Loans (copy_id, member_id, loan_date, due_date, return_date) VALUES
(3,1,GETDATE()-20,GETDATE()-6,NULL),
(6,2,GETDATE()-10,GETDATE()+4,NULL),
(13,3,GETDATE()-30,GETDATE()-16,GETDATE()-2),
(17,6,GETDATE()-25,GETDATE()-11,GETDATE()-10),
(21,7,GETDATE()-5,GETDATE()+9,NULL),
(26,9,GETDATE()-40,GETDATE()-26,GETDATE()-25),
(3,10,GETDATE()-60,GETDATE()-46,GETDATE()-45),
(6,12,GETDATE()-55,GETDATE()-41,GETDATE()-40),
(13,13,GETDATE()-50,GETDATE()-36,GETDATE()-35),
(17,14,GETDATE()-45,GETDATE()-31,GETDATE()-30),
(21,16,GETDATE()-35,GETDATE()-21,GETDATE()-20),
(26,17,GETDATE()-15,GETDATE()-1,GETDATE()-1),
(1,18,GETDATE()-9,GETDATE()+5,GETDATE()-1),
(2,19,GETDATE()-70,GETDATE()-56,GETDATE()-55),
(4,20,GETDATE()-65,GETDATE()-51,GETDATE()-50),
(5,22,GETDATE()-3,GETDATE()+11,NULL),
(7,23,GETDATE()-80,GETDATE()-66,GETDATE()-65),
(8,24,GETDATE()-75,GETDATE()-61,GETDATE()-60),
(9,25,GETDATE()-2,GETDATE()+12,NULL),
(10,26,GETDATE()-90,GETDATE()-76,GETDATE()-75),
(11,28,GETDATE()-85,GETDATE()-71,GETDATE()-70),
(12,29,GETDATE()-1,GETDATE()+13,NULL),
(14,30,GETDATE()-100,GETDATE()-86,GETDATE()-85),
(15,1,GETDATE()-95,GETDATE()-81,GETDATE()-80),
(16,2,GETDATE()-4,GETDATE()+10,NULL),
(18,3,GETDATE()-110,GETDATE()-96,GETDATE()-95),
(19,6,GETDATE()-105,GETDATE()-91,GETDATE()-90),
(20,7,GETDATE()-6,GETDATE()+8,NULL),
(22,9,GETDATE()-120,GETDATE()-106,GETDATE()-105),
(23,10,GETDATE()-115,GETDATE()-101,GETDATE()-100);
```

---

## 📄 `sql/04_master_script.sql`
**The single file that builds the entire project from a blank server — run this top-to-bottom to recreate everything**

```sql
CREATE DATABASE PublicLibraryConsortium;
GO
USE PublicLibraryConsortium;
GO

-- ============================================================
-- 1. TABLES (dependency order: Books → Book_Copies → Members → Book_Loans)
-- ============================================================

CREATE TABLE Books (
    book_id INT IDENTITY(1,1) PRIMARY KEY,
    isbn VARCHAR(20) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    author VARCHAR(150) NOT NULL,
    genre VARCHAR(100) NULL,
    publish_year INT NULL
);
GO

CREATE TABLE Book_Copies (
    copy_id INT IDENTITY(1,1) PRIMARY KEY,
    book_id INT NOT NULL,
    barcode_label VARCHAR(30) NOT NULL UNIQUE,
    copy_condition VARCHAR(20) NOT NULL DEFAULT 'Good'
        CONSTRAINT CK_Copies_Condition CHECK (copy_condition IN ('New','Good','Worn','Damaged')),
    availability_status VARCHAR(20) NOT NULL DEFAULT 'Available'
        CONSTRAINT CK_Copies_Availability CHECK (availability_status IN ('Available','On Loan')),
    CONSTRAINT FK_Copies_Books FOREIGN KEY (book_id)
        REFERENCES Books(book_id) ON DELETE NO ACTION ON UPDATE NO ACTION
);
GO

CREATE TABLE Members (
    member_id INT IDENTITY(1,1) PRIMARY KEY,
    first_name VARCHAR(80) NOT NULL,
    last_name VARCHAR(80) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    phone VARCHAR(20) NULL,
    member_status VARCHAR(20) NOT NULL
        CONSTRAINT CK_Members_Status CHECK (member_status IN ('Active','Suspended','Expired')),
    join_date DATE NOT NULL DEFAULT GETDATE()
);
GO

CREATE TABLE Book_Loans (
    loan_id INT IDENTITY(1,1) PRIMARY KEY,
    copy_id INT NOT NULL,
    member_id INT NOT NULL,
    loan_date DATE NOT NULL DEFAULT GETDATE(),
    due_date DATE NOT NULL,
    return_date DATE NULL,
    CONSTRAINT FK_Loans_Copies FOREIGN KEY (copy_id) REFERENCES Book_Copies(copy_id),
    CONSTRAINT FK_Loans_Members FOREIGN KEY (member_id) REFERENCES Members(member_id)
);
GO

-- ============================================================
-- 2. SEED DATA (Books → Book_Copies → Members → Book_Loans)
-- ============================================================

INSERT INTO Books (isbn, title, author, genre, publish_year) VALUES
('978-1234567897','Brave New World','Aldous Huxley','Dystopian',1932),
('978-0451524935','1984','George Orwell','Dystopian',1949),
('978-0141439518','Pride and Prejudice','Jane Austen','Romance',1813),
('978-0061120084','To Kill a Mockingbird','Harper Lee','Fiction',1960),
('978-0743273565','The Great Gatsby','F. Scott Fitzgerald','Fiction',1925),
('978-0307277671','The Road','Cormac McCarthy','Fiction',2006),
('978-0142437247','Moby-Dick','Herman Melville','Adventure',1851),
('978-1400079988','War and Peace','Leo Tolstoy','Historical',1869),
('978-0486282114','Crime and Punishment','Fyodor Dostoevsky','Classic',1866),
('978-0142424179','The Catcher in the Rye','J.D. Salinger','Fiction',1951),
('978-0156012195','The Little Prince','Antoine de Saint-Exupery','Fantasy',1943),
('978-0747532699','Harry Potter and the Philosophers Stone','J.K. Rowling','Fantasy',1997),
('978-0618640157','The Lord of the Rings','J.R.R. Tolkien','Fantasy',1954),
('978-0544003415','The Hobbit','J.R.R. Tolkien','Fantasy',1937),
('978-0345391803','The Hitchhikers Guide to the Galaxy','Douglas Adams','Science Fiction',1979),
('978-0553293357','Foundation','Isaac Asimov','Science Fiction',1951),
('978-0441172719','Dune','Frank Herbert','Science Fiction',1965),
('978-0316769488','The Catcher Diaries','Salinger Estate','Fiction',1962),
('978-0671027032','A Game of Thrones','George R.R. Martin','Fantasy',1996),
('978-0099908401','Gone with the Wind','Margaret Mitchell','Historical',1936),
('978-0141441146','Jane Eyre','Charlotte Bronte','Romance',1847),
('978-0141439556','Wuthering Heights','Emily Bronte','Romance',1847),
('978-0140268867','The Odyssey','Homer','Classic',-800),
('978-0141439471','Frankenstein','Mary Shelley','Horror',1818),
('978-0141439846','Dracula','Bram Stoker','Horror',1897),
('978-0141442464','The Picture of Dorian Gray','Oscar Wilde','Classic',1890),
('978-0062316097','Sapiens','Yuval Noah Harari','Non-Fiction',2011),
('978-0399590504','Educated','Tara Westover','Non-Fiction',2018),
('978-0735211292','Atomic Habits','James Clear','Self-Help',2018),
('978-0061122415','The Alchemist','Paulo Coelho','Fiction',1988);

INSERT INTO Book_Copies (book_id, barcode_label, copy_condition, availability_status) VALUES
(1,'BC-1001','Good','Available'),
(1,'BC-1002','New','Available'),
(2,'BC-1003','Good','On Loan'),
(2,'BC-1004','Worn','Available'),
(3,'BC-1005','Good','Available'),
(4,'BC-1006','New','On Loan'),
(5,'BC-1007','Good','Available'),
(6,'BC-1008','Good','Available'),
(7,'BC-1009','Worn','Available'),
(8,'BC-1010','Good','Available'),
(8,'BC-1011','New','Available'),
(3,'BC-1012','Good','Available'),
(9,'BC-1013','Good','On Loan'),
(10,'BC-1014','Good','Available'),
(10,'BC-1015','New','Available'),
(11,'BC-1016','Good','Available'),
(12,'BC-1017','Good','On Loan'),
(12,'BC-1018','New','Available'),
(13,'BC-1019','Good','Available'),
(13,'BC-1020','Worn','Available'),
(14,'BC-1021','Good','On Loan'),
(15,'BC-1022','Good','Available'),
(16,'BC-1023','Good','Available'),
(17,'BC-1024','New','Available'),
(18,'BC-1025','Good','Available'),
(19,'BC-1026','Good','On Loan'),
(20,'BC-1027','Good','Available'),
(21,'BC-1028','Worn','Available'),
(22,'BC-1029','Good','Available'),
(23,'BC-1030','Good','Available');

INSERT INTO Members (first_name, last_name, email, phone, member_status) VALUES
('Ali','Khan','ali.khan@example.com','03001234567','Active'),
('Sara','Ahmed','sara.ahmed@example.com','03007654321','Active'),
('Bilal','Iqbal','bilal.iqbal@example.com',NULL,'Active'),
('Hina','Malik','hina.malik@example.com',NULL,'Suspended'),
('Usman','Raza','usman.raza@example.com',NULL,'Expired'),
('Fatima','Sheikh','fatima.sheikh@example.com',NULL,'Active'),
('Zain','Tariq','zain.tariq@example.com',NULL,'Active'),
('Ayesha','Noor','ayesha.noor@example.com',NULL,'Suspended'),
('Hamza','Saeed','hamza.saeed@example.com',NULL,'Active'),
('Mariam','Yousaf','mariam.yousaf@example.com',NULL,'Active'),
('Omar','Farooq','omar.farooq@example.com',NULL,'Expired'),
('Noor','Fatima','noor.fatima@example.com',NULL,'Active'),
('Imran','Sheikh','imran.sheikh@example.com',NULL,'Active'),
('Sana','Riaz','sana.riaz@example.com',NULL,'Active'),
('Tariq','Mehmood','tariq.mehmood@example.com',NULL,'Suspended'),
('Rabia','Aslam','rabia.aslam@example.com',NULL,'Active'),
('Faisal','Hussain','faisal.hussain@example.com',NULL,'Active'),
('Amna','Siddiqui','amna.siddiqui@example.com',NULL,'Active'),
('Kamran','Abbasi','kamran.abbasi@example.com',NULL,'Expired'),
('Lubna','Anwar','lubna.anwar@example.com',NULL,'Active'),
('Saad','Jamil','saad.jamil@example.com',NULL,'Active'),
('Zara','Hashmi','zara.hashmi@example.com',NULL,'Suspended'),
('Adeel','Murtaza','adeel.murtaza@example.com',NULL,'Active'),
('Maham','Qureshi','maham.qureshi@example.com',NULL,'Active'),
('Junaid','Aziz','junaid.aziz@example.com',NULL,'Active'),
('Iqra','Bashir','iqra.bashir@example.com',NULL,'Active'),
('Waqas','Shah','waqas.shah@example.com',NULL,'Expired'),
('Nida','Akram','nida.akram@example.com',NULL,'Active'),
('Asad','Latif','asad.latif@example.com',NULL,'Active'),
('Sidra','Naveed','sidra.naveed@example.com',NULL,'Active');

INSERT INTO Book_Loans (copy_id, member_id, loan_date, due_date, return_date) VALUES
(3,1,GETDATE()-20,GETDATE()-6,NULL),
(6,2,GETDATE()-10,GETDATE()+4,NULL),
(13,3,GETDATE()-30,GETDATE()-16,GETDATE()-2),
(17,6,GETDATE()-25,GETDATE()-11,GETDATE()-10),
(21,7,GETDATE()-5,GETDATE()+9,NULL),
(26,9,GETDATE()-40,GETDATE()-26,GETDATE()-25),
(3,10,GETDATE()-60,GETDATE()-46,GETDATE()-45),
(6,12,GETDATE()-55,GETDATE()-41,GETDATE()-40),
(13,13,GETDATE()-50,GETDATE()-36,GETDATE()-35),
(17,14,GETDATE()-45,GETDATE()-31,GETDATE()-30),
(21,16,GETDATE()-35,GETDATE()-21,GETDATE()-20),
(26,17,GETDATE()-15,GETDATE()-1,GETDATE()-1),
(1,18,GETDATE()-9,GETDATE()+5,GETDATE()-1),
(2,19,GETDATE()-70,GETDATE()-56,GETDATE()-55),
(4,20,GETDATE()-65,GETDATE()-51,GETDATE()-50),
(5,22,GETDATE()-3,GETDATE()+11,NULL),
(7,23,GETDATE()-80,GETDATE()-66,GETDATE()-65),
(8,24,GETDATE()-75,GETDATE()-61,GETDATE()-60),
(9,25,GETDATE()-2,GETDATE()+12,NULL),
(10,26,GETDATE()-90,GETDATE()-76,GETDATE()-75),
(11,28,GETDATE()-85,GETDATE()-71,GETDATE()-70),
(12,29,GETDATE()-1,GETDATE()+13,NULL),
(14,30,GETDATE()-100,GETDATE()-86,GETDATE()-85),
(15,1,GETDATE()-95,GETDATE()-81,GETDATE()-80),
(16,2,GETDATE()-4,GETDATE()+10,NULL),
(18,3,GETDATE()-110,GETDATE()-96,GETDATE()-95),
(19,6,GETDATE()-105,GETDATE()-91,GETDATE()-90),
(20,7,GETDATE()-6,GETDATE()+8,NULL),
(22,9,GETDATE()-120,GETDATE()-106,GETDATE()-105),
(23,10,GETDATE()-115,GETDATE()-101,GETDATE()-100);

-- ============================================================
-- 3. STORED PROCEDURES
-- ============================================================

CREATE PROCEDURE sp_CheckoutBookCopy
    @MemberID INT, @CopyID INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
        DECLARE @MemberStatus VARCHAR(20), @CopyStatus VARCHAR(20);
        SELECT @MemberStatus = member_status FROM Members WHERE member_id = @MemberID;
        SELECT @CopyStatus = availability_status FROM Book_Copies WHERE copy_id = @CopyID;

        IF @MemberStatus IS NULL RAISERROR('Member not found.', 16, 1);
        IF @MemberStatus <> 'Active' RAISERROR('Checkout denied: member is not Active.', 16, 1);
        IF @CopyStatus IS NULL RAISERROR('Copy not found.', 16, 1);
        IF @CopyStatus <> 'Available' RAISERROR('Checkout denied: copy is not Available.', 16, 1);

        INSERT INTO Book_Loans (copy_id, member_id, loan_date, due_date)
        VALUES (@CopyID, @MemberID, GETDATE(), DATEADD(DAY, 14, GETDATE()));

        UPDATE Book_Copies SET availability_status = 'On Loan' WHERE copy_id = @CopyID;

        COMMIT TRANSACTION;
        PRINT 'Checkout successful.';
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
        PRINT 'Checkout failed: ' + ERROR_MESSAGE();
    END CATCH
END;
GO

CREATE PROCEDURE sp_ReturnBookCopy
    @LoanID INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
        DECLARE @CopyID INT, @AlreadyReturned DATE;
        SELECT @CopyID = copy_id, @AlreadyReturned = return_date
        FROM Book_Loans WHERE loan_id = @LoanID;

        IF @CopyID IS NULL RAISERROR('Loan record not found.', 16, 1);
        IF @AlreadyReturned IS NOT NULL RAISERROR('This loan has already been returned.', 16, 1);

        UPDATE Book_Loans SET return_date = GETDATE() WHERE loan_id = @LoanID;
        UPDATE Book_Copies SET availability_status = 'Available' WHERE copy_id = @CopyID;

        COMMIT TRANSACTION;
        PRINT 'Return processed successfully.';
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
        PRINT 'Return failed: ' + ERROR_MESSAGE();
    END CATCH
END;
GO

-- ============================================================
-- 4. VIEWS
-- ============================================================

CREATE VIEW vw_OverdueLoanFulfillment AS
SELECT b.title AS book_title, bc.barcode_label,
       m.first_name + ' ' + m.last_name AS borrower_name,
       bl.due_date, DATEDIFF(DAY, bl.due_date, GETDATE()) AS days_overdue
FROM Book_Loans bl
INNER JOIN Book_Copies bc ON bl.copy_id = bc.copy_id
INNER JOIN Books b ON bc.book_id = b.book_id
INNER JOIN Members m ON bl.member_id = m.member_id
WHERE bl.return_date IS NULL AND bl.due_date < CAST(GETDATE() AS DATE);
GO

CREATE VIEW vw_CatalogCirculationSummary AS
SELECT b.book_id, b.title,
       COUNT(DISTINCT bc.copy_id) AS total_copies_owned,
       COUNT(DISTINCT CASE WHEN bl.return_date IS NULL THEN bl.loan_id END) AS active_loans,
       COUNT(bl.loan_id) AS lifetime_total_loans
FROM Books b
LEFT JOIN Book_Copies bc ON b.book_id = bc.book_id
LEFT JOIN Book_Loans bl ON bc.copy_id = bl.copy_id
GROUP BY b.book_id, b.title;
GO
```
