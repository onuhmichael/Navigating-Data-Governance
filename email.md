Excellent! Let's get started with building the script.

### **Step 1: Setting Up Your Project**

First, we need to organize your project files. Create a new folder on your computer and name it something like `email_automation`. Inside this folder, you will create two files:

1.  `.env`: This file will store your sensitive information, like passwords and server names.
2.  `email_processor.py`: This will be our main Python script.

You will also need a folder to store the attachments that are downloaded from the emails. Let's call it `attachments`.

Your folder structure should look like this:

```
email_automation/
|-- .env
|-- email_processor.py
|-- attachments/
```

### **Step 2: Installing Necessary Libraries**

Next, you'll need to install a few Python libraries that will help us interact with emails, data, and the database. Open your terminal or command prompt and run the following command:

```bash
pip install python-dotenv pandas openpyxl pyodbc
```

Here's what each library does:

* `python-dotenv`: To load the configuration variables from our `.env` file.
* `pandas`: To work with the data from the Excel or CSV file.
* `openpyxl`: To help `pandas` read Excel files.
* `pyodbc`: To connect to the SQL Server database.

### **Step 3: Configuring Your Environment Variables**

Now, let's set up the `.env` file. Open the `.env` file you created and add the following lines. Fill in the placeholders with your actual credentials.

```env
# Email Configuration
OUTLOOK_EMAIL=onuhmichael@mike.com
OUTLOOK_PASSWORD=your_email_password
IMAP_SERVER=outlook.office365.com

# Email Search Criteria
SENDER_EMAIL=alenyi@mike.com
EMAIL_SUBJECT=cinapsis_daily

# Database Configuration
DB_SERVER=your_db_server_name
DB_DATABASE=demo
DB_USERNAME=your_db_username
DB_PASSWORD=your_db_password
DB_TABLE=Inpatients
DB_SCHEMA=dbo

# Attachment Folder
ATTACHMENT_PATH=attachments
```

### **Step 4: The Python Script**

Here is the complete Python script. Copy and paste the following code into your `email_processor.py` file. I've included detailed comments on every line to explain what's happening.

