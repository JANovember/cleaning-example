``` python

import all libraries
import csv
import pandas as pd
import numpy as np
import regex as re
import difflib
import logging
# Configure logging
def setup_logger():
    logger = logging.getLogger("Data Cleaner")  # Create a logger object
    if not logger.hasHandlers():  # Avoid adding handlers multiple times
        logger.setLevel(logging.DEBUG)  # Set the minimum logging level

        # Create handlers
        file_handler = logging.FileHandler("breadcrumbs.log")  # Logs to a file
        console_handler = logging.StreamHandler()  # Logs to the VSCode terminal

        # Set logging level for handlers
        file_handler.setLevel(logging.DEBUG)
        console_handler.setLevel(logging.INFO)

        # Create formatters
        formatter = logging.Formatter(
            "%(asctime)s - %(name)s - %(levelname)s - %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S",
        )
        file_handler.setFormatter(formatter)
        console_handler.setFormatter(formatter)

        # Add handlers to the logger
        logger.addHandler(file_handler)
        logger.addHandler(console_handler)

    return logger

# Initialize the logger
logger = setup_logger()

# Log a message
logger.info("Logging configured successfully")
Define all functions 
def read_csv(file_path, sep=';', header=0, engine=None):
    try:
        df = pd.read_csv(file_path, sep=sep, header=header, engine=engine)
        logging.info('file read, dataframe created')
        return df
    except Exception as e:
        print(f'Error reading csv: {e}')
        return None
    
def remove_duplicates(df):
    try:
        initial_count = len(df)
        df = df.drop_duplicates()
        final_count = len(df)
        print(f'Removed {initial_count - final_count} duplicate rows.')
        return df
    except Exception as e:
        print(f'Error removing duplicates: {e}')
        return df

def standardize_column_headers(df):
    df.columns = df.columns.str.lower().str.replace(' ', '').str.replace('_', '')
    return df

def filter_dataframe(df, columns_to_keep):
    """
    Filters a DataFrame to retain only specified columns, with error handling and logging.

    Parameters:
        df (pd.DataFrame): The input DataFrame.
        columns_to_keep (list): List of column names to retain.

    Returns:
        pd.DataFrame: A filtered DataFrame with only the specified columns.
    """
    try:
        # Validate columns to keep
        valid_columns = [col for col in columns_to_keep if col in df.columns]
        
        if not valid_columns:
            logging.warning("None of the specified columns are in the DataFrame. Returning an empty DataFrame.")
            return pd.DataFrame()  # Return an empty DataFrame if no valid columns

        # Log any missing columns
        missing_columns = set(columns_to_keep) - set(valid_columns)
        if missing_columns:
            logging.warning(f"The following columns were not found in the DataFrame: {missing_columns}")

        # Keep only the valid columns
        filtered_df = df[valid_columns]
        logging.info(f"Successfully filtered DataFrame. Retained columns: {valid_columns}")
        return filtered_df

    except Exception as e:
        logging.error(f"An error occurred while filtering the DataFrame: {e}")
        raise  # Re-raise the exception after logging

def clean_and_standardize_names(df, column_name):
    try:

        def retain_language_characters(text):
            """
            Retains all language-specific characters and removes everything else.
            This includes Unicode letters from all scripts and spaces.
            """
            if isinstance(text, str):
                # Use regex to match Unicode letters (all scripts) and spaces
                language_pattern = re.compile(r'[^\p{L}\s]', flags=re.UNICODE)
                return language_pattern.sub('', text).strip()
            return text

        df[column_name] = df[column_name].apply(retain_language_characters)
        df[column_name] = df[column_name].str.replace(r'\d+', '', regex=True).str.strip()

        def is_invalid_name(name: str) -> bool:
            if pd.isna(name) or not isinstance(name, str): return True
            if any(char.isdigit() for char in name) or any(char in set('~!@#$%^&*()_+=[]}{|\\:;,.<>?') for char in name):
                return True
            return False

        invalid_names = df[df[column_name].apply(is_invalid_name)]
        
        def clean_name(name):
            return '' if is_invalid_name(name) else name.title().strip()

        df[column_name] = df[column_name].apply(clean_name)
        return df
    except Exception as e:
        print(f'Error standardizing names: {e}')
        return df

def validate_emails(df, column):
    try:
        email_pattern = r'^[a-zA-Z0-9._%+-]+@([^\d@]+\.[a-zA-Z]{2,})$'
        valid_domains = ['gmail.com', 'hotmail.com', 'yahoo.com', 'outlook.com', 'aol.com', 'icloud.com','naver.com','naver.net','hanmail.net']
        
        df[column] = df[column].str.replace(' ', '', regex=False)

        def clean_and_match_email(email):
            if isinstance(email, str) and '@' in email:
                username, domain = email.split('@', 1)
                domain = re.sub(r'[^a-zA-Z0-9.-]', '', domain).strip().replace(' ', '').lower()
                corrected_domain = difflib.get_close_matches(domain, valid_domains, n=1, cutoff=0.8)
                if corrected_domain:
                    domain = corrected_domain[0]
                return username + '@' + domain
            return email
        
        df[column] = df[column].apply(clean_and_match_email)
        invalid_emails = df[~df[column].str.match(email_pattern, na=False)]

        # Replace invalid emails with an empty string
        df.loc[~df[column].str.match(email_pattern, na=False), column] = ""
        print(f'Found {len(invalid_emails)} invalid email addresses.')
        return df
    except Exception as e:
        print(f'Error validating emails: {e}')
        return df
    
def validate_phone_numbers(df, column):
    try:
        df[column] = df[column].astype(str).str.replace(r'\D', '', regex=True)
        
        def is_invalid_number(number: str) -> bool:
            if not (6 <= len(number) <= 16) or bool(re.search(r'[^0-9+]', number)):
                return True
            return False

        invalid_numbers = df[df[column].apply(is_invalid_number)]
        df.loc[df[column].apply(is_invalid_number), column] = ''

        print(f'Found {len(invalid_numbers)} invalid phone numbers.')
        return df
    except Exception as e:
        print(f'Error validating phone numbers: {e}')
        return df
        
def can_not_be_empty(df):
    # Specified columns to check for missing values
    columns_to_check = ["givenname", "surname"]
    
    # Replace empty strings ("") with np.nan in the specified columns
    df[columns_to_check] = df[columns_to_check].replace("", np.nan)
    
    # Create a mask where rows with NaN in both 'givenname' and 'surname' are considered trash (True)
    return df[columns_to_check].isna().all(axis=1)

def has_three_consecutive_nulls_or_four_commas(row):
    # Convert row to a string, then check for three consecutive commas
    row_str = row.astype(str).str.cat(sep=',')
    if ',,,,' in row_str:
        return True

    # Check for three consecutive NaNs in the row
    nulls_as_list = row.isnull().astype(int).tolist()
    return [1, 1, 1] in [nulls_as_list[i:i+3] for i in range(len(nulls_as_list) - 2)]

def validate_rows(df):
    # Step 1: Create a mask for rows that should be dropped due to missing columns
    empty_mask = can_not_be_empty(df)

    # Step 2: Apply additional check for three consecutive nulls or four commas in any row
    consecutive_nulls_or_commas_mask = df.apply(has_three_consecutive_nulls_or_four_commas, axis=1)
    
    # Step 3: Combine the two masks to identify all rows to be dropped
    rows_to_drop_mask = empty_mask | consecutive_nulls_or_commas_mask  # Rows that meet either condition
    
    # Step 4: Filter the DataFrame into clean and trash data
    clean_data = df[~rows_to_drop_mask]  # Keep rows that are not marked for dropping
    trash_data = df[rows_to_drop_mask]   # Keep rows that are marked for dropping
    
    # Step 5: Save the trash data to a file
    trash_data.to_csv("6.garbage_rows.csv", index=False)
    
    return clean_data


    

Run all required functions 
df = read_csv(r'', sep=',', header=0, engine=None)

df = standardize_column_headers(df)

df = filter_dataframe(df, ['givenname', 'surname','mail', 'mobilephone', 'businessphones'])
df = remove_duplicates(df)
df.to_csv('1.filtered&duplicatesremoved.csv' , index=False)
df = clean_and_standardize_names(df, 'givenname')
df = clean_and_standardize_names(df, 'surname')
df = validate_emails(df, 'mail')
df.to_csv('2.emailsvalidated.csv', index=False)
df = validate_phone_numbers(df, 'mobilephone')   
df.to_csv('3.phonesvalidated.csv', index=False) 
df = validate_phone_numbers(df, 'businessphones')   
df.to_csv('4.businessphonesvalidated.csv', index=False) 
df = validate_rows(df)
df.to_csv('5.rowsvalidated.csv', index=False)
```
