--DROP
DROP VIEW v_student_dashboard;
DROP VIEW student_pending_assignments;
DROP VIEW student_daily_schedule;
DROP VIEW v_student_gpa;

DROP SEQUENCE seq_reg_id;

DROP TABLE Notifications CASCADE CONSTRAINTS;
DROP TABLE ReviewTeacher CASCADE CONSTRAINTS;
DROP TABLE AssignScore CASCADE CONSTRAINTS;
DROP TABLE Grade CASCADE CONSTRAINTS;
DROP TABLE Register CASCADE CONSTRAINTS;
DROP TABLE StudyRegister CASCADE CONSTRAINTS;
DROP TABLE StdAssignment CASCADE CONSTRAINTS;
DROP TABLE TeachAssignment CASCADE CONSTRAINTS;
DROP TABLE UserInfo CASCADE CONSTRAINTS;
DROP TABLE Subject CASCADE CONSTRAINTS;
DROP TABLE Major CASCADE CONSTRAINTS;
DROP TABLE Event CASCADE CONSTRAINTS;
DROP TABLE Teacher CASCADE CONSTRAINTS;
DROP TABLE Fact CASCADE CONSTRAINTS;

----------------------------------------------
--TABLES
CREATE TABLE Fact(
    FCode       VARCHAR2(4) PRIMARY KEY,
    FNameENG    VARCHAR2(256),
    FNameTHA    VARCHAR2(256)
);

CREATE TABLE Major(
    MjCode      VARCHAR2(4) PRIMARY KEY,
    MjNameENG   VARCHAR2(256),
    MjNameTHA   VARCHAR2(256),
    FCode       VARCHAR2(4),
    CONSTRAINT Major_fk_Fact FOREIGN KEY (FCode) REFERENCES Fact(FCode)
);

CREATE TABLE Teacher(
    TID         VARCHAR2(4) PRIMARY KEY,
    TFName      VARCHAR2(50),
    TLName      VARCHAR2(50),
    FCode       VARCHAR2(4),
    MjCode      VARCHAR2(4),
    CONSTRAINT Teacher_fk_Fact FOREIGN KEY (FCode) REFERENCES Fact(FCode),
    CONSTRAINT Teacher_fk_Major FOREIGN KEY (MjCode) REFERENCES Major(MjCode)
);

CREATE TABLE Event(
    EventID         VARCHAR2(4) PRIMARY KEY,
    EventName       VARCHAR2(150),
    EventStartDate  DATE,
    EventEndDate    DATE,
    EventType       VARCHAR2(50)
);

CREATE TABLE UserInfo(
    UserID      NUMBER(5) PRIMARY KEY,
    UserFName   VARCHAR2(50),
    UserLName   VARCHAR2(50),
    UserEmail   VARCHAR2(100),
    UserPass    VARCHAR2(100),
    FCode       VARCHAR2(4),
    MjCode      VARCHAR2(4),
    UserType    VARCHAR2(5),
    CONSTRAINT UserInfo_fk_Fact FOREIGN KEY (FCode) REFERENCES Fact(FCode),
    CONSTRAINT UserInfo_fk_Major FOREIGN KEY (MjCode) REFERENCES Major(MjCode)
);

CREATE TABLE Subject(
    SubjCode    VARCHAR2(8) PRIMARY KEY,
    SubjName    VARCHAR2(100),
    SubjCredit  NUMBER(2),
    FCode       VARCHAR2(4),
    SubjType    VARCHAR2(150),
    CONSTRAINT Subject_fk_fact FOREIGN KEY(FCode) REFERENCES Fact(FCode)
);

CREATE TABLE Register(
    RegID       VARCHAR2(10) PRIMARY KEY,
    UserID      NUMBER(5),
    EventID     VARCHAR2(4),
    CONSTRAINT Register_fk_UserInfo FOREIGN KEY(UserID) REFERENCES UserInfo(UserID),
    CONSTRAINT Register_fk_Event FOREIGN KEY(EventID) REFERENCES Event(EventID)
);