```python
# Import necessary libraries
import os                 # To interact with the operating system (e.g., file paths)
import imaplib            # To connect to and interact with an email server using IMAP
import email              # To parse email messages
from email.header import decode_header # To decode email headers (like the subject)
import zipfile            # To work with zip files
import pandas as pd       # To work with data in a structured way (DataFrames)
import pyodbc             # To connect to a SQL Server database
from dotenv import load_dotenv # To load environment variables from the .env file
import logging            # To log information, warnings, and errors
from datetime import datetime # To work with dates and times

# --- 1. SCRIPT SETUP ---

# Load environment variables from the .env file
load_dotenv()
print("Loaded environment variables from .env file.")

# Configure logging to provide detailed information about the script's execution
logging.basicConfig(
    level=logging.INFO,  # Set the minimum level of messages to log
    format='%(asctime)s - %(levelname)s - %(message)s',  # Define the format of the log messages
    handlers=[
        logging.FileHandler("script_log.log"),  # Log messages to a file named 'script_log.log'
        logging.StreamHandler()  # Also print log messages to the console
    ]
)
logging.info("Logging has been configured.")

# --- 2. EMAIL PROCESSING FUNCTIONS ---

def get_email_connection():
    """Establishes a connection to the Outlook email server."""
    try:
        # Log in to the email server using credentials from the .env file
        logging.info("Attempting to connect to the email server...")
        mail = imaplib.IMAP4_SSL(os.getenv("IMAP_SERVER"))  # Create a secure connection
        mail.login(os.getenv("OUTLOOK_EMAIL"), os.getenv("OUTLOOK_PASSWORD"))  # Log in
        logging.info("Successfully connected to the email server.")
        return mail  # Return the connection object
    except Exception as e:
        # Log an error if the connection fails
        logging.error(f"Failed to connect to the email server: {e}")
        return None  # Return nothing if the connection fails

def search_and_download_attachment(mail):
    """Searches for a specific email and downloads its attachment."""
    try:
        # Select the inbox to search for emails
        mail.select("inbox")
        logging.info("Inbox selected.")

        # Define the search criteria based on sender and subject from the .env file
        search_criteria = f'(FROM "{os.getenv("SENDER_EMAIL")}" SUBJECT "{os.getenv("EMAIL_SUBJECT")}")'
        
        # Search for emails that match the criteria
        status, messages = mail.search(None, search_criteria)
        if status != "OK":
            logging.error("Failed to search for emails.")
            return None

        # Get a list of email IDs
        email_ids = messages[0].split()
        if not email_ids:
            logging.warning("No emails found matching the criteria.")
            return None

        logging.info(f"Found {len(email_ids)} matching email(s). Processing the latest one.")
        
        # Get the latest email
        latest_email_id = email_ids[-1]
        
        # Fetch the full email message
        status, msg_data = mail.fetch(latest_email_id, "(RFC822)")
        if status != "OK":
            logging.error(f"Failed to fetch email with ID {latest_email_id}.")
            return None

        # Parse the email content
        msg = email.message_from_bytes(msg_data[0][1])
        logging.info("Successfully parsed the email.")

        # Loop through the parts of the email to find the attachment
        for part in msg.walk():
            if part.get_content_maintype() == 'multipart':
                continue # Skip the main container part
            if part.get('Content-Disposition') is None:
                continue # Skip parts that are not attachments

            # Get the filename of the attachment
            filename = part.get_filename()
            if filename and filename.endswith('.zip'):
                logging.info(f"Found attachment: {filename}")
                
                # Define the path to save the attachment
                filepath = os.path.join(os.getenv("ATTACHMENT_PATH"), filename)
                
                # Save the attachment to the specified path
                with open(filepath, 'wb') as f:
                    f.write(part.get_payload(decode=True))
                logging.info(f"Attachment saved to {filepath}")
                return filepath  # Return the path to the saved attachment

        logging.warning("No .zip attachment found in the email.")
        return None # Return nothing if no zip attachment is found

    except Exception as e:
        # Log an error if anything goes wrong
        logging.error(f"An error occurred while searching for and downloading the attachment: {e}")
        return None

def unzip_and_read_data(zip_filepath):
    """Unzips a file and reads the CSV or Excel file inside into a DataFrame."""
    try:
        # Define the directory to extract the files to
        unzip_dir = os.getenv("ATTACHMENT_PATH")
        
        # Open and extract the zip file
        with zipfile.ZipFile(zip_filepath, 'r') as zip_ref:
            zip_ref.extractall(unzip_dir)
        logging.info(f"Successfully unzipped {zip_filepath}")

        # Find the extracted file (CSV or Excel)
        for filename in os.listdir(unzip_dir):
            if filename.endswith(".csv"):
                # If it's a CSV, read it into a DataFrame
                filepath = os.path.join(unzip_dir, filename)
                df = pd.read_csv(filepath)
                logging.info(f"Read data from CSV file: {filename}")
                return df
            elif filename.endswith((".xls", ".xlsx")):
                # If it's an Excel file, read it into a DataFrame
                filepath = os.path.join(unzip_dir, filename)
                df = pd.read_excel(filepath)
                logging.info(f"Read data from Excel file: {filename}")
                return df
        
        logging.warning("No CSV or Excel file found in the zip archive.")
        return None

    except Exception as e:
        # Log an error if the file can't be unzipped or read
        logging.error(f"Failed to unzip or read the data file: {e}")
        return None

# --- 3. DATABASE FUNCTIONS ---

def get_db_connection():
    """Establishes a connection to the SQL Server database."""
    try:
        # Create the database connection string
        conn_str = (
            f"DRIVER={{ODBC Driver 17 for SQL Server}};"
            f"SERVER={os.getenv('DB_SERVER')};"
            f"DATABASE={os.getenv('DB_DATABASE')};"
            f"UID={os.getenv('DB_USERNAME')};"
            f"PWD={os.getenv('DB_PASSWORD')};"
        )
        # Connect to the database
        conn = pyodbc.connect(conn_str)
        logging.info("Successfully connected to the SQL Server database.")
        return conn # Return the connection object
    except Exception as e:
        # Log an error if the connection fails
        logging.error(f"Failed to connect to the database: {e}")
        return None

def insert_data_to_db(conn, df):
    """Inserts new data from a DataFrame into the database."""
    try:
        # Create a cursor to execute SQL commands
        cursor = conn.cursor()
        logging.info("Database cursor created.")

        # Get the full table name from the .env file
        table_name = f"[{os.getenv('DB_SCHEMA')}].[{os.getenv('DB_TABLE')}]"
        
        # Fetch existing PatientIDs to check for duplicates
        cursor.execute(f"SELECT PatientID FROM {table_name}")
        existing_patient_ids = {row.PatientID for row in cursor.fetchall()}
        logging.info(f"Fetched {len(existing_patient_ids)} existing patient IDs.")

        # Keep track of how many new records are inserted
        new_records_count = 0
        
        # Loop through each row in the DataFrame
        for index, row in df.iterrows():
            # Check if the PatientID from the file already exists in the database
            if row['PatientID'] not in existing_patient_ids:
                # If the PatientID is new, insert the data
                try:
                    # The SQL query to insert a new record
                    insert_query = f"""
                        INSERT INTO {table_name} (
                            PatientID, FirstName, LastName, DateOfBirth, Gender, 
                            AdmissionDate, DischargeDate, WardNumber, BedNumber, 
                            Diagnosis, TreatmentPlan, DoctorID, EmergencyContactName, 
                            EmergencyContactPhone, InsuranceProvider, DateInserted
                        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                    """
                    # The data to be inserted
                    # We add the current date and time for the 'DateInserted' column
                    values_to_insert = (
                        row['PatientID'], row['FirstName'], row['LastName'], row['DateOfBirth'],
                        row['Gender'], row['AdmissionDate'], row.get('DischargeDate'), row['WardNumber'],
                        row['BedNumber'], row.get('Diagnosis'), row.get('TreatmentPlan'),
                        row.get('DoctorID'), row.get('EmergencyContactName'),
                        row.get('EmergencyContactPhone'), row.get('InsuranceProvider'),
                        datetime.now() # Add the current timestamp for the new column
                    )
                    
                    # Execute the insert command
                    cursor.execute(insert_query, values_to_insert)
                    new_records_count += 1 # Increment the counter for new records
                    logging.info(f"Inserted new record with PatientID: {row['PatientID']}")

                except Exception as e:
                    # Log an error if a specific row fails to insert
                    logging.error(f"Failed to insert row with PatientID {row['PatientID']}: {e}")

        # Commit the changes to the database to make them permanent
        conn.commit()
        logging.info(f"Successfully inserted {new_records_count} new records into the database.")
        
        # Close the cursor
        cursor.close()
        logging.info("Database cursor closed.")

    except Exception as e:
        # Log an error if anything goes wrong during the database insertion process
        logging.error(f"An error occurred during data insertion: {e}")

# --- 4. MAIN EXECUTION ---

if __name__ == "__main__":
    logging.info("--- Starting Email to Database Script ---")
    
    # Step 1: Connect to Outlook
    mail_conn = get_email_connection()
    if mail_conn:
        # Step 2: Search for the email and download the attachment
        attachment_path = search_and_download_attachment(mail_conn)
        if attachment_path:
            # Step 3: Unzip the file and read the data
            data_frame = unzip_and_read_data(attachment_path)
            if data_frame is not None:
                # Step 4: Connect to the SQL Server database
                db_conn = get_db_connection()
                if db_conn:
                    # Step 5: Insert the new data into the database
                    insert_data_to_db(db_conn, data_frame)
                    # Close the database connection
                    db_conn.close()
                    logging.info("Database connection closed.")
        # Log out from the email server
        mail_conn.logout()
        logging.info("Logged out from the email server.")

    logging.info("--- Script Finished ---")
```

