
-- ==========================================================
-- PART 1: FeePayments Table – Transactions & ACID
-- ==========================================================

-- Drop and recreate the FeePayments table
DROP TABLE IF EXISTS FeePayments;

CREATE TABLE FeePayments (
    payment_id INT PRIMARY KEY,
    student_name VARCHAR(100) NOT NULL,
    amount DECIMAL(10,2) NOT NULL CHECK (amount > 0),
    payment_date DATE NOT NULL
);

--------------------------------------------------------------
-- A. Insert 5 Fee Payments (Atomic Insert)
--------------------------------------------------------------
START TRANSACTION;

INSERT INTO FeePayments (payment_id, student_name, amount, payment_date) VALUES
    (1, 'Ashish',  5000.00, '2024-06-01'),
    (2, 'Smaran',  4500.00, '2024-06-02'),
    (3, 'Vaibhav', 5500.00, '2024-06-03'),
    (7, 'Sneha',   4700.00, '2024-06-04'),
    (8, 'Arjun',   4900.00, '2024-06-05');

COMMIT;

SELECT * FROM FeePayments;

--------------------------------------------------------------
-- B. Demonstrate ROLLBACK (Failure: duplicate + negative)
--------------------------------------------------------------
DELIMITER $$

CREATE PROCEDURE demo_partB()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
    END;

    START TRANSACTION;
        INSERT INTO FeePayments VALUES (9, 'Kiran', 5200.00, '2024-06-06');
        -- Invalid: Duplicate ID (1) + negative amount
        INSERT INTO FeePayments VALUES (1, 'Neha', -3000.00, '2024-06-07');
    COMMIT;
END$$
DELIMITER ;

CALL demo_partB();
DROP PROCEDURE demo_partB;

-- Verify rollback (Kiran should NOT be present)
SELECT * FROM FeePayments;

--------------------------------------------------------------
-- C. Partial Failure (One valid + one invalid)
--------------------------------------------------------------
DELIMITER $$

CREATE PROCEDURE demo_partC()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
    END;

    START TRANSACTION;
        INSERT INTO FeePayments VALUES (10, 'Ananya', 5100.00, '2024-06-08');
        -- Invalid: NULL student_name
        INSERT INTO FeePayments VALUES (11, NULL, 4200.00, '2024-06-09');
    COMMIT;
END$$
DELIMITER ;

CALL demo_partC();
DROP PROCEDURE demo_partC;

-- Verify rollback (Ananya should NOT be present)
SELECT * FROM FeePayments;

--------------------------------------------------------------
-- D. Verify ACID Compliance
--------------------------------------------------------------
-- Valid transaction
START TRANSACTION;
    INSERT INTO FeePayments VALUES (12, 'Manish', 5300.00, '2024-06-10');
COMMIT;

-- Invalid transaction (Duplicate ID = 12)
DELIMITER $$
CREATE PROCEDURE demo_partD()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
    END;

    START TRANSACTION;
        INSERT INTO FeePayments VALUES (12, 'Tina', 4700.00, '2024-06-11'); -- Duplicate
    COMMIT;
END$$
DELIMITER ;

CALL demo_partD();
DROP PROCEDURE demo_partD;

-- Final state of the table
SELECT * FROM FeePayments ORDER BY payment_id;

-- ==========================================================
-- PART 2: StudentEnrollments Table – Transactions & Locks
-- ==========================================================

DROP TABLE IF EXISTS StudentEnrollments;

CREATE TABLE StudentEnrollments (
    enrollment_id  INT PRIMARY KEY,
    student_name   VARCHAR(100) NOT NULL,
    course_id      VARCHAR(10)  NOT NULL,
    enrollment_date DATE NOT NULL,
    UNIQUE(student_name, course_id)
);

--------------------------------------------------------------
-- A. Insert Sample Data + Unique Constraint Demo
--------------------------------------------------------------
START TRANSACTION;
INSERT INTO StudentEnrollments VALUES
    (1,'Ashish','CSE101','2024-07-01'),
    (2,'Smaran','CSE102','2024-07-01'),
    (3,'Vaibhav','CSE101','2024-07-01');
COMMIT;

-- Attempt duplicate (Fails, then rollback)
START TRANSACTION;
INSERT INTO StudentEnrollments VALUES
    (4,'Ashish','CSE101','2024-07-02'); -- ❌ Fails due to UNIQUE constraint
