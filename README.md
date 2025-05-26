# PostSwigger-Blind-SQL-injection-with-time-delays-and-information-retrieval

# Exploit for Lab: Blind SQL injection with time delays and information retrieval

This repository contains a Python exploit script for PortSwigger's Web Security Academy lab: **Blind SQL injection with time delays and information retrieval**.

Lab URL: [https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval)

## Lab Objective 🎯

The objective of this lab is to exploit a blind SQL injection vulnerability to retrieve the password for the `administrator` user from the `users` table. After obtaining the password, you need to log in as the administrator to solve the lab.

## Vulnerability Overview 📝

The application uses a tracking cookie (`TrackingId`) for analytics. The value of this cookie is incorporated into a SQL query in an unsafe way, making it vulnerable to SQL injection.

The key characteristics of this vulnerability are:
* **Blind SQL Injection**: The results of the SQL query are not directly returned in the application's responses.
* **No Differential Responses**: The application does not behave differently based on whether a query returns data or causes an error.
* **Time-Based Detection**: The SQL queries are processed synchronously. This allows us to introduce conditional time delays (e.g., using `pg_sleep()` in PostgreSQL) and observe the server's response time to infer information. If a condition is true, a delay is triggered; otherwise, no delay occurs.

The database contains a `users` table with `username` and `password` columns.

## Exploitation Strategy 🛠️

The exploit script automates the process of extracting the administrator's password using time-based blind SQL injection via stacked queries.

1.  **Confirm Vulnerability (Manual Step - Prerequisite)**: Before using the script, it's good practice to manually confirm that time delays can be triggered. The successful payload structure for this lab involves a **stacked query**:
    * The `TrackingId` cookie is manipulated to terminate the original SQL statement and inject a new `SELECT` statement.
    * This new `SELECT` statement uses a `CASE` expression to conditionally call `pg_sleep()` if a specific condition (related to the password) is true.
    * Example of a successful payload structure: `TrackingId=YOUR_ORIGINAL_TRACKING_ID'%3BSELECT+CASE+WHEN+(YOUR_CONDITION)+THEN+pg_sleep(X)+ELSE+pg_sleep(0)+END--`
    *(The script uses this stacked query approach).*

2.  **Determine Password Length**: The script first determines the length of the `administrator`'s password by injecting conditions like `(SELECT LENGTH(password) FROM users WHERE username='administrator')=N`. It iterates through possible lengths (`N`) and checks for a time delay.

3.  **Extract Password Characters**: Once the length is known, the script iterates through each character position of the password. For each position, it tests characters from a predefined charset (`abcdefghijklmnopqrstuvwxyz0123456789`) by injecting conditions like `(SELECT SUBSTRING(password, P, 1) FROM users WHERE username='administrator')='C'`, where `P` is the position and `C` is the character being tested. A time delay indicates the correct character for that position.

    ![Conceptual diagram of the exploit process](1.jpg)
    *(This image `1.jpg` should illustrate the iterative character guessing process)*

## Python Script Overview 🐍

The `exploit.py` script automates the password extraction process.

### Features:
* **Session Management**: Uses the `requests` library to manage HTTP sessions and cookies.
* **Time-Based Condition Check**: The `check_condition_time_based` function crafts and sends the malicious request. It measures the server's response time to determine if the injected SQL condition was true (i.e., if a delay occurred).
* **Automated Length Detection**: Systematically finds the password length.
* **Automated Character Brute-Force**: Tests characters one by one for each position in the password.
* **Configurable Parameters**:
    * `USERNAME`: Target username (default: "administrator").
    * `MAX_PASSWORD_LENGTH`: Maximum length to check for the password.
    * `CHARSET`: Character set to use for guessing the password.
    * `DELAY_SECONDS`: The duration of the SQL sleep command.
    * `DELAY_THRESHOLD`: The time threshold to decide if a delay occurred.
    * `REQUEST_TIMEOUT`: Timeout for HTTP requests.

### How to Use:
1.  **Prerequisites**:
    * Python 3 installed.
    * `requests` library installed (`pip install requests`).
2.  **Clone the repository or save the script.**
3.  **Run the script from your terminal**:
    ```bash
    python exploit.py
    ```
4.  **Enter the Lab URL** when prompted (e.g., `https://YOUR-LAB-ID.web-security-academy.net/`).
5.  The script will then attempt to find the password length and subsequently each character of the administrator's password.
6.  Once the password is found, use it to log in as `administrator` on the lab website to complete the challenge.

### Script Logic for `check_condition_time_based`:
The core of the exploit lies in how the `TrackingId` cookie is constructed:
1.  The original `TrackingId` value is retrieved from the initial session.
2.  An apostrophe (`'`) is appended to terminate the SQL string literal.
3.  A URL-encoded semicolon (`%3B` or `urllib.parse.quote_plus(";")`) is appended to start a new SQL statement (stacked query).
4.  The actual SQL payload for the time delay is constructed:
    ```sql
    SELECT CASE WHEN (YOUR_CONDITION_HERE) THEN pg_sleep(DELAY_SECONDS) ELSE pg_sleep(0) END -- 
    ```
    This part is then URL-encoded (e.g., spaces become `+` or `%20`).
5.  The script measures the time taken for the HTTP GET request with this modified cookie. If the time exceeds `DELAY_THRESHOLD`, the condition is considered true.

## Disclaimer ⚠️
This script is intended for educational purposes and for use on the PortSwigger Web Security Academy labs only. Do not use it on any other systems without explicit permission.
