# PMC-Downloader-Semi-Automated-V001

This README describes a semi-automated workflow for downloading full-text PDFs from PubMed Central (PMC) based on a list of PMIDs. The script retrieves metadata (publication year, journal name) for each PMID, checks whether a free full-text PDF is available on PMC, and organizes the downloaded PDFs into a structured folder hierarchy (`allFiles/<year>/<journal>`). Finally, it bundles everything into a single ZIP file. Moreover, you will have a .txt file that contains all PMIDs for which no paper is available on PMC, allowing for further analysis.

Below, you’ll find instructions on setting up your environment, running the script, and understanding the outputs.

## How to Use

1. **Visit PubMed**  
   Go to [https://pubmed.ncbi.nlm.nih.gov/](https://pubmed.ncbi.nlm.nih.gov/) to begin your search.

2. **Formulate Your Search Query**  
   Decide which expressions (keywords or phrases) you want to appear in the article’s title or abstract. For example:

   ```text
   (
     ("Escherichia coli"[Title/Abstract]) 
     OR ("Here you can add other synonyms of Escherichia coli"[Title/Abstract])
     OR (...[Title/Abstract])
   )
   AND
   (
     ("Mutation Rate"[Title/Abstract])
     OR ("Here you can add other synonyms of Mutation Rate"[Title/Abstract])
     OR (...[Title/Abstract])
   )

3. **(Optional) Manual Steps**
If you prefer a more manual approach, follow the steps below:

**Step 1**
![1](https://github.com/user-attachments/assets/8f29629c-722d-4a67-984a-93c0549c1eb0)

**Step 2**
![222](https://github.com/user-attachments/assets/8ee93f80-f7a5-4640-a2b8-88a1d8155412)

**Step 3**
![3](https://github.com/user-attachments/assets/26a59872-711c-4868-bdc1-70597ff75dd3)

4. **Download .nbib File**  

**Step 1**
![4](https://github.com/user-attachments/assets/bc9c042a-cdf4-4e83-8c97-413287ea6291)

**Step 2**
![323232](https://github.com/user-attachments/assets/0f35682c-9b90-4415-ad89-70e42104e1fb)

**Then you will have your .nbib file**


## Extract PMIDs from an `.nbib` File

The following function reads an `.nbib` file (exported from PubMed), extracts PMIDs, and writes them to a plain-text file, one PMID per line.
**Install Libraries**
```python
pip install biopython beautifulsoup4 requests
```
**Import Libraries**
```python
from Bio import Entrez
from bs4 import BeautifulSoup
import requests
import os
import time
import re
from collections import defaultdict
import zipfile
```
**Extract PMIDs**
> **Note:** Here you have to change the name of the `.nbib` file (in the last line) to your own file name.
```python

def extract_pmids(nbib_file, output_file):
    """
    Extracts PMIDs from an .nbib file and saves them to a .txt file.

    Args:
        nbib_file (str): Path to the .nbib file.
        output_file (str): Path to save the extracted PMIDs.
    """
    with open(nbib_file, "r", encoding="utf-8") as f:
        content = f.read()

    # Use regex to find all PMIDs (they are usually formatted as "PMID- xxxxxxxx")
    pmids = re.findall(r"PMID-\s*(\d+)", content)

    # Save to a text file
    with open(output_file, "w") as f:
        for pmid in pmids:
            f.write(pmid + "\n")

    print(f"Extracted {len(pmids)} PMIDs and saved to {output_file}")

# Example usage
extract_pmids("Mutation Rate AND Escherichia coli.nbib", "pmids.txt")
```
## Download All Available Papers as a .zip File and Categorize them Based on Publication Year and Journal
> **Note:** Here you have to enter your own email address (in the first line).
```python
# Set your email (required by NCBI)
Entrez.email = "Your Email Address"

# Function to retrieve metadata for a given PMID (year and journal name)
def get_metadata(pmid):
    """Retrieve metadata (year and journal name) for a given PMID."""
    try:
        handle = Entrez.esummary(db="pubmed", id=pmid)
        record = Entrez.read(handle)
        handle.close()

        # Extract year and journal name from the record
        # Note: sometimes PubDate can be in various formats. We take the first token.
        pub_date = record[0].get("PubDate", "")
        year = pub_date.split()[0] if pub_date else None

        # Journal name may be under FullJournalName, if available.
        journal = record[0].get("FullJournalName", None)

        return year, journal
    except Exception as e:
        print(f"Error retrieving metadata for PMID {pmid}: {e}")
        return None, None

# Function to retrieve full-text link from PMC
def get_full_text_links(pmid):
    """Retrieve full-text link if available in PubMed Central (PMC)."""
    try:
        handle = Entrez.elink(dbfrom="pubmed", id=pmid, linkname="pubmed_pmc")
        record = Entrez.read(handle)
        handle.close()

        if record[0]["LinkSetDb"]:
            pmc_id = record[0]["LinkSetDb"][0]["Link"][0]["Id"]
            full_text_url = f"https://www.ncbi.nlm.nih.gov/pmc/articles/PMC{pmc_id}/pdf/"
            return full_text_url
        else:
            return None
    except Exception as e:
        print(f"Error retrieving full-text link for PMID {pmid}: {e}")
        return None

# Function to download the full-text PDF
def download_full_text(url, pmid, folder):
    """Download the full-text PDF from the given URL."""
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
    }

    try:
        response = requests.get(url, stream=True, headers=headers)
        if response.status_code == 200:
            # Ensure folder exists
            os.makedirs(folder, exist_ok=True)
            file_name = os.path.join(folder, f"PMC_{pmid}.pdf")
            with open(file_name, 'wb') as f:
                for chunk in response.iter_content(chunk_size=8192):
                    f.write(chunk)
            print(f"Downloaded: {file_name}")
        else:
            print(f"Failed to download full text for PMID {pmid}. HTTP Status: {response.status_code}")
    except Exception as e:
        print(f"Error downloading full text for PMID {pmid}: {e}")

    time.sleep(1)  # Delay between requests

