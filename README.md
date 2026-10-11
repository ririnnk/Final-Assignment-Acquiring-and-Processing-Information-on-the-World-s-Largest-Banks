import requests
import sqlite3
import pandas as pd
import numpy as np
from bs4 import BeautifulSoup
from datetime import datetime

# ---- Configuration ----
DATA_URL = "https://web.archive.org/web/20230908091635/https://en.wikipedia.org/wiki/List_of_largest_banks"
EXCHANGE_RATE_CSV = "https://cf-courses-data.s3.us.cloud-object-storage.appdomain.cloud/IBMSkillsNetwork-PY0221EN-Coursera/labs/v2/exchange_rate.csv"
TABLE_ATTRIBS = ["Name", "MC_USD_Billion"]
OUTPUT_CSV_PATH = "./Largest_banks_data.csv"
DB_NAME = "Banks.db"
TABLE_NAME = "Largest_banks"
LOG_FILE = "code_log.txt"


# 1.1 Logging Function
def log_progress(message):
    """Logs progress messages with a timestamp to the log file."""
    timestamp_format = "%Y-%h-%d-%H:%M:%S"
    now = datetime.now().strftime(timestamp_format)
    with open(LOG_FILE, "a") as f:
        f.write(f"{now} : {message}\n")


# 1.2 Extract Function
def extract(url, table_attribs):
    """Extracts bank data from the Wikipedia webpage into a DataFrame."""
    html_page = requests.get(url).text
    data = BeautifulSoup(html_page, "html.parser")
    df = pd.DataFrame(columns=table_attribs)

    tables = data.find_all("tbody")
    rows = tables[0].find_all("tr")

    for row in rows:
        col = row.find_all("td")
        if len(col) != 0:
            links = col[1].find_all("a")
            if len(links) > 1:
                name = links[1]["title"]
            else:
                name = col[1].get_text(strip=True)

            # Clean Market Cap string (remove commas and newlines)
            mc = col[2].get_text(strip=True).replace(",", "").replace("\n", "")

            df = pd.concat(
                [df, pd.DataFrame({"Name": [name], "MC_USD_Billion": [float(mc)]})],
                ignore_index=True,
            )

    df["MC_USD_Billion"] = df["MC_USD_Billion"].astype(float)
    return df


# 1.3 Transform Function
def transform(df, csv_path):
    """Adds converted currency columns (GBP, EUR, INR) based on exchange rates."""
    exchange_df = pd.read_csv(csv_path)
    exchange_rate = exchange_df.set_index("Currency").to_dict()["Rate"]

    df["MC_GBP_Billion"] = [np.round(x * exchange_rate["GBP"], 2) for x in df["MC_USD_Billion"]]
    df["MC_EUR_Billion"] = [np.round(x * exchange_rate["EUR"], 2) for x in df["MC_USD_Billion"]]
    df["MC_INR_Billion"] = [np.round(x * exchange_rate["INR"], 2) for x in df["MC_USD_Billion"]]

    return df


# 1.4 Load to CSV Function
def load_to_csv(df, output_path):
    """Saves the DataFrame to a CSV file."""
    df.to_csv(output_path, index=False)


# 1.5 Load to Database Function
def load_to_db(df, sql_connection, table_name):
    """Loads the DataFrame into an SQLite database table."""
    df.to_sql(table_name, sql_connection, if_exists="replace", index=False)


# 1.6 Run Query Function
def run_query(query_statement, sql_connection):
    """Executes a SQL query and logs the execution."""
    print(query_statement)
    query_output = pd.read_sql(query_statement, sql_connection)
    print(query_output)
    log_progress(f"Executed query: {query_statement}")


# ---- Main ETL Pipeline Execution ----
if __name__ == "__main__":
    log_progress("Preliminaries complete. Initiating ETL process")

    # Menjalankan ekstraksi dengan parameter sesuai permintaan
    df_extracted = extract(DATA_URL, ["Name", "MC_USD_Billion"])
    log_progress("Data extraction complete. Initiating Transformation process")
    
    # Menampilkan print statement
    print(df_extracted.head(10))

    df = transform(df_extracted, EXCHANGE_RATE_CSV)
    log_progress("Data transformation complete. Initiating Loading process")

    load_to_csv(df, OUTPUT_CSV_PATH)
    log_progress("Data saved to CSV file")

    conn = sqlite3.connect(DB_NAME)
    log_progress("SQL Connection initiated")

    load_to_db(df, conn, TABLE_NAME)
    log_progress("Data loaded to Database as a table, Executing queries")

    run_query(f"SELECT * FROM {TABLE_NAME}", conn)
    run_query(f"SELECT AVG(MC_GBP_Billion) FROM {TABLE_NAME}", conn)
    run_query(f"SELECT Name FROM {TABLE_NAME} LIMIT 5", conn)

    log_progress("Process Complete")
    conn.close()
    log_progress("Server Connection closed")
