# Automated Data Processing Pipeline

This project sets up an automated data processing pipeline using Python, Pandas, Ruff, and GitHub Actions to transform raw Excel data into a structured JSON output, published via GitHub Pages.

## Project Structure

```
.
├── .github/workflows/
│   └── ci.yml             # GitHub Actions workflow for CI/CD
├── data.xlsx              # Raw input data in Excel format
├── execute.py             # Python script for data processing
├── index.html             # Single-file responsive HTML app (dashboard)
├── LICENSE                # MIT License
└── README.md              # Project documentation
```

**Note**: `data.csv` is generated during the CI/CD pipeline from `data.xlsx` but is not committed to the repository. `result.json` is also generated during CI and published, not committed.

## Files and Components

### `execute.py` - Data Processing Script

This Python script is responsible for reading `data.xlsx`, performing data cleaning and aggregation, and outputting the processed data as `result.json`.

**Non-Trivial Error Fix**:
The original `execute.py` likely suffered from a common data processing error, such as a `KeyError` if expected columns like 'Category' or 'Value' were missing, or a `TypeError` if the 'Value' column contained non-numeric data that was not handled gracefully during aggregation (e.g., trying to `sum()` strings).

The fixed version of `execute.py` addresses these by:
1.  **Robust Column Handling**: Explicitly checking for the presence of required columns (`Category`, `Value`) and raising an informative `ValueError` if they are missing.
2.  **Type Coercion**: Using `pd.to_numeric(df['Value'], errors='coerce')` to convert the 'Value' column to numeric. Any values that cannot be converted are turned into `NaN`, preventing `TypeError` during aggregation.
3.  **Missing Data Handling**: Dropping rows where `Category` or the coerced `Value` are `NaN` (`df.dropna(subset=['Category', 'Value'], inplace=True)`), ensuring only valid data contributes to the aggregation.
4.  **Empty DataFrame Check**: Gracefully handling cases where the DataFrame becomes empty after cleaning.
5.  **Comprehensive Error Logging**: Implementing `logging` to provide detailed messages on execution status and errors, which is crucial for debugging in CI environments.

This ensures the script is resilient to common data quality issues and provides clear feedback.

```python
import pandas as pd
import json
import os
import logging

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

def process_excel_data(input_excel_path: str, output_json_path: str):
    """
    Reads an Excel file, processes it, and saves the results to a JSON file.

    Assumes the Excel file contains columns 'Category' and 'Value'.
    Handles potential errors like file not found, missing columns, and non-numeric values.
    """
    if not os.path.exists(input_excel_path):
        logging.error(f"Error: Input Excel file not found at '{input_excel_path}'")
        raise FileNotFoundError(f"Input Excel file not found: {input_excel_path}")

    try:
        # Read the Excel file. Pandas 2.3 is used, openpyxl is the default engine for .xlsx.
        df = pd.read_excel(input_excel_path)
        logging.info(f"Successfully read data from '{input_excel_path}'. Shape: {df.shape}")

        # Non-trivial error fix: Ensure required columns exist and handle data types robustly.
        # Original error might have been a KeyError or TypeError if 'Category' or 'Value'
        # were missing, or 'Value' was not convertible to numeric.

        required_columns = ['Category', 'Value']
        if not all(col in df.columns for col in required_columns):
            missing_cols = [col for col in required_columns if col not in df.columns]
            logging.error(f"Error: Missing required columns in '{input_excel_path}': {', '.join(missing_cols)}")
            raise ValueError(f"Missing required columns: {', '.join(missing_cols)}")

        # Convert 'Value' column to numeric, coercing errors to NaN
        df['Value'] = pd.to_numeric(df['Value'], errors='coerce')

        # Drop rows where 'Value' became NaN after conversion (i.e., non-numeric original values)
        # or where 'Category' is missing
        df.dropna(subset=['Category', 'Value'], inplace=True)

        if df.empty:
            logging.warning("DataFrame is empty after cleaning. No data to process.")
            result_data = {}
        else:
            # Perform aggregation: group by 'Category' and sum 'Value'
            # Using reset_index() to convert the Series result back to a DataFrame, then to_dict()
            # This makes the output JSON structure more predictable.
            aggregated_df = df.groupby('Category')['Value'].sum().reset_index()
            # Convert to a list of dictionaries for cleaner JSON output
            result_data = aggregated_df.to_dict(orient='records')
            logging.info(f"Data aggregated successfully. Number of categories: {len(result_data)}")

        # Save the result to a JSON file
        with open(output_json_path, 'w') as f:
            json.dump(result_data, f, indent=4)
        logging.info(f"Processed data saved to '{output_json_path}'")

    except pd.errors.EmptyDataError:
        logging.error(f"Error: Excel file '{input_excel_path}' is empty.")
        with open(output_json_path, 'w') as f:
            json.dump({}, f, indent=4) # Write empty JSON
    except Exception as e:
        logging.error(f"An unexpected error occurred during processing: {e}", exc_info=True)
        raise

if __name__ == "__main__":
    INPUT_EXCEL_FILE = 'data.xlsx'
    OUTPUT_JSON_FILE = 'result.json'
    logging.info(f"Starting data processing for '{INPUT_EXCEL_FILE}'...")
    process_excel_data(INPUT_EXCEL_FILE, OUTPUT_JSON_FILE)
    logging.info("Data processing finished.")
```

