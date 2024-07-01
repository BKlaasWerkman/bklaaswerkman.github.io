---
title: Get-Lastpass.py 
description: Python script to log into LastPass CLI and query which accounts based on entered Account Name keyword have weak passwords and export the data with columns containing Account Name and Username to csv & excel files. Exported files will be in current working directory of script. Excel file will open automatically for you.
author: BW
date: 2024-01-19 10:00:00 - 0500
categories: [Scripting, Python]
tags: [scripting]     # TAG names should always be lowercase
---

### Get-Lastpass.py


```python

'''

DISCRIPTION

    Python script to log into LastPass CLI and query which accounts based on entered Account Name keyword have weak passwords and export the data with columns containing Account Name and Username to csv & excel files.
    Exported files will be in current working directory of script.
    Excel file will open automatically for you.

NOTES

    Name: Get-lastpass
    Author: B. Werkman
    Version: 1.0
    DateCreated: Jan 2024

***Passwords will not be exported***

'''

import re
import csv
import getpass
import os
import sys, subprocess
import webbrowser

try:
    from openpyxl import Workbook
    from openpyxl.styles import Font, Alignment, PatternFill, Border, Side

except ImportError:
    subprocess.check_call([sys.executable, "-m", "pip", "install", "openpyxl"])
    from openpyxl import Workbook
    from openpyxl.styles import Font, Alignment, PatternFill, Border, Side

try:
    import lastpass

except ImportError:
    subprocess.check_call([sys.executable, "-m", "pip", "install", "lastpass-python"])
    import lastpass

def is_complex(password):
    if len(password) < 16:
        return False
    has_upper = re.search("[A-Z]", password) is not None
    has_lower = re.search("[a-z]", password) is not None
    has_digit = re.search("[0-9]", password) is not None
    has_special = re.search("[!@#$%^&*(),.<>?/|]", password) is not None
    return has_upper and has_lower and has_digit and has_special

def get_lastpass_accounts(username, password, keyword):
    try:
        print("Logging in to LastPass...Check for Authenticator prompt...")
        vault = lastpass.Vault.open_remote(username, password)
        print("Login successful. Processing accounts...")
    except Exception as e:
        print("Error logging into LastPass:", str(e))
        return []
    filtered_accounts = []
    for i in vault.accounts:
        account_name = i.name.decode('utf-8')
        account_username = i.username.decode('utf-8')  # Decode the username
        if keyword.lower() in account_name.lower() and not is_complex(i.password.decode('utf-8')):
            filtered_accounts.append((account_name, account_username))
    return filtered_accounts

# Prompt for user credentials
username = input("Enter your LastPass username: ")
password = getpass.getpass("Enter your LastPass password: ")
keyword = input("Enter the account name keyword to search for: ")

# Fetch accounts
accounts = get_lastpass_accounts(username, password, keyword)

# Get the directory where the script is located
script_dir = os.path.dirname(os.path.realpath(__file__))

# Define the file path
csv_filename = os.path.join(script_dir, 'non_complex_passwords.csv')

# Export to CSV
with open(csv_filename, mode='w', newline='', encoding='utf-8') as file:
    writer = csv.writer(file)
    writer.writerow(['Account Name', 'Username'])
    for account_name, account_username in accounts:
        writer.writerow([account_name, account_username])

print(f"Accounts with Non-Complex Passwords exported to {csv_filename}")

# Create a new Excel workbook and select the active worksheet
wb = Workbook()
ws = wb.active

# Applying styles
header_font = Font(bold=True, color="FFFFFF")
header_fill = PatternFill(start_color="4F81BD", end_color="4F81BD", fill_type="solid")
header_alignment = Alignment(horizontal='center')
thin_border = Border(left=Side(style='thin'),
                     right=Side(style='thin'),
                     top=Side(style='thin'),
                     bottom=Side(style='thin'))

# Add headers with formatting
headers = ['Account Name', 'Username']
ws.append(headers)
for col in ws.iter_cols(min_col=1, max_col=len(headers), max_row=1):
    for cell in col:
        cell.font = header_font
        cell.alignment = header_alignment
        cell.fill = header_fill
        cell.border = thin_border

# Write data to Excel
data_font = Font(name='Calibri', size=11)
for account_name, account_username in accounts:
    row = [account_name, account_username]
    ws.append(row)
    for cell in ws[ws.max_row]:
        cell.font = data_font
        cell.border = thin_border

# Auto-adjust column widths
for column in ws.columns:
    max_length = max(len(str(cell.value)) for cell in column)
    adjusted_width = (max_length + 2)
    ws.column_dimensions[column[0].column_letter].width = adjusted_width

# Define the file path
excel_filename = os.path.join(script_dir, 'non_complex_passwords.xlsx')

# Save the workbook
wb.save(excel_filename)
print(f"Accounts with Non-Complex Passwords also exported to {excel_filename}")

# Open the Excel file
excel_file_path = os.path.abspath(excel_filename)
webbrowser.open(f'file://{excel_file_path}')

```
{: file='Get-Lastpass.py'}