CREATE TABLE StdAssignment(
    AssignID    VARCHAR2(10) PRIMARY KEY,
    SubjCode    VARCHAR2(8),
    AssName     VARCHAR2(150),
    Dateline    DATE,
    Score       NUMBER(3),
    CONSTRAINT StdAssignment_fk_Subject FOREIGN KEY (SubjCode) REFERENCES Subject(SubjCode)
);

CREATE TABLE TeachAssignment(
    TAssignID  VARCHAR2(5) PRIMARY KEY,
    TID        VARCHAR2(4),
    SubjCode   VARCHAR2(8),
    Year       NUMBER(4),
    Semester   NUMBER(1),
    StudyDay   VARCHAR2(10),
    StartTime  TIMESTAMP,
    EndTime    TIMESTAMP,
    Room       VARCHAR2(20),
    CONSTRAINT chk_study_day CHECK (StudyDay IN ('MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY', 'FRIDAY', 'SATURDAY', 'SUNDAY')),
    CONSTRAINT chk_time_logic CHECK (EndTime > StartTime),
    CONSTRAINT TeachAssignment_fk_Teacher FOREIGN KEY (TID) REFERENCES Teacher(TID),
    CONSTRAINT TeachAssignment_fk_SubjCode FOREIGN KEY (SubjCode) REFERENCES Subject(SubjCode)
);

CREATE TABLE StudyRegister(
    StuRegisID      NUMBER(5) PRIMARY KEY, 
    TAssignID       VARCHAR2(5),
    UserID          NUMBER(5),
    Section         NUMBER(3), 
    StuRegisStatus  VARCHAR2(15),
    CONSTRAINT StudyRegister_fk_TeachAssignment FOREIGN KEY (TAssignID) REFERENCES TeachAssignment(TAssignID),
    CONSTRAINT StudyRegister_fk_UserInfo FOREIGN KEY (UserID) REFERENCES UserInfo(UserID)
);

CREATE TABLE Grade(
    GradeCode   NUMBER(5) PRIMARY KEY,
    StuRegisID  NUMBER(5),
    GradeResult VARCHAR2(2),
    CONSTRAINT Grade_fk_StudyRegister FOREIGN KEY(StuRegisID) REFERENCES StudyRegister(StuRegisID)
);

CREATE TABLE AssignScore(
    ScoreAssignID NUMBER(6) PRIMARY KEY,
    UserID        NUMBER(5),
    AssignID      VARCHAR2(10),
    Status        VARCHAR2(15),
    PriScore      NUMBER(3),
    CONSTRAINT AssignScore_fk_UserInfo FOREIGN KEY(UserID) REFERENCES UserInfo(UserID),
    CONSTRAINT AssignScore_fk_StdAssignment FOREIGN KEY(AssignID) REFERENCES StdAssignment(AssignID)
);

CREATE TABLE ReviewTeacher(
    TeachReview NUMBER(8) PRIMARY KEY,
    TAssignID   VARCHAR2(5),
    UserID      NUMBER(5),
    UserReviewT VARCHAR2(500),
    UserRateT   NUMBER(1),
    CONSTRAINT chk_review_rate CHECK (UserRateT BETWEEN 1 AND 5),
    CONSTRAINT ReviewTeacher_fk_TeachAssign FOREIGN KEY (TAssignID) REFERENCES TeachAssignment(TAssignID),
    CONSTRAINT ReviewTeacher_fk_UserInfo FOREIGN KEY (UserID) REFERENCES UserInfo(UserID)
);

CREATE TABLE Notifications (
    NotiID      NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    UserID      NUMBER(5),
    Message     VARCHAR2(500),
    NotiDate    DATE DEFAULT SYSDATE,
    Status      VARCHAR2(10) DEFAULT 'UNREAD',
    CONSTRAINT Noti_fk_User FOREIGN KEY (UserID) REFERENCES UserInfo(UserID)
);

------------------------------------------------
-- 3. VIEWS

