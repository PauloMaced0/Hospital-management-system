# Hospital Management System Report Summary

## Introduction

The project focuses on automating and streamlining the billing and prescription processes in a hospital setting. 
Key components include patient management, appointment scheduling, billing, and prescription management.

### DER
![DER, Entity relation diagram](/Docs/ER.png)

## Key Features

- **Patient Management**: Handles patient registration, personal details, and insurance information.
- **Appointment Scheduling**: Facilitates scheduling and management of patient appointments.
- **Billing System**: Calculates fees, generates bills, and updates payment status.
- **Prescription Management**: Manages and records patient prescriptions.

<u> Just some of this features will be presented in the next section <u>

## Procedures

  - **AddPatient**:
  ```sql
  CREATE PROCEDURE AddPatient
    @CitizenCardNumber VARCHAR(255),
    @Name VARCHAR(255),
    @Gender CHAR(1),
    @DateOfBirth DATE,
    @ContactInformation VARCHAR(255),
    @Address TEXT,
    @Insurance BIT
  AS
  BEGIN
    INSERT INTO Patient (CitizenCardNumber, Name, Gender, DateOfBirth, ContactInformation, Address, Insurance)
    VALUES (@CitizenCardNumber, @Name, @Gender, @DateOfBirth, @ContactInformation, @Address, @Insurance);
  END
  GO
  ```

  - **GetPatientsByIllness**:
  ```sql
  CREATE PROCEDURE GetPatientsByIllness
  @IllnessName VARCHAR (255)
  AS
  BEGIN
      SELECT
          p.CitizenCardNumber,
          p.Name,
          p.Gender,
          dbo.GetAge(p.DateOfBirth) AS Age,
          p.ContactInformation,
          p.Address,
          p.Insurance
      FROM Patient p
      JOIN Illness i ON p.CitizenCardNumber = i.PatientID
      WHERE i.IllnessName = @IllnessName
      ORDER BY p.Name;
  END
  GO
  ```

## Triggers

  - **LogPatientUpdates**

  ```sql
  CREATE TRIGGER LogPatientUpdates
  ON Patient
  AFTER UPDATE
  AS
  BEGIN
      IF UPDATE (ContactInformation)
      BEGIN
          INSERT INTO PatientUpdateLog (PatientID, OldContact, NewContact, UpdateDate)
          SELECT i.CitizenCardNumber, d.ContactInformation, i.ContactInformation, GETDATE()
          FROM inserted i
          JOIN deleted d ON i.CitizenCardNumber = d.CitizenCardNumber;
      END
  END;
  GO
  ```

  - **AutoGenerateBill**

  ```sql
  CREATE TRIGGER AutoGenerateBill
  ON Appointment
  AFTER INSERT
  AS
  BEGIN
      DECLARE @Fee DECIMAL(10, 2);
      DECLARE @Specialization INT;
      SELECT @Specialization = d.Specialization
      FROM Doctor d
      JOIN inserted i ON i.DoctorID = d.CitizenCardNumber;
      SELECT @Fee = dbo.CalculateConsultationFee(i.PatientID, @Specialization)
      FROM inserted i;
      INSERT INTO Bill(PatientID, DoctorID, AppointmentID, PaymentStatus, BillDate, TotalAmount)
      SELECT PatientID, DoctorID, AppointmentID, 'Pending', GETDATE(), @Fee
      FROM inserted;
  END;
  GO
  ```

## Functions

  - **GetAge**

  ```sql
  CREATE FUNCTION GetAge (@dob DATE)
  RETURNS INT
  AS
  BEGIN
      DECLARE @age INT;
      SET @age = DATEDIFF(year, @dob, GETDATE());
      IF (MONTH(@dob) > MONTH(GETDATE())) OR (MONTH(@dob) = MONTH(GETDATE()) AND DAY(@dob) > DAY(GETDATE()))
          SET @age = @age - 1;
      RETURN @age;
  END;
  GO
  ```

