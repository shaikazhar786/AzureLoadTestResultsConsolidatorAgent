# AzureLoadTestResultsConsolidatorAgent
Azure Load Test Results Consolidator Agent





Azure Load Test Results Consolidator Agent
Upload multiple Azure Load Testing result ZIP files to filter out ramp-up periods and generate a JMeter-style Aggregate Report.

**Step 1 --create a app.py in vscode**

import streamlit as st
import pandas as pd
import numpy as np
import zipfile
import io
import os

st.set_page_config(page_title="Azure Load Test Consolidator Agent", layout="wide")

st.title("📊 Azure Load Test Results Consolidator Agent")
st.write("Upload multiple Azure Load Testing result ZIP files to filter out ramp-up periods and generate a JMeter-style Aggregate Report.")

--- UI Sidebar Inputs ---
st.sidebar.header("Configuration")
ramp_up_seconds = st.sidebar.number_input(
"Enter Ramp-up Period (in seconds):",
min_value=0,
value=60,
step=1,
help="Transactions executed within this timeframe from the start of the test will be excluded."
)

--- File Uploader ---
uploaded_zips = st.file_uploader(
"Upload Azure Load Test Result ZIP files",
type=["zip"],
accept_multiple_files=True
)

def process_load_test_data(zip_files, ramp_up_sec):
all_raw_data = []

for uploaded_zip in zip_files:
    # Open the zip archive from memory
    with zipfile.ZipFile(io.BytesIO(uploaded_zip.read())) as z:
        zip_engine_data = []
        
        # Loop through all files in the individual zip archive
        for file_info in z.infolist():
            if file_info.filename.endswith('.csv') or file_info.filename.endswith('.xlsx'):
                with z.open(file_info) as f:
                    try:
                        if file_info.filename.endswith('.csv'):
                            df = pd.read_csv(f)
                        else:
                            df = pd.read_excel(f)
                        
                        # Standardizing column headers to lowercase to handle variations
                        df.columns = [col.strip().lower() for col in df.columns]
                        
                        # Ensure essential columns exist
                        required_cols = ['timestamp', 'label', 'elapsed', 'success', 'bytes', 'sentbytes']
                        if all(col in df.columns for col in ['timestamp', 'label', 'elapsed']):
                            zip_engine_data.append(df)
                    except Exception as e:
                        st.warning(f"Could not parse file {file_info.filename} in {uploaded_zip.name}: {e}")
        
        if zip_engine_data:
            # Merge all individual engine files for this specific zip
            consolidated_zip_df = pd.concat(zip_engine_data, ignore_index=True)
            all_raw_data.append(consolidated_zip_df)

if not all_raw_data:
    return None

# Step 1: Combine everything into one absolute Master Sheet
master_df = pd.concat(all_raw_data, ignore_index=True)

# Step 2: Standardize Timestamps & Filter out Ramp-up
# Convert timestamp column to numeric (JMeter/Azure default is unix epoch millis)
master_df['timestamp'] = pd.to_numeric(master_df['timestamp'], errors='coerce')
master_df = master_df.dropna(subset=['timestamp'])

min_timestamp_ms = master_df['timestamp'].min()
ramp_up_ms = ramp_up_seconds * 1000
steady_state_start_ms = min_timestamp_ms + ramp_up_ms

# Filter for Steady State
steady_state_df = master_df[master_df['timestamp'] >= steady_state_start_ms].copy()

if steady_state_df.empty:
    st.error("Error: No data left after excluding the ramp-up period! Lower your ramp-up seconds input.")
    return None
    
# Calculate total test duration in seconds for Throughput calculation
max_timestamp_ms = steady_state_df['timestamp'].max()
total_duration_sec = (max_timestamp_ms - steady_state_start_ms) / 1000.0
if total_duration_sec <= 0:
    total_duration_sec = 1 # Avoid division by zero
    
# Step 3: Compute JMeter Aggregate Report Metrics
# Ensure numerical columns are correctly typed
steady_state_df['elapsed'] = pd.to_numeric(steady_state_df['elapsed'], errors='coerce').fillna(0)
steady_state_df['bytes'] = pd.to_numeric(steady_state_df.get('bytes', 0), errors='coerce').fillna(0)
steady_state_df['sentbytes'] = pd.to_numeric(steady_state_df.get('sentbytes', 0), errors='coerce').fillna(0)