CREATE OR REPLACE VIEW v_student_gpa AS
SELECT 
    u.UserID,
    u.UserFName || ' ' || u.UserLName AS FullName,
    ROUND(SUM(
        CASE g.GradeResult
            WHEN 'A'  THEN 4.0
            WHEN 'B+' THEN 3.5
            WHEN 'B'  THEN 3.0
            WHEN 'C+' THEN 2.5
            WHEN 'C'  THEN 2.0
            WHEN 'D+' THEN 1.5
            WHEN 'D'  THEN 1.0
            WHEN 'F'  THEN 0.0
            ELSE 0.0
        END * s.SubjCredit
    ) / NULLIF(SUM(s.SubjCredit), 0), 2) AS GPAX
FROM Grade g
JOIN StudyRegister sr ON g.StuRegisID = sr.StuRegisID
JOIN TeachAssignment ta ON sr.TAssignID = ta.TAssignID
JOIN Subject s ON ta.SubjCode = s.SubjCode
JOIN UserInfo u ON sr.UserID = u.UserID
GROUP BY u.UserID, u.UserFName, u.UserLName;

CREATE OR REPLACE VIEW student_daily_schedule AS
SELECT 
    u.UserID,
    u.UserFName || ' ' || u.UserLName AS FullName,
    s.SubjCode,
    s.SubjName,
    ta.StudyDay,
    TO_CHAR(ta.StartTime, 'HH24:MI') AS StartTime,
    TO_CHAR(ta.EndTime, 'HH24:MI') AS EndTime,
    EXTRACT(HOUR FROM (ta.EndTime - ta.StartTime)) AS StudyHours,
    ta.Room
FROM UserInfo u
JOIN StudyRegister sr ON u.UserID = sr.UserID
JOIN TeachAssignment ta ON sr.TAssignID = ta.TAssignID
JOIN Subject s ON ta.SubjCode = s.SubjCode
WHERE sr.StuRegisStatus = 'ENROLLED';

CREATE OR REPLACE VIEW student_pending_assignments AS
SELECT 
    sr.UserID,
    sa.AssignID,
    sa.AssName,
    s.SubjName,
    sa.Dateline,
    NVL(ans.Status, 'NOT SUBMITTED') as CurrentStatus
FROM StudyRegister sr
JOIN TeachAssignment ta ON sr.TAssignID = ta.TAssignID
JOIN Subject s ON ta.SubjCode = s.SubjCode
JOIN StdAssignment sa ON s.SubjCode = sa.SubjCode
LEFT JOIN AssignScore ans ON sr.UserID = ans.UserID AND sa.AssignID = ans.AssignID
WHERE (ans.Status IS NULL OR ans.Status != 'SUBMITTED')
  AND sa.Dateline >= TRUNC(SYSDATE);

CREATE OR REPLACE VIEW v_student_dashboard AS
SELECT 
    n.UserID,
    n.Message AS Notification,
    n.NotiDate,
    n.Status AS ReadStatus,
    'ALERT' AS Type
FROM Notifications n
UNION ALL
SELECT 
    UserID,
    '??????????: ' || AssName || ' (???? ' || SubjName || ')',
    Dateline,
    CurrentStatus,
    'PENDING_TASK' AS Type
FROM student_pending_assignments;

CREATE OR REPLACE VIEW v_faculty_student_count AS
SELECT 
    f.FCode,
    f.FNameENG AS FacultyName,
    m.MjNameENG AS MajorName,
    COUNT(u.UserID) AS TotalStudents
FROM Fact f
JOIN Major m ON f.FCode = m.FCode
LEFT JOIN UserInfo u ON m.MjCode = u.MjCode
WHERE u.UserType = 'STD' OR u.UserID IS NULL
GROUP BY f.FCode, f.FNameENG, m.MjNameENG
ORDER BY f.FCode, m.MjNameENG;