ROLLBACK;

SELECT * FROM StudentEnrollments;

--------------------------------------------------------------
-- B. Row Locking with SELECT FOR UPDATE
--------------------------------------------------------------
-- Session 1:
-- START TRANSACTION;
-- SELECT * FROM StudentEnrollments
-- WHERE student_name='Ashish' AND course_id='CSE101' FOR UPDATE;
-- (User B trying to update same row will BLOCK until commit)
-- COMMIT;

--------------------------------------------------------------
-- C. Update with Locking
--------------------------------------------------------------
START TRANSACTION;
UPDATE StudentEnrollments
SET enrollment_date='2024-07-04'
WHERE student_name='Ashish' AND course_id='CSE101';
COMMIT;

START TRANSACTION;
SELECT * FROM StudentEnrollments
WHERE student_name='Ashish' AND course_id='CSE101' FOR UPDATE;

UPDATE StudentEnrollments
SET enrollment_date='2024-07-05'
WHERE student_name='Ashish' AND course_id='CSE101';
COMMIT;

SELECT * FROM StudentEnrollments;

-- ==========================================================
-- PART 3: Transactions & Concurrency Control Practice
-- ==========================================================

-- Step 1: Create Table
DROP TABLE IF EXISTS StudentEnrollments;

CREATE TABLE StudentEnrollments (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100),
    course_id VARCHAR(10),
    enrollment_date DATE
);

-- Step 2: Insert sample data
INSERT INTO StudentEnrollments VALUES
(1, 'Ashish',  'CSE101', '2024-06-01'),
(2, 'Smaran',  'CSE102', '2024-06-01'),
(3, 'Vaibhav', 'CSE103', '2024-06-01');

SELECT * FROM StudentEnrollments;

--------------------------------------------------------------
-- A. Simulating Deadlock (Run in TWO Sessions)
--------------------------------------------------------------
-- SESSION 1:
-- START TRANSACTION;
-- UPDATE StudentEnrollments SET course_id = 'CSE201' WHERE student_id = 1;
-- (pause, then run next)
-- UPDATE StudentEnrollments SET course_id = 'CSE301' WHERE student_id = 2;

-- SESSION 2:
-- START TRANSACTION;
-- UPDATE StudentEnrollments SET course_id = 'CSE202' WHERE student_id = 2;
-- (pause, then run next)
-- UPDATE StudentEnrollments SET course_id = 'CSE302' WHERE student_id = 1;

-- Expected: One transaction is rolled back
-- ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction.

--------------------------------------------------------------
-- B. MVCC Snapshot Read
--------------------------------------------------------------
-- SESSION 1 (Reader)
-- SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- START TRANSACTION;
-- SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- (Reader holds snapshot)

-- SESSION 2 (Writer)
-- START TRANSACTION;
-- UPDATE StudentEnrollments SET enrollment_date = '2024-07-10' WHERE student_id = 1;
-- COMMIT;

-- Back to SESSION 1
-- SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- COMMIT;

-- Expected:
-- Session 1 keeps seeing enrollment_date = '2024-06-01'
-- Session 2 successfully updates to '2024-07-10'

--------------------------------------------------------------
-- C. Locking vs MVCC Comparison
--------------------------------------------------------------
-- Case 1: Locking with SELECT FOR UPDATE
-- SESSION 1:
-- START TRANSACTION;
-- SELECT * FROM StudentEnrollments WHERE student_id = 1 FOR UPDATE;

-- SESSION 2:
-- UPDATE StudentEnrollments SET course_id = 'CSE401' WHERE student_id = 1;
-- (❌ BLOCKED until Session 1 commits)

-- Case 2: MVCC (No Blocking)
-- SESSION 1:
-- SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- START TRANSACTION;
-- SELECT * FROM StudentEnrollments WHERE student_id = 1;

-- SESSION 2:
-- UPDATE StudentEnrollments SET course_id = 'CSE402' WHERE student_id = 1;
-- COMMIT;

-- Back to Session 1:
-- SELECT * FROM StudentEnrollments WHERE student_id = 1;
-- COMMIT;

-- Expected:
-- ❌ With Locking: Writer waits for reader.
-- ✅ With MVCC: Reader sees old snapshot, writer updates concurrently.