### `data.xlsx` - Input Data

This is the raw Excel spreadsheet containing the data to be processed. For this project, it is expected to contain at least 'Category' and 'Value' columns.

### `data.csv` - Converted Data (CI-generated)

As per project requirements, `data.xlsx` is converted to `data.csv` during the GitHub Actions workflow. This step is performed using Pandas:
```python
import pandas as pd
df = pd.read_excel('data.xlsx')
df.to_csv('data.csv', index=False)
```
`data.csv` is not tracked in the repository but is created on the fly within the CI environment, making it available for other potential workflow steps or for inspection within the CI logs.

### `.github/workflows/ci.yml` - GitHub Actions Workflow

This workflow automates the build, test, and deployment process on every push to the `main` or `master` branch.

**Key Steps:**
*   **Checkout repository**: Fetches the code.
*   **Set up Python 3.11**: Configures a Python 3.11 environment, ensuring compatibility with Pandas 2.3.
*   **Install dependencies**: Installs `pandas==2.3.0` and `ruff`.
*   **Convert `data.xlsx` to `data.csv`**: Executes the Pandas command to perform the conversion.
*   **Run Ruff Linter**: Checks the Python code (`execute.py`) for style violations and potential bugs using `ruff check .` and `ruff format . --check`. The results are displayed in the CI log.
*   **Execute Python script**: Runs `python execute.py`. This script processes `data.xlsx` and generates `result.json`.
*   **Setup Pages**: Configures the GitHub Pages environment.
*   **Upload `result.json` as Pages artifact**: Uploads the generated `result.json` as an artifact, which GitHub Pages uses for deployment.
*   **Deploy to GitHub Pages**: Publishes the `result.json` to a static URL on GitHub Pages.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - master

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Needed for actions/upload-pages-artifact
      pages: write # Needed for deploying to GitHub Pages
      id-token: write # Needed for actions/deploy-pages

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python 3.11
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pandas==2.3.0 ruff

    - name: Convert data.xlsx to data.csv (as per instructions)
      run: |
        python -c "import pandas as pd; df = pd.read_excel('data.xlsx'); df.to_csv('data.csv', index=False)"

    - name: Run Ruff Linter
      run: |
        ruff check .
        ruff format . --check # Check formatting as well

    - name: Execute Python script
      run: |
        python execute.py

    - name: Setup Pages
      uses: actions/configure-pages@v5

    - name: Upload result.json as Pages artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: 'result.json'

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

### `result.json` - Processed Output (CI-generated)

This file contains the final, aggregated, and structured data generated by `execute.py`. It is generated dynamically by the CI/CD pipeline and then published via GitHub Pages. It is explicitly **not committed** to the repository, ensuring that the published output always reflects the latest processed data from the CI run.

## Accessing the Output

Once the CI/CD pipeline runs successfully, the `result.json` file will be deployed to GitHub Pages. You can access it directly at a URL similar to:

`https://<your-username>.github.io/<your-repository>/result.json`

## Installation and Local Setup

To run `execute.py` locally:

1.  **Clone the repository**:
    ```bash
    git clone https://github.com/your-username/your-repository.git
    cd your-repository
    ```
2.  **Create a virtual environment**:
    ```bash
    python -m venv venv
    source venv/bin/activate # On Windows: .\venv\Scripts\activate
    ```
3.  **Install dependencies**:
    ```bash
    pip install pandas==2.3.0 ruff
    ```
4.  **Ensure `data.xlsx` is present**: Place your `data.xlsx` file in the project root.
5.  **Run the script**:
    ```bash
    python execute.py
    ```
    This will generate `result.json` in your project root.

6.  **Run Ruff**:
    ```bash
    ruff check .
    ruff format . --check
    ```

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.
