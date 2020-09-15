```CSHARP
public class EmailService : MailKit.SmtpClient {
```
This class will deliver emails and locally keep track of which recipients need retries. After a number of retries have been made, the current state of attempts will be committed to the database.

This class relies on:
- a `DatabaseConnector` with the ability to insert `emails` and `recipients` in a Task-based manner.
- a third-party SMTP library called `Mailkit`. This allows us a Task-based implementation to send emails.

We assume that:
- there is a SMTP mail server available to be used
- there is an available MySQL server available with the tables described in `schema.dbml`.

In the `SendMessage` method defined later in this class, the steps involving sending the email and inserting into the database can be `Await`ed on, suspending `SendMessage` and returning control to the caller while longer run operatings continue.

Given that the goal is to be able to perform these operations without requiring the user to wait for a result, `SendMessage` is defined as async.

Mailkit's SMTP client can connecto to a SMTP sever to send the emails to. Mailkit also includes a class implementation of an email message called `MimeMessage`, which is used by the SMTP client to send the email to recipients.

```CSHARP
static Dictionary<string, (int, bool)> RecipientsToAttempts;
```
Given a list of recipients to deliver to, map the recipients from 
email to a new DeliveryStatus class. Each DeliveryStatus class contains the number of attempts so far, a whether the last attempt failed.

```CSHARP
public static async void SendMessage(
    List<string> recipientMailboxes,
    string senderMailbox,
    string body,
    string subject,
    int retries,
    DatabaseConnector databaseConnector
)
```
This method will attempt one delivery first, tracking which recipients failed to accept the delivery. Those that failed will be re-attempted.

1. Given a list of recipients to deliver to, map the recipients from 
email to a tuple initialized with `(1, false)`, representing our first attempt at delivery, and whether or not the delivery failed. Use `RecipientsToAttempts`.

2. Try to send the message to all of the recipients in the map. When a delivery fails, update the map of the recipient to increase the attempts and mark it as failed.

3. One attempt has been made, try again `retries-1` times to send the message to recipients in the map where their DeliveryStatus attempt is still marked as failed, and the number of attempts are less than two.

4. After `retries-1` attempts, commit the map to the database using the provided `DatabaseConnector`.
    - Insert the email into the `emails` table, and the recipients into the `recipients` table with the `send_date` of today.

5. Clear the `RecipientsToAttempts` map.

```CSHARP
protected override void OnRecipientNotAccepted(
	MimeMessage message,
	MailboxAddress mailbox,
	SmtpResponse response
)
```
When a recipient is not accepted by the SMTP server, this method will update recipientsToAttempts, increasing the number of attempts so far for the recipient, and setting the bool indicating failure to true.

The recipient in this case is stored in the `mailbox` parameter.