### **Step 5: How to Run the Script**

1.  **Fill in the `.env` File**: Make sure you have entered your correct email and database credentials in the `.env` file.
2.  **Run from Terminal**: Open your terminal or command prompt, navigate to your `email_automation` folder, and run the script with the following command:

    ```bash
    python email_processor.py
    ```

3.  **Check the Logs**: As the script runs, you will see log messages in your terminal. You can also open the `script_log.log` file to see a complete record of the script's activity.

### Below is the exact content you need to copy and paste into your `.env` file. You will need to replace the placeholder values (like `your_email_password`, `your_db_server_name`, etc.) with your actual credentials and information.

### **Content for Your `.env` File**

Copy the following text directly into the `.env` file you created:

```env
# --- Email Configuration ---
# Your full Outlook email address
OUTLOOK_EMAIL=onuhmichael@mike.com

# The password for your Outlook email account.
# Note: If you have two-factor authentication (2FA) enabled, you might need to generate an "app password".
OUTLOOK_PASSWORD=your_email_password

# The IMAP server for Outlook. This is usually the correct address.
IMAP_SERVER=outlook.office365.com

# --- Email Search Criteria ---
# The email address of the sender you are looking for.
SENDER_EMAIL=alenyi@mike.com

# The subject of the email you are looking for.
EMAIL_SUBJECT=cinapsis_daily

# --- Database Configuration ---
# The name or IP address of your SQL Server instance.
# Example: "my-server.database.windows.net" or "192.168.1.100"
DB_SERVER=your_db_server_name

# The name of the database you are connecting to.
DB_DATABASE=demo

# The username for your database login.
DB_USERNAME=your_db_username

# The password for your database login.
DB_PASSWORD=your_db_password

# The name of the table where data will be inserted.
DB_TABLE=Inpatients

# The schema of the table. In SQL Server, this is often "dbo".
DB_SCHEMA=dbo

# --- File Path Configuration ---
# The name of the folder where attachments will be downloaded and unzipped.
# The script will look for this folder in the same directory where the script is run.
ATTACHMENT_PATH=attachments
```

### **Important Notes:**

* **No Quotes**: Do not use quotation marks (`"` or `'`) around the values in the `.env` file, even for passwords or server names.
* **App Passwords**: If your email account uses two-factor authentication (2FA), you will likely need to generate an "app password" from your email account security settings and use that for the `OUTLOOK_PASSWORD`. Standard passwords often do not work with scripts when 2FA is enabled.
* **Database Driver**: The Python script assumes you have the `ODBC Driver 17 for SQL Server` installed on the machine where you are running the script. If you are using a different driver, you will need to update the connection string in the `get_db_connection` function within the `email_processor.py` file.

Once you have filled in your specific details and saved the `.env` file, you'll be ready to run the Python script.
