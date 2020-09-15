The following tables are described in regular mariaDB SQL. Originally the plan was to have a many to many relationsihp from `emails` to `recipients` with the in-between table also holding the status of wheter or not the email was delivered to this recipient.

However since saving the status of the email long-term is not a requirement, and neither are the number of attempts, it is suffice to use two tables with a one to many relationship between `emails` and `recipients`.

```SQL
CREATE TABLE emails (
    email_id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    body TEXT,
    email_subject varchar(256),
    email_date DATETIME NOT NULL DEFAULT NOW(),
    sender varchar(256) NOT NULL
);

CREATE TABLE recipients (
    recipient_id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    email_id INT NOT NULL,
    CONSTRAINT `fk_recipient_email`
        FOREIGN KEY (email_id) REFERENCES emails (email_id)
        ON DELETE CASCADE
        ON UPDATE RESTRICT,
    email_address varchar(256) NOT NULL
);
```

This will require a few stored procedures to insert into these tables and save the state of an email.
- inserting into `emails` table, called `InsertEmail`
- inserting into `recipients` table, called `InsertRecipient`

Of course more will be needed in order to retrieve data, however at a minimum, these are necessary. One consideration to keep in mind is that because the `recipients` table is seperate from the `email` table, inserting a large amount of recipients could be done either one a time, or in larger bulk inserts by programatically building an `INSERT` statement.