CREATE OR REPLACE VIEW v_student_event_summary AS
SELECT 
    u.UserID,
    u.UserFName || ' ' || u.UserLName AS FullName,
    e.EventName,
    e.EventStartDate,
    e.EventEndDate,
    e.EventType,
    CASE 
        WHEN e.EventEndDate < TRUNC(SYSDATE) THEN 'FINISHED'
        WHEN e.EventStartDate <= TRUNC(SYSDATE) AND e.EventEndDate >= TRUNC(SYSDATE) THEN 'ONGOING'
        ELSE 'UPCOMING'
    END AS EventStatus
FROM UserInfo u
JOIN Register r ON u.UserID = r.UserID
JOIN Event e ON r.EventID = e.EventID;
        
CREATE OR REPLACE VIEW v_teacher_performance_rating AS
SELECT 
    t.TID,
    t.TFName || ' ' || t.TLName AS TeacherName,
    s.SubjCode,
    s.SubjName,
    COUNT(rt.TeachReview) AS TotalReviews,
    ROUND(AVG(rt.UserRateT), 2) AS AverageRating
FROM Teacher t
JOIN TeachAssignment ta ON t.TID = ta.TID
JOIN Subject s ON ta.SubjCode = s.SubjCode
LEFT JOIN ReviewTeacher rt ON ta.TAssignID = rt.TAssignID
GROUP BY t.TID, t.TFName, t.TLName, s.SubjCode, s.SubjName;

CREATE OR REPLACE VIEW v_event_popularity AS
SELECT 
    e.EventID,
    e.EventName,
    e.EventType,
    e.EventStartDate,
    COUNT(r.RegID) AS TotalParticipants
FROM Event e
LEFT JOIN Register r ON e.EventID = r.EventID
GROUP BY e.EventID, e.EventName, e.EventType, e.EventStartDate
ORDER BY TotalParticipants DESC;

CREATE OR REPLACE VIEW v_student_feedbacks AS
SELECT 
    t.TFName || ' ' || t.TLName AS TeacherName,
    s.SubjCode,
    s.SubjName,
    rt.UserRateT AS StarRating,
    rt.UserReviewT AS ReviewComment
FROM ReviewTeacher rt
JOIN TeachAssignment ta ON rt.TAssignID = ta.TAssignID
JOIN Subject s ON ta.SubjCode = s.SubjCode
JOIN Teacher t ON ta.TID = t.TID
ORDER BY rt.TeachReview DESC;

CREATE OR REPLACE VIEW v_freshmen_events_board AS
SELECT 
    EventID,
    EventName,
    EventType,
    EventStartDate,
    EventEndDate,
    EventEndDate - TRUNC(SYSDATE) AS Days_Remaining
FROM Event
WHERE EventStartDate >= TRUNC(SYSDATE)
ORDER BY EventStartDate ASC;

----------------------------------------------
--SEQUENCE & PROCEDURES
CREATE SEQUENCE seq_reg_id START WITH 2 INCREMENT BY 1;

