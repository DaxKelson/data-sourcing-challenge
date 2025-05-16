# data-sourcing-challenge
This project retrieves **Coronal Mass Ejection (CME)** and **Geomagnetic Storm (GST)** event data from NASA’s DONKI (Space Weather Database Of Notifications, Knowledge, Information) API. DONKI provides a curated database of space weather events with *linkedEvents* that associate related phenomena (e.g. flares→CMEs→storms). We use the API to fetch recent CME and GST records, match CMEs to ensuing geomagnetic storms, and compute the time delay between each CME’s launch and the resulting storm at Earth. In NOAA’s terms, CMEs are huge expulsions of plasma/magnetic field from the Sun’s corona, and geomagnetic storms are major disturbances of Earth’s magnetosphere often triggered by CMEs. By linking DONKI events via their `activityID` references, the script identifies which CMEs correspond to which storms and calculates the transit time between them.

## Setup

- **Python:** Install Python 3.8 or newer.  
- **Dependencies:** The script uses common packages: `requests`, `pandas`, `python-dotenv` (for environment variables), etc.  
- **Install:** Clone the repository and run:  
  ```bash
  pip install -r requirements.txt
  ```  
- **Virtual Environment (optional):** We recommend using a Python virtual environment to manage dependencies.

## Environment Configuration

Create a `.env` file in the project directory containing your NASA API key. For example:  
```text
NASA_API_KEY=your_nasa_api_key_here
```  
Load this in Python using `python-dotenv`. For example:  
```python
from dotenv import load_dotenv
import os
load_dotenv()  
api_key = os.getenv("NASA_API_KEY")
```  
You can obtain a (free) NASA API key from [api.nasa.gov](https://api.nasa.gov). The key should match the name used in your `.env` (e.g. `NASA_API_KEY`).

## Data Retrieval and Analysis

1. **Retrieve CME data:** Use `requests.get` on the DONKI CME endpoint with your date range. The API returns a JSON array of CME records. Each record includes fields like `activityID`, `startTime`, and a `linkedEvents` list.  
2. **Retrieve GST data:** Similarly, query the DONKI GST endpoint. Each storm record has `gstID`, `startTime`, Kp index list (`allKpIndex`), and its own `linkedEvents`.  
3. **Normalize and clean:** Load the JSON into pandas DataFrames. Flatten any nested lists (e.g. Kp indices in GST, or instrument lists) as needed, and parse the `startTime` fields into Python `datetime` objects.  
4. **Merge/link events:** Join the CME and GST tables by matching their linked IDs. For example, if a GST record’s `linkedEvents` contains a CME’s `activityID`, pair those records. (DONKI’s *linkedEvents* allows identifying which CME(s) led to a given storm.) You may use pandas merge or iterate through the links to create pairs of related CME and GST events.  
5. **Compute time delays:** For each linked CME–GST pair, compute the time difference `delay = (GST_startTime – CME_startTime)`. Convert to hours or days as needed. This gives the transit time for each CME-driven storm.  
6. **Summary statistics:** Calculate summary metrics (e.g. average, median, and max delay; count of events). Also aggregate statistics of the storms (like peak Kp index). You can use pandas functions (`.mean()`, `.describe()`, etc.) on the combined data.  
7. **Export results:** Save the merged dataset with delays to a CSV file (e.g. `cme_gst_analysis.csv`). Print or save summary statistics for quick reference.

## Output

- **CSV Export:** The script generates an output CSV (e.g. `cme_gst_analysis.csv`) with columns such as `CME_ID`, `CME_startTime`, `GST_ID`, `GST_startTime`, and `Delay_hours`. Each row corresponds to a CME–GST pair.  
- **Sample Content (CSV):**  
  ```
  CME_ID,            CME_startTime,       GST_ID,            GST_startTime,      Delay_hours
  2023-11-03T05:48:00-CME-001,2023-11-03T05:48Z,2023-11-05T09:00:00-GST-001,2023-11-05T09:00Z,51.20
  2023-11-10T12:30:00-CME-001,2023-11-10T12:30Z,2023-11-12T15:15:00-GST-001,2023-11-12T15:15Z,50.75
  ```  
- **Summary Statistics:** The script prints (or logs) a brief summary. For example:  
  ```text
  Total CMEs processed: 25
  Total GSTs processed: 10
  Linked CME–GST pairs: 9
  Avg. CME→GST delay: 49.8 hours
  Max delay: 55.3 hours
  Avg. peak Kp: 5.7
  ```  
  These values will vary with the data period.

## Data Assumptions and Source

- **Source:** All data come from NASA’s DONKI API (CME and GST endpoints). We assume the API provides complete event lists for the queried date range.  
- **Time zones:** All `startTime` values from DONKI are in UTC (denoted by “Z”). No time zone conversion is applied.  
- **Date range:** By default, DONKI returns data for the last 30 days if no `startDate`/`endDate` are given. The script allows specifying custom date ranges via its parameters or environment.  
- **Event linking:** We rely on DONKI’s `linkedEvents` fields to pair CMEs and GSTs. If an expected link is missing, that pair is skipped. (In practice, complex events may have multiple CMEs for one storm, or vice versa.)  
- **Other filters:** The analysis focuses on Earth-directed events (the DONKI data used is the Earth catalog, by default). Geomagnetic storms by definition affect Earth’s magnetosphere.
