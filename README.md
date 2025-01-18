# stockman-extract
!apt-get update
!apt-get install -y poppler-utils
!apt-get install -y libpoppler-cpp-dev
!pip install pdf2image
!pip install pytesseract
!sudo apt install tesseract-ocr
!pip install ace-tools
import os
os.environ['PATH'] += os.pathsep + '/usr/bin'
import pytesseract
from PIL import Image
import pandas as pd
from pdf2image import convert_from_path
from google.colab import files
import re

# Ensure the correct path to Tesseract is set
pytesseract.pytesseract.tesseract_cmd = '/usr/bin/tesseract'

# Upload the PDF file
print("Please upload your PDF file:")
uploaded = files.upload()
pdf_path = list(uploaded.keys())[0]

# Convert the PDF into a list of images
images = convert_from_path(pdf_path, poppler_path='/usr/bin')

# Initialize dictionaries to store data for each section
sections_data = {
    "Account Summary": [],
    "Deposits": [],
    "Other Credits": [],
    "Other Debits": [],
    "Daily Balances": []
}

# Process each page of the PDF
current_section = None

for page_number, image in enumerate(images, start=1):
    print(f"Processing page {page_number}...")
    image = image.convert("L")
    ocr_result = pytesseract.image_to_string(image, config="--psm 6")
    lines = ocr_result.split("\n")

    for line in lines:
        line = line.strip()
        if not line:
            continue

        # Detect sections based on keywords
        if "Account Summary" in line:
            current_section = "Account Summary"
            continue
        elif "Deposits" in line:
            current_section = "Deposits"
            continue
        elif "Other Credits" in line:
            current_section = "Other Credits"
            continue
        elif "Other Debits" in line:
            current_section = "Other Debits"
            continue
        elif "Daily Balances" in line:
            current_section = "Daily Balances"
            continue

        # Match rows with Date, Description, and Amount
        if current_section in ["Deposits", "Other Credits", "Other Debits"]:
            match = re.match(r"^(\d{2}/\d{2}/\d{4})\s+(.+?)\s+\$([\d,\.]+)$", line)
            if match:
                date, description, amount = match.groups()
                sections_data[current_section].append([date, description, amount])
            else:
                if len(sections_data[current_section]) > 0:
                    sections_data[current_section][-1][1] += f" {line}"

        # Handle Account Summary rows
        elif current_section == "Account Summary":
            match = re.match(r"^(.+?)\s+\$([\d,\.]+)$", line)
            if match:
                description, amount = match.groups()
                date_match = re.search(r"\d{2}/\d{2}/\d{4}", description)
                date = date_match.group() if date_match else None
                description = description.replace(date, "").strip() if date else description
                sections_data[current_section].append([date, description, amount])
            else:
                if len(sections_data[current_section]) > 0:
                    sections_data[current_section][-1][1] += f" {line}"

        # Handle Daily Balances rows
        elif current_section == "Daily Balances":
            match = re.findall(r"(\d{2}/\d{2}/\d{4})\s+\$([\d,\.]+)", line)
            if match:
                for date, amount in match:
                    sections_data[current_section].append([date, amount])

# Convert data for each section into separate DataFrames
account_summary_df = pd.DataFrame(sections_data["Account Summary"], columns=["Date", "Description", "Amount"])
deposits_df = pd.DataFrame(sections_data["Deposits"], columns=["Date", "Description", "Amount"])
other_credits_df = pd.DataFrame(sections_data["Other Credits"], columns=["Date", "Description", "Amount"])
other_debits_df = pd.DataFrame(sections_data["Other Debits"], columns=["Date", "Description", "Amount"])
daily_balances_df = pd.DataFrame(sections_data["Daily Balances"], columns=["Date", "Amount"])

# Format and clean data
account_summary_df["Date"] = account_summary_df["Date"].fillna("")
account_summary_df["Amount"] = account_summary_df["Amount"].apply(lambda x: f"${x}")
deposits_df["Description"] = deposits_df["Description"].str.replace("=", "").str.strip()
deposits_df["Amount"] = deposits_df["Amount"].apply(lambda x: f"${x}")
other_credits_df["Amount"] = other_credits_df["Amount"].apply(lambda x: f"${x}")
other_debits_df["Amount"] = other_debits_df["Amount"].apply(lambda x: f"${x}")
daily_balances_df["Amount"] = daily_balances_df["Amount"].apply(lambda x: f"${x}")
daily_balances_df = daily_balances_df.sort_values(by="Date").reset_index(drop=True)

# Save individual Other Credits and Other Debits tables
other_credits_file = "other_credits.csv"
other_debits_file = "other_debits.csv"
other_credits_df.to_csv(other_credits_file, index=False)
other_debits_df.to_csv(other_debits_file, index=False)

# Merge Other Credits and Other Debits tables
other_credits_df['Type'] = 'Credit'
other_debits_df['Type'] = 'Debit'
merged_other_transactions_df = pd.concat([other_credits_df, other_debits_df])
merged_other_transactions_df['Date'] = pd.to_datetime(merged_other_transactions_df['Date'])
merged_other_transactions_df = merged_other_transactions_df.sort_values(by=['Date']).reset_index(drop=True)

# Save the merged table as a separate CSV file
merged_other_transactions_file = "merged_other_transactions.csv"
merged_other_transactions_df.to_csv(merged_other_transactions_file, index=False)

# Save other tables
account_summary_file = "account_summary.csv"
deposits_file = "deposits.csv"
daily_balances_file = "daily_balances.csv"
account_summary_df.to_csv(account_summary_file, index=False)
deposits_df.to_csv(deposits_file, index=False)
daily_balances_df.to_csv(daily_balances_file, index=False)

# Display all tables
print("Account Summary:")
display(account_summary_df)
print("\nDeposits:")
display(deposits_df)
print("\nOther Credits:")
display(other_credits_df)
print("\nOther Debits:")
display(other_debits_df)
print("\nDaily Balances:")
display(daily_balances_df)
print("\nMerged Other Transactions:")
display(merged_other_transactions_df)

# Print file save messages
print(f"Account Summary saved to {account_summary_file}")
print(f"Deposits saved to {deposits_file}")
print(f"Other Credits saved to {other_credits_file}")
print(f"Other Debits saved to {other_debits_file}")
print(f"Daily Balances saved to {daily_balances_file}")
print(f"Merged Other Transactions saved to {merged_other_transactions_file}")