# Function to zip the files into the final structure
def zip_folders():
    """Zip the top-level folder containing all the year and journal subfolders."""
    folder_name = "allFiles"
    zip_filename = f"{folder_name}.zip"
    with zipfile.ZipFile(zip_filename, "w", zipfile.ZIP_DEFLATED) as zipf:
        for root, dirs, files in os.walk(folder_name):
            for file in files:
                # write file into zip with relative path
                zipf.write(os.path.join(root, file), os.path.relpath(os.path.join(root, file), folder_name))
    print(f"Zipped all files into {zip_filename}")

# Main function to process PMIDs
def main():
    # Read PMIDs from the file
    with open('pmids.txt', 'r') as file:
        pmids = [line.strip() for line in file.readlines() if line.strip()]

    # Dictionary to hold files organized by year and journal (optional use)
    year_journal_dict = defaultdict(lambda: defaultdict(list))
    # List to hold PMIDs with no free full text access
    no_full_text_pmids = []

    # Process each PMID
    for pmid in pmids:
        print(f"Checking metadata and full text for PMID: {pmid}")

        # Retrieve metadata (year and journal name)
        year, journal = get_metadata(pmid)
        # If metadata is missing, assign "Unknown"
        if not year or not journal:
            folder = os.path.join("allFiles", "Unknown")
            print(f"Metadata missing for PMID {pmid}. Using folder 'Unknown'.")
        else:
            folder = os.path.join("allFiles", year, journal)

        # Retrieve full-text link
        full_text_url = get_full_text_links(pmid)
        if full_text_url:
            print(f"Full-text URL: {full_text_url}")
            download_full_text(full_text_url, pmid, folder)
            # (Optional) store pmid in dict
            if year and journal:
                year_journal_dict[year][journal].append(pmid)
            else:
                year_journal_dict["Unknown"]["Unknown"].append(pmid)
        else:
            print(f"No free full-text available for PMID {pmid}.")
            no_full_text_pmids.append(pmid)

        print('-' * 80)

    # Write the PMIDs with no free full-text access to a txt file
    if no_full_text_pmids:
        with open('no_full_access.txt', 'w') as file:
            for pmid in no_full_text_pmids:
                file.write(f"{pmid}\n")
        print(f"PMIDs with no free full-text access have been saved to 'no_full_access.txt'")
    else:
        print("All articles have free full-text access.")

    # Zip the folder structure
    zip_folders()

if __name__ == "__main__":
    main()