# Parse success flag safely
if 'success' in steady_state_df.columns:
    steady_state_df['error_count'] = steady_state_df['success'].apply(lambda x: 1 if str(x).strip().lower() in ['false', '0', 'f'] else 0)
else:
    steady_state_df['error_count'] = 0

# Custom aggregation functions matching the JMeter Logic
def percentile_90(x): return np.percentile(x, 90) if len(x) > 0 else 0
def percentile_95(x): return np.percentile(x, 95) if len(x) > 0 else 0
def percentile_99(x): return np.percentile(x, 99) if len(x) > 0 else 0

# Group by transaction label
grouped = steady_state_df.groupby('label')

summary_rows = []

for label, group in grouped:
    samples = len(group)
    avg_rt = group['elapsed'].mean()
    median_rt = group['elapsed'].median()
    p90 = percentile_90(group['elapsed'])
    p95 = percentile_95(group['elapsed'])
    p99 = percentile_99(group['elapsed'])
    min_rt = group['elapsed'].min()
    max_rt = group['elapsed'].max()
    
    error_pct = (group['error_count'].sum() / samples) * 100 if samples > 0 else 0
    throughput = samples / total_duration_sec if total_duration_sec > 0 else 0
    
    # Received / Sent rates convert bytes to KB/sec
    received_kb_sec = (group['bytes'].sum() / 1024.0) / total_duration_sec if total_duration_sec > 0 else 0
    sent_kb_sec = (group['sentbytes'].sum() / 1024.0) / total_duration_sec if total_duration_sec > 0 else 0
    
    summary_rows.append({
        "Label": label,
        "# Samples": samples,
        "Average": round(avg_rt),
        "Median": round(median_rt),
        "90% Line": round(p90),
        "95% Line": round(p95),
        "99% Line": round(p99),
        "Min": round(min_rt),
        "Max": round(max_rt),
        "Error %": f"{error_pct:.2f}%",
        "Throughput": round(throughput, 5),
        "Received KB/sec": round(received_kb_sec, 2),
        "Sent KB/sec": round(sent_kb_sec, 2)
    })
    
# Generate Total/Summary Row
total_samples = len(steady_state_df)
if total_samples > 0:
    total_error_pct = (steady_state_df['error_count'].sum() / total_samples) * 100
    total_received = (steady_state_df['bytes'].sum() / 1024.0) / total_duration_sec
    total_sent = (steady_state_df['sentbytes'].sum() / 1024.0) / total_duration_sec
    
    summary_rows.append({
        "Label": "TOTAL",
        "# Samples": total_samples,
        "Average": round(steady_state_df['elapsed'].mean()),
        "Median": round(steady_state_df['elapsed'].median()),
        "90% Line": round(percentile_90(steady_state_df['elapsed'])),
        "95% Line": round(percentile_95(steady_state_df['elapsed'])),
        "99% Line": round(percentile_99(steady_state_df['elapsed'])),
        "Min": round(steady_state_df['elapsed'].min()),
        "Max": round(steady_state_df['elapsed'].max()),
        "Error %": f"{total_error_pct:.2f}%",
        "Throughput": round(total_samples / total_duration_sec, 5),
        "Received KB/sec": round(total_received, 2),
        "Sent KB/sec": round(total_sent, 2)
    })

return pd.DataFrame(summary_rows)
--- Processing Execution ---
if uploaded_zips:
if st.button("🚀 Consolidate & Calculate Steady State"):
with st.spinner("Processing archives, filtering ramp-up, and computing metrics..."):
final_report_df = process_load_test_data(uploaded_zips, ramp_up_seconds)

        if final_report_df is not None:
            st.success("✅ Consolidation complete!")
            
            # Render UI preview matching the excel format
            st.dataframe(final_report_df)
            
            # Convert DataFrame to standard CSV layout string
            csv_buffer = io.StringIO()
            final_report_df.to_csv(csv_buffer, index=False)
            csv_data = csv_buffer.getvalue()
            
            # Download Button
            st.download_button(
                label="📥 Download Consolidated Aggregate Report CSV",
                data=csv_data,
                file_name="consolidated_aggregate_report.csv",
                mime="text/csv"
            )
else:
st.info("Please upload one or multiple Azure Load Test zip outputs to begin.")

**Step 2 --run this command -- python -m streamlit run app.py**