CREATE OR REPLACE PROCEDURE sp_get_student_alerts (
    p_user_id IN NUMBER
) AS
BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Daily Schedule ---');
    FOR rec IN (
        SELECT * FROM student_daily_schedule 
        WHERE UserID = p_user_id 
          AND StudyDay = TRIM(UPPER(TO_CHAR(SYSDATE, 'FMDAY', 'NLS_DATE_LANGUAGE=AMERICAN')))
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('Class: ' || rec.SubjName || ' at ' || rec.StartTime || ' Room: ' || rec.Room);
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('--- Pending Assignments ---');
    FOR rec IN (SELECT * FROM student_pending_assignments WHERE UserID = p_user_id) LOOP
        DBMS_OUTPUT.PUT_LINE('Task: ' || rec.AssName || ' | Deadline: ' || TO_CHAR(rec.Dateline, 'DD-MON-YYYY'));
    END LOOP;
END;
/

CREATE OR REPLACE PROCEDURE sp_safe_event_register (
    p_user_id IN NUMBER,
    p_event_id IN VARCHAR2
) AS
    v_conflict_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_conflict_count
    FROM student_daily_schedule v
    JOIN Event e ON e.EventID = p_event_id
    WHERE v.UserID = p_user_id 
      AND v.StudyDay = TRIM(UPPER(TO_CHAR(e.EventStartDate, 'FMDAY', 'NLS_DATE_LANGUAGE=AMERICAN')));

    IF v_conflict_count > 0 THEN
        DBMS_OUTPUT.PUT_LINE('Warning: This event conflicts with your class schedule!');
    ELSE
        INSERT INTO Register (RegID, UserID, EventID) 
        VALUES ('R'||LPAD(seq_reg_id.NEXTVAL, 4, '0'), p_user_id, p_event_id);
        COMMIT;
        DBMS_OUTPUT.PUT_LINE('Registration Successful.');
    END IF;
END;
/

CREATE OR REPLACE PROCEDURE sp_user_register (
    p_id IN NUMBER,
    p_fname IN VARCHAR2,
    p_lname IN VARCHAR2,
    p_email IN VARCHAR2,
    p_pass IN VARCHAR2,
    p_fcode IN VARCHAR2,
    p_mjcode IN VARCHAR2
) AS
BEGIN
    INSERT INTO UserInfo (UserID, UserFName, UserLName, UserEmail, UserPass, FCode, MjCode, UserType)
    VALUES (p_id, p_fname, p_lname, p_email, p_pass, p_fcode, p_mjcode, 'STD');
    COMMIT;
END;
/

CREATE OR REPLACE PROCEDURE sp_user_login (
    p_email IN VARCHAR2,
    p_pass IN VARCHAR2,
    p_status OUT VARCHAR2
) AS
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count 
    FROM UserInfo 
    WHERE UserEmail = p_email AND UserPass = p_pass;

    IF v_count > 0 THEN
        p_status := 'SUCCESS';
    ELSE
        p_status := 'FAILED';
    END IF;
END;
/

------------------------------------------------------------
--TRIGGERS
CREATE OR REPLACE TRIGGER trg_score_notification
AFTER UPDATE OF PriScore ON AssignScore
FOR EACH ROW
BEGIN
    INSERT INTO Notifications (UserID, Message, NotiDate)
    VALUES (:NEW.UserID, '?????????: ??? ' || :NEW.AssignID || ' ????????! ?????? ' || :NEW.PriScore || ' ?????', SYSDATE);
    
    DBMS_OUTPUT.PUT_LINE('Notification sent to UserID: ' || :NEW.UserID);
END;
/

CREATE OR REPLACE TRIGGER trg_new_assignment_notification
AFTER INSERT ON StdAssignment
FOR EACH ROW
BEGIN
    INSERT INTO Notifications (UserID, Message)
    SELECT sr.UserID, '??????????????????: ' || :NEW.AssName || ' (???? ' || :NEW.SubjCode || ') ???????? ' || TO_CHAR(:NEW.Dateline, 'DD/MM/YYYY')
    FROM StudyRegister sr
    JOIN TeachAssignment ta ON sr.TAssignID = ta.TAssignID
    WHERE ta.SubjCode = :NEW.SubjCode 
      AND sr.StuRegisStatus = 'ENROLLED';
END;
/

-----------------------------------------------
--TEST INSERT
-----------------------------------------------
--TEST INSERT
INSERT INTO Event (EventID, EventName, EventStartDate, EventEndDate, EventType) 
VALUES ('E001', 'Freshman Orientation', TO_DATE('2026-06-01', 'YYYY-MM-DD'), TO_DATE('2026-06-01', 'YYYY-MM-DD'), 'Academic');

INSERT INTO Fact (FCode, FNameENG, FNameTHA) VALUES ('F001', 'Science', 'Science');
INSERT INTO Major (MjCode, MjNameENG, MjNameTHA, FCode) VALUES ('M001', 'Computer Science', 'Computer Science', 'F001');

INSERT INTO Subject (SubjCode, SubjName, SubjCredit, FCode, SubjType) 
VALUES ('CP101001', 'Introduction to AI', 3, 'F001', 'Core Course');
INSERT INTO Subject (SubjCode, SubjName, SubjCredit, FCode, SubjType) 
VALUES ('CP101002', 'Database Systems', 3, 'F001', 'Core Course');

-- ใช้ INSERT Teacher ชุดใหม่ที่มี FCode, MjCode
INSERT INTO Teacher (TID, TFName, TLName, FCode, MjCode) VALUES ('T001', 'Obi-Wan', 'Kenobi', 'F001', 'M001');
INSERT INTO Teacher (TID, TFName, TLName, FCode, MjCode) VALUES ('T002', 'Yoda', 'Master', 'F001', 'M001');

INSERT INTO UserInfo (UserID, UserFName, UserLName, UserEmail, UserPass, FCode, MjCode, UserType) 
VALUES (67001, 'Anakin', 'Skywalker', 'anakin@kku.ac.th', 'pass123', 'F001', 'M001', 'STD');

-- ใช้ INSERT TeachAssignment ชุดใหม่ที่เป็น TO_TIMESTAMP
INSERT INTO TeachAssignment (TAssignID, TID, SubjCode, Year, Semester, StudyDay, StartTime, EndTime, Room) 
VALUES ('TA001', 'T001', 'CP101001', 2026, 1, 'MONDAY', TO_TIMESTAMP('09:00', 'HH24:MI'), TO_TIMESTAMP('12:00', 'HH24:MI'), 'CP.01');
INSERT INTO TeachAssignment (TAssignID, TID, SubjCode, Year, Semester, StudyDay, StartTime, EndTime, Room) 
VALUES ('TA002', 'T002', 'CP101002', 2026, 1, 'WEDNESDAY', TO_TIMESTAMP('13:00', 'HH24:MI'), TO_TIMESTAMP('16:00', 'HH24:MI'), 'CP.02');

INSERT INTO StudyRegister (StuRegisID, TAssignID, UserID, Section, StuRegisStatus) 
VALUES (10001, 'TA001', 67001, 1, 'ENROLLED');
INSERT INTO StudyRegister (StuRegisID, TAssignID, UserID, Section, StuRegisStatus) 
VALUES (10002, 'TA002', 67001, 1, 'ENROLLED');

INSERT INTO Register (RegID, UserID, EventID) 
VALUES ('R0001', 67001, 'E001');

INSERT INTO StdAssignment (AssignID, SubjCode, AssName, Dateline, Score) 
VALUES ('ASS00001', 'CP101001', 'Neural Network Lab', TO_DATE('2026-03-15', 'YYYY-MM-DD'), 100);

INSERT INTO AssignScore (ScoreAssignID, UserID, AssignID, Status, PriScore) 
VALUES (20001, 67001, 'ASS00001', 'SUBMITTED', 85);

INSERT INTO Grade (GradeCode, StuRegisID, GradeResult) 
VALUES (30001, 10001, 'B+');

-- ใช้ INSERT ReviewTeacher ชุดใหม่ที่แก้เป็น TAssignID แล้ว (มีแค่บรรทัดเดียว)
INSERT INTO ReviewTeacher (TeachReview, TAssignID, UserID, UserReviewT, UserRateT) 
VALUES (40001, 'TA001', 67001, 'Good material and clear explanation.', 5);

UPDATE AssignScore SET PriScore = 95 WHERE ScoreAssignID = 20001;
SELECT * FROM Notifications;

COMMIT;

-----------------------------------------------
--TEST QUERIES
SELECT * FROM v_student_gpa;

INSERT INTO StdAssignment (AssignID, SubjCode, AssName, Dateline, Score) 
VALUES ('ASS999', 'CP101002', 'SQL Project Design', TO_DATE('2026-04-01', 'YYYY-MM-DD'), 50);

SELECT * FROM Notifications ORDER BY NotiDate DESC;

SELECT * FROM v_student_dashboard;

SET SERVEROUTPUT ON;
EXEC sp_safe_event_register(67001, 'E001');
EXEC sp_get_student_alerts(67001);

--;

SELECT * FROM UserInfo;
